# Tutoriel : Environnement de Développement React avec Docker et Ansible

Ce tutoriel vous guidera pas à pas dans la mise en place d'un environnement de développement React en utilisant deux conteneurs Docker, où l'un servira de nœud de contrôle Ansible pour configurer l'autre.

## Prérequis

Avant de commencer, assurez-vous d'avoir les éléments suivants installés sur votre machine hôte :

*   **Docker** : Pour la gestion des conteneurs.
*   **Docker Compose** : Pour définir et exécuter des applications Docker multi-conteneurs.

## Architecture de l'environnement

Nous allons créer deux conteneurs Docker :

1.  **`ansible_control`** : Ce conteneur agira comme le nœud de contrôle Ansible. Il contiendra Ansible et sera responsable de l'exécution des playbooks.
2.  **`ansible_managed`** : Ce conteneur sera le nœud géré, la cible sur laquelle les playbooks Ansible seront exécutés pour installer l'environnement React.

## Étape 1 : Création de la structure du projet

Commencez par créer un répertoire pour votre projet et naviguez-y :

```bash
mkdir -p docker-ansible-react
cd docker-ansible-react
```

Ensuite, créez les sous-répertoires pour nos Dockerfiles :

```bash
mkdir ansible_control ansible_managed
```

## Étape 2 : Création des Dockerfiles

### Dockerfile pour `ansible_control` (Nœud de contrôle Ansible)

Ce Dockerfile installera Python, Ansible et le client OpenSSH. Nous utiliserons Ubuntu 22.04 comme image de base.

Créez le fichier `ansible_control/Dockerfile` avec le contenu suivant :

```dockerfile
FROM ubuntu:22.04

RUN apt-get update && \
    apt-get install -y python3 python3-pip openssh-client && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

RUN pip install ansible

# Générer une paire de clés SSH pour l'accès sans mot de passe
RUN ssh-keygen -t rsa -b 2048 -f /root/.ssh/id_rsa -N ""

# Ajouter l'hôte cible à known_hosts pour éviter les invites de confirmation
# Ceci sera fait dynamiquement plus tard ou via un playbook Ansible
```

### Dockerfile pour `ansible_managed` (Nœud géré)

Ce Dockerfile installera Python (nécessaire pour Ansible), le serveur OpenSSH et `sudo` sur une image Ubuntu 22.04.

Créez le fichier `ansible_managed/Dockerfile` avec le contenu suivant :

```dockerfile
FROM ubuntu:22.04

RUN apt-get update && \
    apt-get install -y openssh-server python3 sudo && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Créer un utilisateur 'ansible' et lui donner les privilèges sudo sans mot de passe
RUN useradd -rm -d /home/ansible -s /bin/bash ansible && \
    echo "ansible:ansible" | chpasswd && \
    echo "ansible ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/ansible

# Configurer SSH pour permettre la connexion root et l'authentification par clé
RUN mkdir /var/run/sshd
COPY sshd_config /etc/ssh/sshd_config

EXPOSE 22

CMD ["/usr/sbin/sshd", "-D"]
```

Créez également le fichier `ansible_managed/sshd_config` :

```
PermitRootLogin yes
StrictModes no
PasswordAuthentication yes
PubkeyAuthentication yes
AuthorizedKeysFile	.ssh/authorized_keys

Subsystem sftp /usr/lib/openssh/sftp-server
```

**Note** : `PermitRootLogin yes` est utilisé ici pour simplifier le tutoriel. Dans un environnement de production, il est recommandé d'utiliser des utilisateurs non-root avec des clés SSH.

## Étape 3 : Création du fichier Docker Compose

Le fichier `docker-compose.yml` définira nos deux services, leurs dépendances et les volumes partagés.

Créez le fichier `docker-compose.yml` à la racine de votre projet avec le contenu suivant :

```yaml
version: '3.8'

services:
  ansible_control:
    build: ./ansible_control
    container_name: ansible_control
    volumes:
      - ./ansible_control:/ansible
      - ./ansible_playbook:/etc/ansible/playbook
    command: tail -f /dev/null # Garde le conteneur en vie

  ansible_managed:
    build: ./ansible_managed
    container_name: ansible_managed
    volumes:
      - ./ansible_managed:/app
    ports:
      - "3000:3000" # Pour l'application React
    command: /usr/sbin/sshd -D # Démarre le serveur SSH
```

