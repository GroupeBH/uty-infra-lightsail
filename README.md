# Uty API Lightsail Infrastructure

Projet d'infrastructure autonome pour déployer une API NestJS déjà publiée sur Docker Hub vers une instance AWS Lightsail simple, prévisible et peu coûteuse.

La cible initiale est volontairement sobre : une instance Linux/Unix Lightsail, une IP statique, un firewall minimal, Docker Compose, l'API NestJS et Caddy en reverse proxy HTTP/HTTPS. Pas d'EC2, pas de VPC custom, pas de NAT Gateway, pas d'ALB, pas d'Auto Scaling Group, pas d'ECS et pas de RDS dans cette première version.

## Architecture

- Terraform provisionne l'instance Lightsail, l'IP statique, l'attachement IP et les ports publics.
- `deploy.sh` lance Terraform, lit les outputs, génère `ansible/inventory.ini`, attend SSH puis lance Ansible.
- Ansible installe Docker et le plugin Docker Compose, crée `/opt/uty-api`, copie l'environnement applicatif local vers `/opt/uty-api/.env`, rend les templates Compose/Caddy et démarre les conteneurs.
- Docker Compose exécute `app` en interne sur le port 3000 et `caddy` en public sur 80/443.

## Coût Estimé

Le bundle par défaut est `micro_3_1`, correspondant au plan Linux/Unix avec IPv4 public autour de 7 USD/mois : 1 GB RAM, 2 vCPU, 40 GB SSD et environ 2 TB de transfert mensuel inclus.

Points de coût à garder en tête :

- L'instance Lightsail est le coût principal.
- L'IP statique Lightsail est incluse tant qu'elle reste attachée à une instance.
- Les snapshots Lightsail sont facturés séparément au GB/mois.
- Le transfert sortant au-delà du quota du plan peut générer des frais.
- Redis peut rester externe, par exemple Upstash, via variables dans `.env.production`.
- Aucune base de données Lightsail n'est créée ici.

Vérifier les bundles disponibles avant production :

```bash
aws lightsail get-bundles --include-inactive
```

Si AWS change l'identifiant du plan, remplace `lightsail_bundle_id` dans `terraform/terraform.tfvars`.

## Prérequis Locaux

Depuis Linux ou WSL :

- Terraform
- AWS CLI
- Ansible
- `ssh`
- des credentials AWS capables de gérer Lightsail
- une clé SSH Lightsail existante dans la région choisie

Configurer les credentials AWS, par exemple :

```bash
aws configure
aws sts get-caller-identity
```

Ou utiliser un profil :

```bash
export AWS_PROFILE=uty-prod
```

## Clé SSH Lightsail

Pour cette infra, on utilise la clé par défaut Lightsail de la région. Dans `terraform/terraform.tfvars`, garder :

```hcl
key_pair_name = ""
ssh_user      = "ubuntu"
```

Télécharger la clé par défaut depuis la console AWS Lightsail : `Lightsail > Account > SSH keys > Default keys`, puis choisir la clé de `eu-central-1`.

