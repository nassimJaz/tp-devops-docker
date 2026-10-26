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

## TP 3 - Ansible

Tout se trouve dans le dossier `ansible/`. Les commandes ci-dessous sont lancées depuis ce dossier.

Le serveur est `nassim.jazouli.takima.school` (Debian 12), on s'y connecte avec l'utilisateur `admin` et la clé privée fournie (`~/.ssh/id_rsa_takima`).

### Questions

**3-1 Documentez votre inventaire et les commandes de base**

Inventaire `inventories/setup.yml` :

```yaml
all:
  vars:
    ansible_user: admin
    ansible_ssh_private_key_file: ~/.ssh/id_rsa_takima
  children:
    prod:
      hosts: nassim.jazouli.takima.school
```

- `all` est le groupe qui contient tous les hôtes, les `vars` définies ici s'appliquent donc à toutes les machines : l'utilisateur SSH et la clé privée.
- `children` permet de créer des sous-groupes. Ici un groupe `prod` qui contient notre unique serveur.

Commandes de base (ad hoc) :

- `ansible all -i inventories/setup.yml -m ping` : teste la connexion. Ansible répond `pong` si la machine est joignable, que l'utilisateur existe et que l'authentification SSH fonctionne.
- `ansible all -i inventories/setup.yml -m setup -a "filter=ansible_distribution*"` : récupère les *facts*, c'est-à-dire les informations découvertes sur la machine (ici Debian 12.7, `bookworm`).
- `ansible all -i inventories/setup.yml -m apt -a "name=apache2 state=absent" --become` : supprime apache2. `--become` exécute la commande en root (nécessaire pour installer ou supprimer un paquet).

On décrit l'état voulu (`state=absent`) et Ansible ne fait que ce qui est nécessaire : la première exécution renvoie `CHANGED`, la seconde `SUCCESS` car apache2 est déjà absent.
