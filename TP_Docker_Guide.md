# TP Docker : Guide Pas à Pas

Ce TP vous guidera à travers les concepts fondamentaux de Docker et vous permettra de répondre aux questions du fichier Questions.txt.

---

## Prérequis

- Docker installé sur votre machine
- Accès au terminal/commande prompt

Pour vérifier que Docker est installé, exécutez :

```bash
docker --version
```

---

## Étape 1 : Comprendre Docker

### Qu'est-ce que Docker ?

Docker est une **plateforme devirtualisation** qui permet de créer, déployer et exécuter des applications dans des **conteneurs**.

### L'avantage principal des conteneurs Docker

Les conteneurs Docker sont **légers** et **isolés** du système hôte. Ils contiennent tout ce qui est nécessaire pour faire fonctionner une application (code, bibliothèques, dépendances) sans installer quoi que ce soit sur la machine hôte.

**Pratique !** Exécutez cette commande pour voir les informations de Docker :

```bash
docker info
```

---

## Étape 2 : Les Images Docker

### Qu'est-ce qu'une image Docker ?

Une **image Docker** est un modèle en lecture seule utilisé pour créer des conteneurs. Elle contient le système de fichiers et les applications nécessaires.

### Docker Hub

**Docker Hub** est un registry public (site web) où vous pouvez trouver et partager des images Docker. C'est comme un "GitHub" pour les images Docker.

**Pratique !** Recherchez une image sur Docker Hub :

```bash
docker search ubuntu
```

---

## Étape 3 : Les Conteneurs Docker

### Qu'est-ce qu'un conteneur Docker ?

Un **conteneur Docker** est une instance en cours d'exécution d'une image Docker. C'est un environnement isolé qui exécute votre application.

### Liste des conteneurs en cours d'exécution

```bash
docker ps
```

Pour voir **tous** les conteneurs (y compris ceux qui sont arrêtés) :

```bash
docker ps -a
```

---

## Étape 4 : Créer et exécuter un conteneur

### Commande pour démarrer un conteneur

La commande de base est :

```bash
docker run <nom_de_l_image>
```

**Pratique !** Lancez un conteneur Ubuntu simple :

```bash
docker run hello-world
```

Essayez avec Ubuntu en mode interactif :

```bash
docker run -it ubuntu bash
```

Cette commande :
- `-i` : mode interactif
- `-t` : alloue un pseudo-terminal
- `ubuntu` : le nom de l'image
- `bash` : la commande à exécuter

Pour quitter le conteneur, tapez `exit`.

---

## Étape 5 : Le Dockerfile

### À quoi sert le fichier Dockerfile ?

Un **Dockerfile** est un fichier texte qui contient des instructions pour construire une image Docker automatiquement. Il définit :
- L'image de base
- Les commandes à exécuter
- Les fichiers à copier
- La commande de démarrage

**Exemple de Dockerfile simple :**

```dockerfile
FROM ubuntu:20.04
RUN apt-get update
RUN apt-get install -y nginx
CMD ["nginx", "-g", "daemon off;"]
```

---

## Étape 6 : Construire une image

### Commande pour construire une image à partir d'un Dockerfile

```bash
docker build -t nom_de_l_image .
```

L'option `-t` permet de taguer (nommer) l'image.

**Pratique !** Créez un répertoire de test et un Dockerfile simple :

```bash
mkdir tp-docker
cd tp-docker
```

Créez un fichier `Dockerfile` avec ce contenu :

```dockerfile
FROM alpine
RUN echo "Hello from my first Docker image!" > /hello.txt
CMD cat /hello.txt
```

Construisez l'image :

```bash
docker build -t mon-image-test .
```

Exécutez le conteneur :

```bash
docker run mon-image-test
```

---

## Étape 7 : Les Volumes Docker

### Qu'est-ce qu'un volume Docker ?

Un **volume Docker** est un mécanisme pour persister les données générées par un conteneur. Il permet de partager des données entre le conteneur et la machine hôte, ou entre plusieurs conteneurs.

**Pratique !** Créez un volume :

```bash
docker volume create mon_volume
```

Montez un volume dans un conteneur :

```bash
docker run -v mon_volume:/data alpine
```

---

## Étape 8 : Docker Compose

### Qu'est-ce que docker-compose ?

**Docker Compose** est un outil pour définir et exécuter des applications multi-conteneurs. Vous utilisez un fichier `docker-compose.yml` pour configurer tous les services de votre application.

**Exemple de fichier docker-compose.yml :**

```yaml
version: '3'
services:
  web:
    image: nginx
    ports:
      - "8080:80"
  db:
    image: mysql
    environment:
      MYSQL_ROOT_PASSWORD: motdepasse
```

**Commandes utiles :**
- Démarrer les services : `docker-compose up`
- Arrêter les services : `docker-compose down`
- Voir les logs : `docker-compose logs`

---

## Résumé des commandes essentielles

| Action | Commande |
|--------|----------|
| Lister les conteneurs | `docker ps` |
| Lister tous les conteneurs | `docker ps -a` |
| Démarrer un conteneur | `docker run <image>` |
| Construire une image | `docker build -t <nom> .` |
| Créer un volume | `docker volume create <nom>` |
| Voir les images | `docker images` |
| Supprimer un conteneur | `docker rm <id>` |
| Supprimer une image | `docker rmi <id>` |

---

## Réponses aux questions

Voici les réponses aux questions du fichier Questions.txt :

1. **Qu'est-ce que Docker selon le document ?**  
   Docker est une plateforme de virtualisation qui permet de créer, déployer et exécuter des applications dans des conteneurs.

2. **Quel est l'avantage principal des conteneurs Docker ?**  
   Ils sont légers et isolés du système hôte, contenant tout ce nécessaire sans installation sur la machine hôte.

3. **Qu'est-ce qu'une image Docker ?**  
   Un modèle en lecture seule utilisé pour créer des conteneurs, contenant le système de fichiers et les applications.

4. **Qu'est-ce qu'un conteneur Docker ?**  
   Une instance en cours d'exécution d'une image Docker, c'est un environnement isolé.

5. **À quoi sert le fichier Dockerfile ?**  
   Il contient des instructions pour construire automatiquement une image Docker.

6. **Qu'est-ce que Docker Hub ?**  
   Un registry public où trouver et partager des images Docker.

7. **Quelle commande permet de lister les conteneurs en cours d'exécution ?**  
   `docker ps`

8. **Quelle commande permet de démarrer un conteneur ?**  
   `docker run <nom_de_l_image>`

9. **Qu'est-ce qu'un volume Docker ?**  
   Un mécanisme pour persister les données générées par un conteneur.

10. **Quelle commande permet de construire une image à partir d'un Dockerfile ?**  
    `docker build -t nom_de_l_image .`

11. **Qu'est-ce que docker-compose ?**  
    Un outil pour définir et exécuter des applications multi-conteneurs avec un fichier YAML.

---

*TP Docker - Exercices guidés*