Placer la clé dans WSL :

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
mv /chemin/vers/LightsailDefaultKey-eu-central-1.pem ~/.ssh/uty-lightsail.pem
chmod 600 ~/.ssh/uty-lightsail.pem
```

Au déploiement local :

```bash
export SSH_PRIVATE_KEY_PATH=~/.ssh/uty-lightsail.pem
```

Si une clé custom est utilisée plus tard, renseigner `key_pair_name` avec son nom Lightsail et adapter `SSH_PRIVATE_KEY_PATH`.

## Configuration

Créer le fichier Terraform local :

```bash
cp terraform/terraform.tfvars.example terraform/terraform.tfvars
```

À remplir avec vos vraies valeurs :

- `admin_cidr` : votre IP publique admin en `/32`, jamais `0.0.0.0/0` pour SSH.
- `key_pair_name` : laisser vide `""` pour utiliser la clé par défaut Lightsail.
- `aws_region` et `availability_zone` si vous ne voulez pas `eu-central-1`.
- `lightsail_bundle_id` si AWS a remplacé `micro_3_1`.
- `domain_name` si vous utilisez autre chose que `api.uty-app.com`.

Créer le fichier d'environnement applicatif local :

```bash
touch .env.production
chmod 600 .env.production
```

À remplir dans `.env.production` avec les variables NestJS réelles : base de données externe, Redis Upstash, JWT secrets, CORS, etc. Ce fichier est ignoré par Git et sera copié sur le serveur vers `/opt/uty-api/.env` avec le mode `0600`.

Exporter les variables de déploiement :

```bash
export APP_IMAGE_REPOSITORY=dockerhub-user/uty-api
export APP_IMAGE_TAG=2026-04-18
export APP_ENV_FILE=.env.production
export SSH_PRIVATE_KEY_PATH=~/.ssh/uty-lightsail.pem
export DOMAIN_NAME=api.uty-app.com
export CADDY_EMAIL=admin@uty-app.com
export HEALTHCHECK_PATH=/health
```

Pour tester d'abord en HTTP par IP, utiliser temporairement :

```bash
export DOMAIN_NAME=
```

## Déploiement

```bash
chmod +x deploy.sh
./deploy.sh
```

Le script exécute :

1. `terraform init`
2. `terraform apply`
3. lecture des outputs Terraform
4. génération de `ansible/inventory.ini`
5. attente SSH
6. `ansible-playbook`
7. `docker compose pull`
8. `docker compose up -d --remove-orphans`
9. affichage de `docker compose ps`

## CI/CD GitHub Actions

Un workflow est fourni dans `.github/workflows/deploy-lightsail.yml`. Il se déclenche sur un push vers `master`.

Pour qu'un push du repository GitHub `uty-api` déclenche ce pipeline, ce workflow doit être présent dans ce repository `uty-api`. Si ce projet infra reste dans un repository séparé, GitHub ne déclenchera pas automatiquement le workflow sur les pushes de `uty-api` sans `repository_dispatch` ou sans copier le workflow dans le repo applicatif.

Le workflow attend une image Docker déjà publiée sur Docker Hub. Il déploie le tag suivant :

1. input manuel `app_image_tag` ;
2. variable GitHub `APP_IMAGE_TAG` ;
3. SHA du commit GitHub.

Secrets GitHub requis :

- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
- `LIGHTSAIL_SSH_PRIVATE_KEY`
- `APP_ENV_PRODUCTION`

Variables GitHub minimales :

- `APP_IMAGE_REPOSITORY`

Voir [docs/github-actions.md](docs/github-actions.md) pour la configuration complète. Le backend Terraform utilise le bucket S3 `uty-lightsail-tfstate`.

## DNS

Quand Terraform affiche l'IP statique, créer ou modifier le record DNS :

```text
api.uty-app.com.  A  <IP_STATIQUE_LIGHTSAIL>
```

Pour HTTPS automatique avec Caddy, le DNS doit pointer vers l'IP Lightsail avant le déploiement avec `DOMAIN_NAME=api.uty-app.com`, ou au moins avant de relancer `./deploy.sh` avec le domaine activé.

## Rollback Simple

Changer uniquement le tag Docker et relancer :

```bash
export APP_IMAGE_TAG=previous-known-good-tag
./deploy.sh
```

Ansible relancera `docker compose pull` puis `docker compose up -d --remove-orphans`.

## Backup

- Activer ou créer des snapshots Lightsail pour l'instance si le disque contient des données utiles.
- Sauvegarder séparément la base de données externe.
- Sauvegarder séparément les services managés externes, par exemple Redis/Upstash si nécessaire.
- Ne pas considérer l'instance comme source de vérité applicative : l'image Docker et `.env.production` doivent être reproductibles.

## Validation Locale

```bash
terraform -chdir=terraform fmt
terraform -chdir=terraform validate
ansible-playbook --syntax-check ansible/playbook.yml
```

`terraform validate` nécessite un `terraform init` préalable.

## Détruire l'Infra

```bash
terraform -chdir=terraform destroy
```

Vérifier les snapshots et les ressources externes manuellement avant de considérer la facture comme arrêtée.

## Références Utiles

- Pricing Lightsail : https://aws.amazon.com/lightsail/pricing/
- Bundles Lightsail : https://docs.aws.amazon.com/lightsail/latest/userguide/amazon-lightsail-bundles.html
- Commande API bundles : https://docs.aws.amazon.com/boto3/latest/reference/services/lightsail/client/get_bundles.html
