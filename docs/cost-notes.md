# Cost Notes

Objectif : rester proche du coût fixe du plan Lightsail Linux/Unix à environ 7 USD/mois pour la première version.

## Ressources Créées

- 1 instance Lightsail.
- 1 IP statique Lightsail attachée à l'instance.
- Règles firewall Lightsail pour SSH, HTTP et HTTPS.

Aucun service suivant n'est créé :

- EC2
- VPC custom
- NAT Gateway
- ALB
- Auto Scaling Group
- ECS
- RDS
- base de données Lightsail

## Bundle Par Défaut

`lightsail_bundle_id = "micro_3_0"`

Ce bundle correspond au plan Linux/Unix avec IPv4 public autour de 7 USD/mois, 1 GB RAM, 2 vCPU, 40 GB SSD et environ 2 TB de transfert mensuel.

AWS peut faire évoluer les bundles. Vérifier avant production :

```bash
aws lightsail get-bundles --include-inactive
```

Puis remplacer dans `terraform/terraform.tfvars` :

```hcl
lightsail_bundle_id = "nouvel_identifiant"
```

## IP Statique

L'IP statique est incluse si elle est attachée à une instance. Une IP statique non attachée peut être facturée. Ne pas laisser d'IP orpheline.

## Snapshots

Les snapshots sont facturés séparément selon le volume stocké. Ils sont utiles avant migration, mise à jour majeure ou suppression d'ancienne infra, mais il faut une politique de rétention.

## Transfert

Le plan inclut un quota mensuel. Le trafic entrant et sortant compte dans l'allocation, mais les frais d'excès concernent surtout le trafic sortant selon les règles AWS. Surveiller la consommation si l'app Play Store commence à croître.

## Quand Monter de Taille

Le plan 1 GB / 2 vCPU est raisonnable pour un test fermé et environ 1000 comptes inscrits si la concurrence réelle reste modérée et si la base de données/Redis sont externes.

Surveiller :

- CPU burst et charge moyenne.
- mémoire disponible.
- latence API.
- erreurs 5xx.
- temps de réponse de la base externe.
- logs Caddy et NestJS.

Si la charge devient soutenue, envisager d'abord un bundle Lightsail supérieur. Ne migrer vers ALB/ECS/RDS que lorsque le besoin opérationnel justifie la complexité et le coût.
