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

**3-2 Documentez votre playbook**

`playbook.yml` :

```yaml
- hosts: all
  gather_facts: true
  become: true

  roles:
    - docker
```

- `hosts: all` : le playbook s'applique à toutes les machines de l'inventaire.
- `gather_facts: true` : Ansible récupère les facts avant de lancer les tâches. On en a besoin pour `ansible_facts['distribution_release']` (`bookworm`) dans l'URL du dépôt Docker.
- `become: true` : toutes les tâches sont lancées en root (installation de paquets, gestion des services).
- `roles` : au lieu d'écrire les tâches dans le playbook, on appelle le rôle `docker`. Un rôle regroupe des tâches réutilisables dans une arborescence standard (créée avec `ansible-galaxy init roles/docker`). On n'a gardé que `tasks/` et `handlers/`.

Tâches du rôle `roles/docker/tasks/main.yml`, qui suivent la [doc d'installation de Docker sur Debian](https://docs.docker.com/engine/install/debian/#install-using-the-repository) :

1. `apt` : installe les prérequis (`ca-certificates`, `curl`, `gnupg`, `python3-venv`...) et met à jour le cache apt.
2. `apt_key` : ajoute la clé GPG de Docker, pour que apt fasse confiance aux paquets du dépôt.
3. `apt_repository` : ajoute le dépôt officiel de Docker pour la version de Debian du serveur.
4. `apt` : installe `docker-ce`. Si le paquet est installé ou mis à jour, la tâche notifie le handler `Restart docker`.
5. `apt` : installe `python3` et `python3-pip`.
6. `pip` : installe le SDK Docker pour Python dans un virtualenv `/opt/docker_venv`. Les modules Ansible `community.docker` en ont besoin. On passe par le module `pip` plutôt qu'un `command: pip install`, pour que la tâche ne soit relancée que si le paquet manque.
7. `service` : s'assure que le service Docker est démarré.

Le handler `roles/docker/handlers/main.yml` redémarre Docker. Un handler n'est lancé qu'une fois en fin de play, et seulement si une tâche l'a notifié.

Lancement :

```bash
ansible-playbook -i inventories/setup.yml playbook.yml --syntax-check
ansible-playbook -i inventories/setup.yml playbook.yml
```

Le premier lancement renvoie `changed=6`, le second `changed=0` : tout est déjà dans l'état demandé.

**3-3 Documentez la configuration de vos tâches docker_container**

Le playbook contient maintenant deux plays :

1. Le rôle `docker`, qui installe Docker (avec l'interpréteur Python système, nécessaire au module `apt`).
2. Les rôles de déploiement `network`, `database`, `app` et `proxy`, avec `ansible_python_interpreter: /opt/docker_venv/bin/python`. Les modules `community.docker` ont besoin du SDK Docker pour Python, qui est installé dans ce venv par le rôle `docker`.

Variables communes dans `group_vars/all.yml` : `dockerhub_user`, le nom du réseau (`docker_network`) et le mot de passe de la base (`postgres_password`).

| Rôle | Module | Configuration |
|---|---|---|
| `network` | `docker_network` | Crée le réseau `app-network`, qui permet aux conteneurs de se joindre par leur nom. |
| `database` | `docker_container` | Conteneur `database`, image `nassimjaz/tp-devops-database:latest`, variable `POSTGRES_PASSWORD`, volume `db-volume` monté sur `/var/lib/postgresql/data` pour garder les données si le conteneur est recréé. |
| `app` | `docker_container` | Conteneur `simple-api`, image `nassimjaz/tp-devops-simple-api:latest`, variables `DATABASE_HOST=database` (nom du conteneur de la base) et `DATABASE_PASSWORD`, lues par l'`application.yml`. Il doit s'appeler `simple-api`, car c'est l'hôte utilisé dans la conf du proxy. |
| `proxy` | `docker_container` | Conteneur `httpd`, image `nassimjaz/tp-devops-httpd:latest`, port `80:80`. C'est le seul conteneur exposé sur le serveur : la base et l'API ne sont joignables que via le réseau Docker. |

Options communes aux trois conteneurs :

- `networks: [{name: app-network}]` : branche le conteneur sur le réseau de l'application.
- `pull: true` : télécharge toujours la dernière version de l'image. Si elle a changé, le conteneur est recréé avec la nouvelle (indispensable pour le déploiement continu).
- `restart_policy: unless-stopped` : le conteneur redémarre s'il plante ou si le serveur redémarre. Ça règle aussi le cas où l'API démarre avant que la base soit prête.

`community.docker` doit être en version 3.10 minimum (`requirements.yml`, installé avec `ansible-galaxy collection install -r requirements.yml`). La 3.7 livrée avec le paquet `ansible` d'Ubuntu plante avec `Not supported URL scheme http+docker` à cause de `requests` 2.32.

À cause de `pull: true`, les trois tâches `docker_container` sont toujours marquées `changed`, même si l'image n'a pas changé (le module considère qu'un pull a eu lieu). Les conteneurs ne sont pas recréés pour autant.

Résultat :

```bash
$ curl http://nassim.jazouli.takima.school/departments/IRC/students
[{"id":1,"firstname":"Eli","lastname":"Copter","department":{"id":1,"name":"IRC"}}]
```

### Déploiement continu

Nouveau workflow `deploy.yml`, lancé quand `Build and push docker images` s'est terminé avec succès sur `main`. Il installe Ansible et la collection `community.docker`, écrit la clé SSH du serveur (secret `SSH_PRIVATE_KEY`) au chemin attendu par l'inventaire, ajoute le serveur aux `known_hosts`, puis lance le playbook. Grâce à `pull: true`, les conteneurs sont recréés avec les nouvelles images.

Chaîne complète : push sur `develop` → `Test backend` → PR auto vers `main` → merge → `Test backend` sur `main` → `Build and push docker images` → `Deploy`.

**Est-il vraiment sûr de déployer automatiquement chaque nouvelle image du hub ? Comment le sécuriser ?**

Non. Tout ce qui est poussé sous le tag `latest` part en production, sans contrôle humain : une image cassée qui passe les tests, ou une image modifiée par quelqu'un qui aurait récupéré le token Docker Hub, serait déployée directement. Et comme `latest` est écrasé à chaque fois, on ne sait plus exactement quelle version tourne et le retour arrière est compliqué.

Pour le sécuriser :

- déployer un tag précis (le SHA du commit, déjà poussé par le workflow de build) plutôt que `latest`, ce qui permet de savoir ce qui tourne et de revenir à la version précédente ;
- ne déployer qu'après les tests (c'est le cas ici : déploiement seulement si le build, lancé lui-même après les tests, a réussi) ;
- ajouter une validation manuelle avec un *environment* GitHub protégé (`environment: production` avec des reviewers obligatoires) ;
- scanner les images (Trivy, Docker Scout) et les signer (Docker Content Trust, cosign) pour n'accepter que des images vérifiées ;
- limiter les droits : token Docker Hub en lecture seule sur le serveur, secrets chiffrés (vault Ansible), clé SSH dédiée au déploiement ;
- vérifier après déploiement que l'application répond (healthcheck) et revenir en arrière sinon.

### Front

Le code du [front fourni](https://github.com/takima-training/devops-front) (Vue.js, servi par nginx) est copié dans `front/`.

- `front/.env.production` : `VUE_APP_API_URL=nassim.jazouli.takima.school/api`. Le front appelle l'API depuis le navigateur, il lui faut donc l'adresse publique du serveur et pas le nom du conteneur.
- `http-server/my-httpd.conf` : httpd devient le point d'entrée unique. `/api/...` est envoyé à l'API (`simple-api:8080`, en retirant le préfixe `/api`), tout le reste au front (`front:80`). La règle `/api/` est écrite avant `/` car httpd applique la première qui correspond.
- `build-and-push-docker-image.yml` : construit et pousse aussi l'image `tp-devops-front`.
- Nouveau rôle Ansible `front` : conteneur `front` sur le réseau `app-network`, sans port exposé (on y accède par le proxy). Il est lancé avant `proxy`.

Résultat : le site est sur http://nassim.jazouli.takima.school et l'API sur http://nassim.jazouli.takima.school/api/departments.

### Going further : vault

Le mot de passe de la base n'est plus en clair dans le repo. Il est chiffré avec `ansible-vault` :

- `group_vars/all/vault.yml` : contient `vault_postgres_password`, chiffré en AES256 (`ansible-vault encrypt`). Il est commité, mais illisible sans le mot de passe du vault.
- `group_vars/all/main.yml` : `postgres_password: "{{ vault_postgres_password }}"`. Les rôles utilisent toujours `postgres_password`, ils n'ont pas changé.
- Le mot de passe du vault n'est jamais dans le repo : en local dans `~/.ansible/vault_pass_tp` (droits 600), et dans le secret GitHub `ANSIBLE_VAULT_PASSWORD` pour la CI.

Commandes :

```bash
ansible-vault view group_vars/all/vault.yml --vault-password-file ~/.ansible/vault_pass_tp
ansible-vault edit group_vars/all/vault.yml --vault-password-file ~/.ansible/vault_pass_tp
ansible-playbook -i inventories/setup.yml playbook.yml --vault-password-file ~/.ansible/vault_pass_tp
```

Dans `deploy.yml`, le secret est écrit dans le dossier temporaire du runner et passé avec `--vault-password-file`.

Le mot de passe de la base a été changé au passage (l'ancien, `pwd`, était dans l'historique Git). Postgres ne lit `POSTGRES_PASSWORD` qu'à la création du volume : il a donc fallu supprimer une fois le conteneur `database` et le volume `db-volume` pour que le nouveau mot de passe soit pris en compte. Les données sont recréées par les scripts SQL de l'image.
