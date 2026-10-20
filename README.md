# TP DevOps Correction Docker

Correction de la partie Docker du module DevOps. Amusez-vous bien avec GitHub Actions !

## TP 2 - GitHub Actions

### Workflows

- `test-backend.yml` : lancé sur `main` et `develop`. Build + tests (unitaires et intégration) avec `mvn verify`, puis analyse SonarCloud. Sur `develop`, si les tests passent, une PR vers `main` est créée automatiquement.
- `build-and-push-docker-image.yml` : lancé uniquement sur `main`, quand `Test backend` est terminé avec succès. Build et push des 3 images (api, database, httpd) sur Docker Hub.

La branche `main` est protégée : on ne peut pas push directement dessus, il faut passer par une PR et le check `test-backend` doit être vert.

### Questions

**2-1 Qu'est-ce que les testcontainers ?**

Ce sont des librairies Java qui lancent des conteneurs Docker pendant les tests. Ici, un conteneur PostgreSQL est lancé pour les tests d'intégration, ce qui permet de tester avec une vraie base sans rien installer.

**2-2 Pourquoi utiliser des variables sécurisées ?**

Pour ne pas mettre nos identifiants (token Docker Hub, token Sonar) en clair dans le code. Les secrets GitHub sont chiffrés et masqués dans les logs.

**2-3 Pourquoi mettre `needs: test-backend` sur le job ?**

Sans `needs`, les jobs tournent en parallèle, donc on pourrait publier une image même si les tests échouent. Avec `needs`, le build des images n'est lancé que si les tests sont passés.

**2-4 Pourquoi pousser les images Docker ?**

Pour les stocker sur un registry (Docker Hub) et pouvoir les récupérer depuis n'importe quelle machine, par exemple pour déployer l'application, sans avoir à les reconstruire.
