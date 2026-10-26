# TP3 – Ansible

## Inventaire

Fichier : [inventories/setup.yml](inventories/setup.yml)

```yaml
all:
 vars:
   ansible_user: admin
   ansible_ssh_private_key_file: ~/.ssh/id_rsa
 children:
   prod:
     hosts: pierre.ingrachen--boutantin.takima.school
```

- `all` : groupe racine, contient tous les hôtes.
- `vars` : variables appliquées à tous les hôtes.
  - `ansible_user` : utilisateur SSH utilisé pour la connexion (`admin`).
  - `ansible_ssh_private_key_file` : clé privée SSH utilisée pour s'authentifier.
- `children.prod` : groupe `prod`, qui contient le serveur du TP (Debian 12 « bookworm »).

> La clé privée `id_rsa` ne doit pas être commitée.

## Commandes de base

Toutes les commandes se lancent depuis le dossier `tp3`. `-i` indique l'inventaire,
`-m` le module, `-a` ses arguments, `--become` exécute la commande en root.

| Objectif | Commande |
|---|---|
| Tester la connexion SSH | `ansible all -i inventories/setup.yml -m ping` |
| Récupérer la distribution du serveur | `ansible all -i inventories/setup.yml -m setup -a "filter=ansible_distribution*"` |
| Installer Apache | `ansible all -i inventories/setup.yml -m apt -a "name=apache2 state=present" --become` |
| Désinstaller Apache | `ansible all -i inventories/setup.yml -m apt -a "name=apache2 state=absent" --become` |
| Créer la page d'accueil | `ansible all -i inventories/setup.yml -m shell -a 'echo "<html><h1>Hello World</h1></html>" > /var/www/html/index.html' --become` |
| Démarrer Apache | `ansible all -i inventories/setup.yml -m service -a "name=apache2 state=started" --become` |

Détails :

- `ping` vérifie que Ansible peut se connecter et exécuter Python sur l'hôte. Il ne s'agit pas d'un ping ICMP.
- `setup` collecte les facts (informations système) de l'hôte. `filter` limite les facts affichés.
- `apt` gère les paquets Debian : `present` installe, `absent` supprime.
- `shell` exécute une commande shell sur l'hôte.
- `service` gère les services : `started` démarre le service s'il est arrêté.
- Sans `--become`, `apt` échoue car l'installation de paquets demande les droits root.
- Ces modules sont idempotents (sauf `shell`) : relancer la commande ne change rien si l'état voulu est déjà atteint (`changed: false`).

## Playbook

Fichier : [playbook.yml](playbook.yml)

Lancement :

```bash
ansible-playbook -i inventories/setup.yml playbook.yml
```

Vérification sans modification : ajouter `--check`. Pour ne rejouer que le démarrage de Docker : `--tags docker`.

### Structure

- `hosts: all` : le playbook s'applique à tous les hôtes de l'inventaire.
- `gather_facts: true` : collecte les facts, nécessaires pour `ansible_facts['distribution_release']` (ici `bookworm`).
- `become: true` : toutes les tâches s'exécutent en root.

### Tâches

| Tâche | Module | Rôle |
|---|---|---|
| Install required packages | `apt` | Installe les prérequis (`apt-transport-https`, `ca-certificates`, `curl`, `gnupg`, `lsb-release`, `python3-venv`) et met à jour le cache apt. |
| Add Docker GPG key | `apt_key` | Ajoute la clé GPG officielle de Docker pour vérifier la signature des paquets. |
| Add Docker APT repository | `apt_repository` | Ajoute le dépôt stable de Docker pour Debian, selon la version de la distribution. |
| Install Docker | `apt` | Installe `docker-ce`. |
| Install Python3 and pip3 | `apt` | Installe `python3` et `python3-pip`. |
| Create a virtual environment for Docker SDK | `command` | Crée `/opt/docker_venv`. `creates` rend la tâche idempotente : elle est ignorée si le dossier existe. |
| Install Docker SDK for Python in virtual environment | `command` | Installe le paquet pip `docker` dans le venv, ce qui permet aux modules Ansible `docker_*` de piloter Docker. |
| Make sure Docker is running | `service` | Démarre le service `docker` s'il est arrêté (tag `docker`). |

### Remarques

- Un venv est utilisé car Debian 12 interdit `pip install` global (PEP 668).
- `command` n'est pas idempotent pour l'installation pip : la tâche est toujours `changed`.
- `state: latest` met à jour les prérequis à chaque exécution, `state: present` installe seulement s'ils sont absents.

## Déploiement de l'application (rôles)

Le playbook ne contient plus de tâches : il appelle des rôles, dans l'ordre.

```bash
ansible-playbook -i inventories/setup.yml playbook.yml
```

| Rôle | Contenu |
|---|---|
| `install_docker` | Installation de Docker, du venv Python et du SDK `docker` (tâches de la section précédente). |
| `create_network` | `docker_network` : crée le réseau `my-network`. |
| `launch_database` | `docker_container` : PostgreSQL. |
| `launch_app` | `docker_container` : API Spring Boot. |
| `launch_proxy` | `docker_container` : HTTPD en reverse proxy. |

Chaque rôle peut être lancé seul avec son tag (`--tags app`).

Variables communes : dans les `vars` de l'inventaire [inventories/setup.yml](inventories/setup.yml) (`dockerhub_username`, `docker_network_name`, `db_*`, `docker_python_interpreter`).

### 3-3 Configuration des tâches `docker_container`

Paramètres communs :

