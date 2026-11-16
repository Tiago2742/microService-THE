# Projet fil rouge

## Introduction

Ce projet est une application microservices construite avec Node.js et React. Il est conçu pour être déployé sur Kubernetes.

## Table des matières

- [Introduction](#introduction)
- [Table des matières](#table-des-matières)
- [Architecture](#architecture)
- [Chemins d'Ingress](#chemins-dingress)
- [Noms de Services Kubernetes](#noms-de-services-kubernetes)
- [Ports des Services](#ports-des-services)
- [Prérequis](#prérequis)
- [Installation](#installation)
- [Déploiement](#déploiement)
- [Arrêt](#arrêt)
- [Mise à jour d'un service](#mise-à-jour-dun-service)
- [Commandes utiles](#commandes-utiles)

## Architecture

L'application est composée des services suivants :

- **Client** : Interface utilisateur construite avec React.
- **Posts** : Service pour la gestion des posts.
- **Comments** : Service pour la gestion des commentaires.
- **Query** : Service pour la gestion des requêtes.
- **Moderation** : Service pour la modération des commentaires.
- **Event Bus** : Service pour la gestion des événements entre les services.

### Chemins d'Ingress

- `/posts/create` : Dirigé vers le service `posts-clusterip-srv` sur le port 4000.
  - Utilisé pour créer de nouveaux posts.
  
- `/posts` : Dirigé vers le service `query-srv` sur le port 4002.
  - Utilisé pour récupérer la liste des posts existants.
  
- `/posts/?(.*)/comments` : Dirigé vers le service `comments-srv` sur le port 4001.
  - Utilisé pour créer ou récupérer les commentaires associés à un post spécifique.
  
- `/?(.*)` : Dirigé vers le service `client-srv` sur le port 3000.
  - Utilisé pour accéder à l'interface utilisateur.

### Noms de Services Kubernetes

Assurez-vous que les noms de services dans vos fichiers de déploiement Kubernetes correspondent aux noms de services utilisés dans le code de l'application. Voici les noms de services attendus :

- **client-srv**: Service pour l'interface utilisateur.
- **posts-clusterip-srv**: Service pour la gestion des posts.
- **query-srv**: Service pour la gestion des requêtes.
- **comments-srv**: Service pour la gestion des commentaires.
- **moderation-srv**: Service pour la modération des commentaires.
- **event-bus-srv**: Service pour la gestion des événements entre les services.

Si vous modifiez ces noms, assurez-vous également de mettre à jour les références correspondantes dans le code de l'application.

### Ports des Services

Chaque service écoute sur un port spécifique. Assurez-vous que ces ports sont correctement configurés dans vos fichiers de déploiement Kubernetes et dans tout autre outil de gestion des conteneurs que vous pourriez utiliser. Voici les ports attendus pour chaque service :

- **client-srv**: Écoute sur le port 3000.
- **posts-clusterip-srv**: Écoute sur le port 4000.
- **query-srv**: Écoute sur le port 4002.
- **comments-srv**: Écoute sur le port 4001.
- **moderation-srv**: Écoute sur le port 4003.
- **event-bus-srv**: Écoute sur le port 4005.

Si vous modifiez ces ports, assurez-vous également de mettre à jour les références correspondantes dans le code de l'application et les fichiers de configuration Kubernetes.

## Prérequis

- **Docker Desktop** avec **Kubernetes activé** (Settings > Kubernetes > Enable Kubernetes)
- **kubectl** (fourni avec Docker Desktop)
- Le contrôleur **ingress-nginx** installé dans le cluster (voir ci-dessous)

Vérifier que tout est prêt :

```bash
docker version
kubectl get nodes
```

`kubectl get nodes` doit afficher un node `docker-desktop` en statut `Ready`.

## Installation

1. Clonez ce dépôt :

    ```bash
    git clone https://github.com/Tiago2742/microService-THE.git
    cd microService-THE
    ```

2. Installez le contrôleur ingress-nginx (une seule fois par cluster) :

    ```bash
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.15.1/deploy/static/provider/cloud/deploy.yaml
    ```

    Attendez qu'il soit prêt (statut `Running`) :

    ```bash
    kubectl get pods -n ingress-nginx
    ```

    Pas besoin de `npm install` en local : les dépendances sont installées dans les images Docker au moment du build.

## Déploiement

Toutes les commandes se lancent depuis la racine du dépôt.

1. Construisez les 6 images Docker (basées sur **node:alpine**) :

    ```bash
    docker build -t client ./client
    docker build -t posts ./posts
    docker build -t comments ./comments
    docker build -t query ./query
    docker build -t moderation ./moderation
    docker build -t event-bus ./event-bus
    ```

    Vérifiez qu'elles existent :

    ```bash
    docker images
    ```

2. Déployez tous les objets Kubernetes (6 Deployments + Services, 1 Ingress) :

    ```bash
    kubectl apply -f k8s/
    ```

3. Vérifiez que les 6 pods sont en statut `Running` :

    ```bash
    kubectl get pods
    ```

4. Ouvrez l'application dans le navigateur : **http://localhost**

    Test rapide : créez un post, ajoutez un commentaire, puis rafraîchissez la page. Un commentaire contenant le mot « orange » doit apparaître comme rejeté.

## Arrêt

Supprimer tous les objets déployés (les images Docker restent sur la machine) :

```bash
kubectl delete -f k8s/
```

## Mise à jour d'un service

Après une modification du code d'un service (exemple avec `posts`) :

```bash
docker build -t posts ./posts
kubectl rollout restart deployment posts-depl
```

Le `rollout restart` est nécessaire : l'image garde le même nom (`posts:latest`), Kubernetes ne détecte donc pas le changement tout seul.

## Commandes utiles

| Commande | Usage |
| --- | --- |
| `kubectl get all` | Vue d'ensemble des pods, deployments et services |
| `kubectl get ingress` | Vérifier l'Ingress |
| `kubectl logs <nom-du-pod>` | Voir les logs d'un service |
| `kubectl describe pod <nom-du-pod>` | Comprendre pourquoi un pod ne démarre pas |
| `kubectl delete pod <nom-du-pod>` | Supprimer un pod : le Deployment en recrée un automatiquement |

### Problèmes fréquents

- **`ImagePullBackOff` / `ErrImagePull`** : l'image n'a pas été construite. Relancez le `docker build` correspondant.
- **`CrashLoopBackOff`** : le conteneur plante au démarrage. Consultez `kubectl logs <nom-du-pod>`.
- **http://localhost ne répond pas** : vérifiez que le contrôleur ingress-nginx tourne (`kubectl get pods -n ingress-nginx`) et que le port 80 n'est pas déjà utilisé par un autre programme.