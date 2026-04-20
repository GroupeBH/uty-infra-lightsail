# Operations

Ce document regroupe les gestes courants après le premier déploiement.

## Se Connecter au Serveur

```bash
terraform -chdir=terraform output -raw ssh_command
ssh -i ~/.ssh/uty-lightsail.pem ubuntu@<IP_STATIQUE_LIGHTSAIL>
```

## Voir l'État des Conteneurs

```bash
cd /opt/uty-api
sudo docker compose ps
sudo docker compose logs -f --tail=200
```

Logs ciblés :

```bash
sudo docker compose logs -f --tail=200 app
sudo docker compose logs -f --tail=200 caddy
```

## Redéployer

Depuis le poste local :

```bash
export APP_IMAGE_REPOSITORY=dockerhub-user/uty-api
export APP_IMAGE_TAG=2026-04-18
export APP_ENV_FILE=.env.production
./deploy.sh
```

## Rollback

Revenir au dernier tag connu comme stable :

```bash
export APP_IMAGE_TAG=previous-known-good-tag
./deploy.sh
```

## Modifier l'Environnement Applicatif

Mettre à jour `.env.production` localement, puis relancer :

```bash
./deploy.sh
```

Le fichier est recopié vers `/opt/uty-api/.env` avec permission `0600`. Ne pas afficher ce fichier dans les logs, tickets ou captures.

## Healthcheck

Par défaut, Ansible vérifie `/health` via Caddy. Si l'API expose plutôt `/categories` :

```bash
export HEALTHCHECK_PATH=/categories
./deploy.sh
```

Si `DOMAIN_NAME` est défini, le healthcheck passe par `https://DOMAIN_NAME`. Le DNS doit donc pointer vers l'IP statique Lightsail.

## Certificats HTTPS

Caddy gère automatiquement les certificats Let's Encrypt quand `DOMAIN_NAME` pointe vers l'instance et que les ports 80/443 sont ouverts.

Sur le serveur :

```bash
cd /opt/uty-api
sudo docker compose logs -f caddy
```

## Mise à Jour Système

Pour une maintenance simple :

```bash
sudo apt update
sudo apt upgrade
sudo reboot
```

Après reboot :

```bash
cd /opt/uty-api
sudo docker compose ps
```

Les services Docker ont `restart: unless-stopped`.

## Snapshots

Créer un snapshot avant une opération risquée :

```bash
aws lightsail create-instance-snapshot \
  --instance-name uty-api-prod \
  --instance-snapshot-name uty-api-prod-$(date +%Y%m%d-%H%M)
```

Supprimer les vieux snapshots pour éviter une facture silencieuse.
