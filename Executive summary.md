---
title: Brief6
tags: presentation
slideOptions:
  theme: moon
  transition: 'fade'
  spotlight:
    enabled: true
---

# Brief 6 : Kubernetes

---

## Contexte et objectifs

----

### Kubernetes

Outil permettant d’automatiser le déploiement, la mise à l’échelle et la gestion des conteneurs, qui contiennent les ressources nécessaires pour faire tourner l’application.

----

**Plan de contrôle** : ensemble de processus qui contrôle les nœuds et assigne les tâches
![](https://i.imgur.com/YX5A9wj.png)

----

**Controller-manager** : maintient l'état des applications
![](https://i.imgur.com/YX5A9wj.png)

----

**Serveur API** : permet la configuration des objets (Deployment, Service, Volume, Secrets...)
![](https://i.imgur.com/YX5A9wj.png)

----

**Nœuds** : serveurs physiques ou virtuels qui les tâches assignées par le plan de contrôle
![](https://i.imgur.com/YX5A9wj.png)

----

**Pod** : un ou plusieurs conteneurs déployés sur un seul nœud
![](https://i.imgur.com/YX5A9wj.png)

----

### Contexte

- Déployer l’application Azure Voting App sur le cluster Kubernetes de Azure
- Déployer sa base de données Redis sur le cluster Kubernetes de Azure
- Permettre le scaling de l'application Azure Voting App

----

**Cluster AKS**

- 4 nodes
- 1 gateway
- Ingress
- Clé SSH

----

**Redis**

- ClusterIP
- Mot de passe
- Stockage permanent

----

**Voting App**

- ClusterIP
- Nom de domaine
- Certificat TLS
- Scaling : 2 à 8 instances (70% CPU)

---

## Déploiement

----

### Cluster

*Groupe de ressource / Nom / Nombre de noeuds / Ingress / Gateway / Subnet*

![](https://i.imgur.com/M7Piqz6.png)

----

### Base de données Redis

- Deployment et Service
- Passage de Load Balancer à Cluster
- Port 6379
- Env : Mot de passe sécurisé dans un Secret
- Stockage permanent : PersistentVolumeClaim et StorageClass

----

### Voting App

- Deployment et Service
- Passage de Load Balancer à Cluster
- Port 80
- Env : Cluster et mot de passe Redis et variable de stress
- Ajout du CPU
- Nom de domaine (Gandi)
- TLS (certificat dans un secret)
- Scaling jusqu'à 8 instances

---

## Architecture finale

![](https://i.imgur.com/PT9lgtI.png)

----

### Topologie

![](https://i.imgur.com/EVItc8H.png)

---

### Test de charge

![](https://i.imgur.com/Mr4rHxW.png)

----

## Difficultés rencontrées

- Pas d'indication en cas d'erreur : on peut chercher très longtemps
- Lier les différentes ressources entre elles
- Différencier ce qui est obligatoire de ce qui ne l'est pas
- Comprendre les différences entre labels/noms de metadata/noms de containers
