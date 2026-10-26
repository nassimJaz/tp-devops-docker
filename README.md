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