- `name` : nom du conteneur. Il sert aussi de nom DNS dans le réseau Docker (l'API joint la base via `database`, HTTPD joint l'API via `simple-api`, comme dans `my-httpd.conf`).
- `image` : image poussée sur DockerHub par la CI du TP2 (`<user>/tp-devops-database|simple-api|httpd:latest`).
- `pull: true` : retélécharge l'image à chaque exécution, pour récupérer un nouveau `latest`.
- `state: started` : crée le conteneur s'il n'existe pas et le démarre (idempotent).
- `restart_policy: always` : le conteneur redémarre après un reboot ou un crash.
- `networks` : rattache le conteneur à `my-network`, créé par le rôle `network`.
- `vars: ansible_python_interpreter` : les modules `docker_*` s'exécutent avec `/opt/docker_venv/bin/python`, seul interpréteur qui possède le SDK Docker (le Python système ne l'a pas, PEP 668).

Par conteneur :

| Conteneur | Paramètres spécifiques |
|---|---|
| `database` | `env` : `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD` (les variables du `.env` du TP1, passées directement au conteneur). `volumes: db-volume:/var/lib/postgresql/data` : les données survivent à la recréation du conteneur. Aucun port publié : la base n'est joignable que depuis le réseau Docker. |
| `simple-api` | `env` : `DATABASE_HOST=database` et `DATABASE_PASSWORD`, lus par `application.yml` via `${DATABASE_HOST}` / `${DATABASE_PASSWORD}` : l'image est déjà construite, Ansible n'a pas à modifier le fichier, il fournit les valeurs. Aucun port publié. |
| `httpd` | `ports: "80:80"` : seul conteneur exposé. Il fait le proxy vers `simple-api:8080`. |

Le mot de passe est dans l'inventaire ; pour un vrai déploiement il faudrait le chiffrer avec `ansible-vault`.

### Vérification

```bash
curl http://pierre.ingrachen--boutantin.takima.school/departments/IRC/students
```

## Continuous Deployment

Le job `deploy` est dans `.github/workflows/main.yml` du dépôt du TP2 (le code Ansible y est copié dans `ansible/`). Il s'exécute après `build-and-push-docker-image`, uniquement sur `main` :

1. installe Ansible ;
2. écrit la clé SSH (secret `SSH_PRIVATE_KEY`) et enregistre l'hôte avec `ssh-keyscan` ;
3. lance `ansible-playbook` (la clé et le compte DockerHub sont passés par `-e`, ce qui prend le pas sur l'inventaire) ;
4. vérifie avec `curl --fail` que l'API répond.

`pull: true` dans les tâches `docker_container` fait que le conteneur n'est recréé que si l'image du hub a changé.

Secrets GitHub nécessaires : `SSH_PRIVATE_KEY`, `SSH_HOST` (en plus de `DOCKERHUB_USERNAME` et `DOCKERHUB_TOKEN` déjà présents).

### Est-ce sûr de déployer automatiquement chaque nouvelle image ?

Non, pas tel quel :

- Le tag `latest` est mutable : n'importe qui ayant accès au compte DockerHub, ou un token volé, peut pousser une image malveillante qui sera déployée en production sans contrôle.
- Une image qui contient un bug ou une faille est déployée immédiatement, sans validation ni retour arrière simple.
- Rien ne prouve que l'image vient bien de notre CI.
- La clé SSH de déploiement donne un accès au serveur : si elle fuit (logs, fork, PR malveillante), le serveur est compromis.
- Les secrets, dont le mot de passe de la base, sont en clair dans le dépôt.

Pour sécuriser :

- Déployer uniquement depuis `main` et après réussite des tests (déjà le cas : `needs` et `if`).
- Utiliser des tags immuables (SHA du commit ou version) au lieu de `latest`, ce qui permet aussi de revenir en arrière.
- Scanner les images (Trivy) et les signer (cosign), puis vérifier la signature avant de déployer.
- Protéger `main` (revue obligatoire) et utiliser un GitHub Environment `production` avec approbation manuelle.
- Activer l'authentification à deux facteurs sur DockerHub et utiliser des tokens à droits minimaux.
- Utiliser un utilisateur SSH dédié au déploiement, avec des droits limités.
- Chiffrer les mots de passe avec `ansible-vault` ou les mettre dans les secrets GitHub.
- Ajouter un contrôle de santé après déploiement (déjà fait avec `curl --fail`) et un rollback automatique.

## Front

Code : [takima-training/devops-front](https://github.com/takima-training/devops-front) (Vue.js + nginx), copié dans `tp2/front/`.

- Le front appelle l'API avec `fetch("http://" + VUE_APP_API_URL + "/departments...")`, valeur fixée à la compilation. `front/.env.production` contient `VUE_APP_API_URL=pierre.ingrachen--boutantin.takima.school/api`.
- `http-server/my-httpd.conf` (HTTPD = reverse proxy) : `/api` est redirigé vers `simple-api:8080` (le préfixe est retiré), tout le reste vers `front:80`. Tout passe par le port 80, donc pas de problème CORS.
- Rôle Ansible `launch_front` : conteneur `front` sur `my-network`, lancé avant `launch_proxy` (HTTPD ne démarre pas si l'hôte `front` est introuvable).
- CI : le job de build pousse aussi `tp-devops-front`, et le job `deploy` vérifie l'API (`/api/departments/IRC/students`) et la page d'accueil.
- L'API n'est plus à `/departments` mais à `/api/departments`.