## Étape 4 : Configuration SSH entre les conteneurs

Pour qu'Ansible puisse se connecter au nœud géré sans mot de passe, nous devons copier la clé publique SSH du nœud de contrôle vers le nœud géré.

1.  **Construire et démarrer les conteneurs** :

    ```bash
    docker-compose build
    docker-compose up -d
    ```

2.  **Récupérer la clé publique SSH du nœud de contrôle** :

    ```bash
    docker exec ansible_control cat /root/.ssh/id_rsa.pub > ansible_control/id_rsa.pub
    ```

3.  **Copier la clé publique vers le nœud géré** :

    ```bash
    docker cp ansible_control/id_rsa.pub ansible_managed:/home/ansible/.ssh/authorized_keys
    docker exec ansible_managed chown ansible:ansible /home/ansible/.ssh/authorized_keys
    docker exec ansible_managed chmod 600 /home/ansible/.ssh/authorized_keys
    ```

4.  **Ajouter l'hôte géré au `known_hosts` du nœud de contrôle** :

    ```bash
    MANAGED_IP=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' ansible_managed)
    docker exec ansible_control ssh-keyscan $MANAGED_IP >> ansible_control/known_hosts
    docker cp ansible_control/known_hosts ansible_control:/root/.ssh/known_hosts
    ```

## Étape 5 : Création du Playbook Ansible pour React

Créez un répertoire `ansible_playbook` à la racine de votre projet :

```bash
mkdir ansible_playbook
```

Dans ce répertoire, créez le fichier `ansible_playbook/react_setup.yml` avec le contenu suivant :

```yaml
---
- name: Setup React Development Environment
  hosts: all
  become: yes
  tasks:
    - name: Update apt cache
      apt:
        update_cache: yes

    - name: Install Node.js and npm (using NodeSource)
      shell: |
        curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
        apt-get install -y nodejs
      args:
        warn: no

    - name: Install create-react-app globally (optional, npx is preferred)
      npm:
        name: create-react-app
        global: yes
        state: present

    - name: Create React App
      command: npx create-react-app my-react-app
      args:
        chdir: /app
        creates: /app/my-react-app

    - name: Install dependencies for React App
      npm:
        path: /app/my-react-app
        state: present

    - name: Start React App (in background)
      command: npm start
      args:
        chdir: /app/my-react-app
      async: 1000 # Exécute en arrière-plan
      poll: 0 # Ne pas attendre la fin de la commande

    - name: Display message about React App URL
      debug:
        msg: "Your React application should be running on http://{{ ansible_host }}:3000"
```

Créez également le fichier `ansible_playbook/hosts.ini` (votre inventaire Ansible) :

```ini
[react_app_servers]
ansible_managed ansible_host={{ MANAGED_IP }} ansible_user=ansible ansible_ssh_private_key_file=/ansible/.ssh/id_rsa
```

**Note** : Nous utiliserons une variable `MANAGED_IP` qui sera remplacée dynamiquement.

## Étape 6 : Exécution du Playbook Ansible

Maintenant, nous allons exécuter le playbook depuis le conteneur `ansible_control`.

1.  **Récupérer l'adresse IP du conteneur géré** :

    ```bash
    MANAGED_IP=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' ansible_managed)
    ```

2.  **Exécuter le playbook Ansible** :

    ```bash
    docker exec -e MANAGED_IP=$MANAGED_IP ansible_control ansible-playbook -i /etc/ansible/playbook/hosts.ini /etc/ansible/playbook/react_setup.yml
    ```

## Étape 7 : Vérification de l'application React

Une fois le playbook exécuté, votre application React devrait être en cours d'exécution dans le conteneur `ansible_managed`.

Ouvrez votre navigateur et accédez à `http://localhost:3000`.

Vous devriez voir l'application React par défaut.

## Nettoyage

Pour arrêter et supprimer les conteneurs :

```bash
docker-compose down
```

Pour supprimer tous les fichiers créés :

```bash
rm -rf docker-ansible-react
```

## Conclusion

Vous avez maintenant un environnement de développement React entièrement configuré et automatisé à l'aide de Docker et Ansible. Cette approche offre une grande reproductibilité et facilite la gestion de vos environnements de développement.
