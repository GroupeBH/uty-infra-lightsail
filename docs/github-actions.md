# GitHub Actions CI/CD

Le workflow `.github/workflows/deploy-lightsail.yml` se déclenche sur un push vers `master`.

Important : pour qu'un `push` du repository GitHub `uty-api` déclenche ce pipeline, ce fichier doit être présent dans ce repository `uty-api`. Si ce projet infra reste dans un repository séparé, GitHub ne déclenchera pas ce workflow sur les pushes de `uty-api` sans mécanisme supplémentaire de type `repository_dispatch`.

## Ce que fait le workflow

1. Configure les credentials AWS.
2. Installe Terraform et Ansible.
3. Écrit la clé SSH Lightsail depuis un secret GitHub.
4. Écrit `.env.production` depuis un secret GitHub.
5. Génère `terraform/backend.hcl` pour utiliser un backend S3 distant.
6. Valide Terraform et Ansible.
7. Lance `./deploy.sh`.

Le workflow ne construit pas l'image Docker. Il déploie une image déjà publiée sur Docker Hub. Par défaut, le tag déployé est :

1. l'input manuel `app_image_tag` si le workflow est lancé à la main ;
2. la variable GitHub `APP_IMAGE_TAG` si elle existe ;
3. sinon `latest`.

Par défaut, le workflow déploie donc `gbhsarl/uty-api:latest`.

## Secrets GitHub Requis

Dans `Settings > Secrets and variables > Actions > Secrets` :

- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
- `LIGHTSAIL_SSH_PRIVATE_KEY` : contenu complet de la clé privée Lightsail par défaut de la région.
- `APP_ENV_PRODUCTION` : contenu complet du fichier `.env.production`.
- `TERRAFORM_TFVARS` : contenu complet de `terraform/terraform.tfvars`, sans secrets applicatifs. Recommandé pour éviter de créer beaucoup de variables GitHub.

Ne pas stocker `.env.production` dans Git.

## Secret Terraform Tfvars

Vous pouvez mettre tout le contenu de `terraform/terraform.tfvars` dans un seul secret GitHub nommé `TERRAFORM_TFVARS`.

Exemple :

```hcl
aws_region        = "eu-central-1"
availability_zone = "eu-central-1a"
lightsail_bundle_id    = "micro_3_0"
lightsail_blueprint_id = "ubuntu_24_04"
admin_cidr = "TON_IP/32"
key_pair_name = ""
ssh_user      = "ubuntu"
domain_name = "api.uty-app.com"
```

Ne mettez pas les variables applicatives NestJS dans `TERRAFORM_TFVARS`; elles vont dans `APP_ENV_PRODUCTION`.

## Variables GitHub Optionnelles

Dans `Settings > Secrets and variables > Actions > Variables` :

- Aucune n'est strictement requise si vous gardez les valeurs par défaut du workflow.
- `APP_IMAGE_REPOSITORY` : optionnel. Défaut `gbhsarl/uty-api`.

Variables recommandées :

- `AWS_REGION` : défaut `eu-central-1`.
- `TF_BACKEND_BUCKET` : défaut `uty-lightsail-tfstate`. Remplacer seulement si vous changez de bucket.
- `TF_BACKEND_REGION` : défaut `AWS_REGION`.
- `TF_BACKEND_KEY` : défaut `uty-api-lightsail/terraform.tfstate`.
- `ADMIN_CIDR` : votre IP admin stable en `/32`.
- `DOMAIN_NAME` : défaut `api.uty-app.com`.
- `CADDY_EMAIL`.
- `HEALTHCHECK_PATH` : défaut `/health`.
- `APP_IMAGE_TAG` : optionnel. Défaut `latest`.
- `LIGHTSAIL_BUNDLE_ID` : défaut `micro_3_0`.
- `LIGHTSAIL_BLUEPRINT_ID` : défaut `ubuntu_24_04`.
- `INSTANCE_NAME` : défaut `uty-api-prod`.
- `STATIC_IP_NAME` : défaut `uty-api-prod-ip`.
- `SSH_USER` : défaut `ubuntu`.

## SSH Depuis GitHub Actions

Le firewall Lightsail limite SSH à `admin_cidr` plus `extra_ssh_cidrs`.

Dans GitHub Actions, l'IP du runner hébergé change à chaque exécution. Le workflow détecte l'IP publique du runner et l'ajoute temporairement à `extra_ssh_cidrs` via Terraform. Si `ADMIN_CIDR` n'est pas défini, le workflow utilise uniquement l'IP du runner pour SSH.

Pour un contrôle plus stable, utilisez un runner self-hosted avec une IP fixe et définissez `ADMIN_CIDR` sur cette IP en `/32`.

## Backend Terraform

Un runner GitHub est éphémère. Il ne faut pas utiliser un état Terraform local en CI/CD, sinon Terraform perdra la mémoire des ressources créées.

Le bucket S3 de state configuré pour ce projet est :

```text
TF_BACKEND_BUCKET=uty-lightsail-tfstate
TF_BACKEND_REGION=eu-central-1
TF_BACKEND_KEY=uty-api-lightsail/terraform.tfstate
```

Le bucket doit déjà exister. Ce projet ne crée pas ce bucket pour garder l'infra Lightsail simple.
