# Migration Depuis EC2

Objectif : migrer progressivement l'API vers Lightsail sans interrompre l'app Play Store en test fermé.

## Étapes Recommandées

1. Déployer Lightsail avec `DOMAIN_NAME=` pour tester en HTTP par IP statique.
2. Vérifier que Docker, Caddy et l'API démarrent correctement.
3. Tester les endpoints principaux depuis un poste externe.
4. Créer le record DNS `A api.uty-app.com` vers l'IP statique Lightsail.
5. Relancer `./deploy.sh` avec `DOMAIN_NAME=api.uty-app.com` et `CADDY_EMAIL` défini.
6. Vérifier HTTPS et les logs Caddy.
7. Surveiller les erreurs applicatives, la latence et les retours de l'app mobile.
8. Garder l'ancienne infra EC2 active pendant une fenêtre d'observation.
9. Détruire ou arrêter l'ancienne infra seulement après validation.

## Tests Avant Bascule DNS

Si le domaine pointe encore vers EC2, tester par IP :

```bash
curl -i http://<IP_STATIQUE_LIGHTSAIL>/health
curl -i http://<IP_STATIQUE_LIGHTSAIL>/categories
```

Selon la configuration CORS ou les guards applicatifs, certains endpoints peuvent nécessiter des headers ou un token.

## Bascule DNS

Réduire le TTL DNS avant migration si possible. Ensuite :

```text
api.uty-app.com.  A  <IP_STATIQUE_LIGHTSAIL>
```

Après propagation :

```bash
dig +short api.uty-app.com
curl -i https://api.uty-app.com/health
```

## Rollback DNS

Si un incident apparaît :

1. Replacer le record `A` vers l'ancienne IP EC2.
2. Remettre le dernier tag Docker stable côté Lightsail si nécessaire.
3. Comparer les logs NestJS, Caddy et l'ancien reverse proxy.

Ne pas supprimer l'ancienne infra tant que la nouvelle n'a pas passé une période d'observation suffisante.
