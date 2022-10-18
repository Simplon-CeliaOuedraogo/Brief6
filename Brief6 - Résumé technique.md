# **Brief 6**

*Dans WSL Ubuntu*

## Sommaire

- [Brief 6 - part 1 : Kubernetes](#B6-1)
    - [Chapitre 1 : Déployer un cluster AKS](#C1)
    - [Chapitre 2 : Déployer un container Redis](#C2)
    - [Chapitre 3 : Déployer un container Voting App](#C3)
    - [Chapitre 4 : Un mot de passe pour Redis](#C4)
    - [Chapitre 5 : Configurer un stockage persistent pour Redis](#C5)
- [Brief 6 - part 2 : Kubernetes scaling](#B6-2)
    - [Chapitre 6 : Utiliser Azure Application Gateway avec AKS](#C6)
    - [Chapitre 7 : Un nom de domaine pour Voting App](#C7)
    - [Chapitre 8 : Un certificat TLS pour Voting App](#C8)
    - [Chapitre 9 : Scaling de la Voting App](#C9)
- [Fonctionnement de Kubernetes](#Kube)

<div id=B6-1>
     
# Brief 6 - part 1 : Kubernetes

<div id=C1>
    
## Chapitre 1 : Déployer un cluster AKS
    
### Créer un cluster AKS avec 2 nodes

Connexion azure CLI
``az login``

Création d'un groupe de ressource
``az group create --name B6Celia --location eastus``

Création du cluster AKS avec 2 nodes et la clé SSH de l'utilisateur dans le groupe de ressource
``az aks create -g B6Celia -n AKSCluster --node-count 2 --ssh-key-value .\.ssh\id_rsa.pub``

### Connexion au cluster

Téléchargement de la dernière version stable de kubectl binary
``curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"``

Vérification de l'intégrité des données téléchargées à l'aide d'un checksum 
- Téléchargement du checksum
``curl -LO "https://dl.k8s.io/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"``
- Vérification
``echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check``

Installation de kubectl
``sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl``

Test pour vérifier que la version est à jour
``kubectl version --client``

Téléchargement des informations d’identification et configuration de l’interface de ligne de commande Kubernetes
``az aks get-credentials --resource-group B6Celia --name AKSCluster``

Vérification de la connexion au cluster
``kubectl get nodes``
    
<div id=C2>

## Chapitre 2 : Déployer un container Redis

### Création d'un Deployment 

Deployment nommé "redis" avec la dernière image Redis, 1 réplica, en utilisant le port 6379, qui est le port par défaut du serveur Redis. Le label de l'application est "redislabel". Le conteneur est nommé "redis".

```consol
apiVersion: apps/v1
kind: Deployment
metadata:
  name : redis
  labels:
  app: redislabel
spec:
  selector:
    matchLabels:
      app: redislabel
  replicas: 1
  template:
    metadata:
      labels:
        app: redislabel
    spec:
      containers:
      - name: redis
        image: redis:latest
        ports:
        - containerPort: 6379
```

### Création d'un Service de type LoadBalancer pour Redis 

Service nommé "loadredis", en indiquant le label du déployment concerné et le port.

```control
apiVersion: v1
kind: Service
metadata:
  name : loadredis
spec:
  type: LoadBalancer
  selector:
    app: redislabel
  ports:
  - port: 6379
```

On lance le déploiement pour tester la connexion au container Redis déployé : ``kubectl apply -f redis.yaml``

### Installation d'un client Redis sur le poste de travail

Ajout du repository redis à "apt", mise à jour et installation

```control
curl -fsSL https://packages.redis.io/gpg | sudo gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] https://packages.redis.io/deb $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/redis.list
sudo apt-get update
sudo apt-get install redis
```

Ouverture de redis sur une autre session WSL (parce que Windows) : ``redis-server``

Utilisation de redis cli pour vérifier que redis fonctionne : ``redis-cli ping``

### Test de la connection au container Redis déployé à l'aide de l'adresse IP externe

``kubectl get svc`` (pour obtenir l'adresse IP externe)
``redis-cli -h ExternalIPAdress``

<div id=C3>
                                      
## Chapitre 3 : Déployer un container Voting App

### Création d'un Deployment 

Deployment nommé "vote" avec une image de l’application Azure Voting App qui :
- dépend de Redis
- comprend la variable d’environnement STRESS_SECS à 2
- comprend un Service de type LoadBalancer

1 réplica, le label du deployment est "votelabel", le port est 80. Dans les variables d'environnement, on retrouve stress_secs et le cluster pour redis. Le conteneur est nomé "vote".
Pour le LoadBalancer, le service est nommé "loadvote" et utilise le port 80. Le label du déployment concerné est indiqué.

```control
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vote
  labels:
    app: votelabel
spec:
  replicas: 1
  selector:
    matchLabels:
      app: votelabel
  template:
    metadata:
      labels:
        app: votelabel
    spec:
      containers:
      - name: vote
        image: whujin11e/public:azure_voting_app
        ports:
        - containerPort: 80
        env:
        - name: REDIS
          value: "rediscluster"
        - name: STRESS_SECS
          value: "2"
---
apiVersion: v1
kind: Service
metadata:
  name: loadvote
spec:
  type: LoadBalancer
  ports:
  - port: 80
  selector:
    app: votelabel
```

### Modification du type du Service de Redis pour utiliser ClusterIP

Service nommé "rediscluster".

```control
apiVersion: v1
kind: Service
metadata:
  name : rediscluster
spec:
  type: ClusterIP
  selector:
    app: redislabel
  ports:
  - port: 6379
```

### Test du fonctionnement de l'application déployée à l'aide de l'adresse IP externe

![](https://i.imgur.com/FYSHGsq.png)
    
<div id=C4>

## Chapitre 4 : Un mot de passe pour Redis

*A partir d'ici, pour les scripts déjà présentés précédemment, seules les modifications des scripts apparaitront, pas les scripts en entier*

### Configuration du container Redis pour authentifier les clients avec un mot de passe

- Création d'un mot de passe codé en base64

```control
PASSWORD=simplonbrief6
echo -n ${PASSWORD} | base64
c2ltcGxvbmJyaWVmNg==
```

- Ajout de l'argument --requirepass dans le déployment de Redis

```control
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: redis
        args: ["--requirepass", "$(REDIS_PASS)"]
```

### Mot de passe sécurisé dans un Kubernetes Secret

- Kubernetes Secret : mot de passe codé en base64 plus haut

```control
apiVersion: v1
kind: Secret
metadata:
  name: redispw
type: Opaque
data:
  password: c2ltcGxvbmJyaWVmNg==
```

- Variables d'environnement pour le container Redis : ALLOW_EMPTY_PASSWORD indique qu'il faut un mot de passe, et REDIS_PASS indique où trouver le mot de passe

```control
      containers:
      - name: redis
        env:
        - name: ALLOW_EMPTY_PASSWORD
          value: "no"
        - name: REDIS_PASS
          valueFrom:
            secretKeyRef:
              name: redispw
              key: password
```

### Configuration du container Voting App pour utiliser ce mot de passe avec la variable d’environnement REDIS_PWD 

REDIS_PWD indique où trouver le mot de passe

```control
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: vote
        env:
        - name: REDIS_PWD
          valueFrom:
            secretKeyRef:
              name: redispw
              key: password
```
    
<div id=C5>

## Chapitre 5 : Configurer un stockage persistent pour Redis

### Stockage volatile : compte remis à 0 après suppression

### Création d'un PersistentVolumeClaim pour le container Redis avec une StorageClass adéquate pour permettre le scale out de Redis

- Création d'un PersistentVolumeClaim nommé "pvclaim" pour lequel le node est en autorisation ReadWrite, avec 1G de stockage. Le Storage Class utilisé est indiqué.

```control
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
     name: pvclaim
spec:
     accessModes:
     -  ReadWriteOnce
     resources:
          requests:
            storage: 1Gi
     storageClassName: storageclass
```
     
- Création d'une Storage Class nommé "storageclass" qui peut s'agrandir (permet le scale out). Le sku est Standard_LRS.

```control
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: storageclass
provisioner: kubernetes.io/azure-disk
parameters:
  skuName: Standard_LRS
  location: eastus
allowVolumeExpansion: true
```

- Ajout du PersistentVolumeClaim dans **volumes** dans le déploiement de Redis, et le chemin du stokage dans le conteneur dans volumeMounts.

```control
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      volumes:
        - name: redis-storage
          persistentVolumeClaim:
            claimName: pvclaim
      containers:
      - name: redis
        volumeMounts:
           -    mountPath: “/votingapp”
                name: redis-storage
```
              
### Stockage permanent : le PersistentVolumeClaim et les votes restent quand on supprime les containers

<div id=B6-2>
    
# Brief 6 - part 2 : Kubernetes scaling

<div id=C6>
    
## Chapitre 6 : Utiliser Azure Application Gateway avec AKS

### Recréer un cluster AKS avec l’add-on AGIC et 4 nodes

```control
az aks create -g B6Celia -n AKSCluster --node-count 4 --ssh-key-value ./.ssh/id_rsa.pub --enable-managed-identity -a ingress-appgw --appgw-name B6Gateway --appgw-subnet-cidr "10.225.0.0/16"
az aks get-credentials -n AKSCluster -g B6Celia
```

### Créer un Ingress dans votre configuration Kubernetes pour l’Application Gateway

```consol
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: vote-ingress
  annotations:
    kubernetes.io/ingress.class: azure/application-gateway
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: votecluster
            port:
              number: 80
```
              
### Vérification de la création de l'ingress

``kubectl get ingress``

### Modification du Service Voting App en conséquence

Changement pour le container "vote" d'un LoadBalancer à un ClusterIP nommé "votecluster" après l'ajout de la gateway : 

```console
apiVersion: v1
kind: Service
metadata:
  name: votecluster
spec:
  type: ClusterIP
  ports:
  - port: 80
  selector:
    app: votelabel
```

<div id=C7>
    
## Chapitre 7 : Un nom de domaine pour Voting App

Modification de l'adresse IP référencée par l'enregistrement DNS créé dans le Brief 5.

![](https://i.imgur.com/GqeDgV0.png)

<div id=C8>
    
## Chapitre 8 : Un certificat TLS pour Voting App

### Création d'un certificat TLS pour la Voting App en utilisant Certbot et le challenge DNS

Fait dans le [Brief 5](https://github.com/Simplon-CeliaOuedraogo/Brief5) !

### Update du certificat de l’Application Gateway

- Codage du certificat et de la clé privée en base64

```console
sudo su -
cd /etc/letsencrypt/live/vote.simplon-celia.space
cat cert.pem | base64 -w0
cat privkey.pem | base64 -w0
```

- Création d'un Secret nommé "ingress-tls" pour ajouter le certificat et la clé

```console
apiVersion: v1
kind: Secret
metadata:
  name: ingress-tls
type: kubernetes.io/tls
data:
  tls.crt: LS0...
  tls.key: LS0...
```
  
- Dans Ingress, ajout du nom du Secret pour lier l'ingress et le TLS

```console
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: vote-ingress
spec:
  tls: 
    - secretName: ingress-tls
```
  
<div id=C9>
    
## Chapitre 9 : Scaling de la Voting App

### Configuration du nombre de replicas à 2 pour le container Voting App

- Dans le Deployment de l'application de vote

```console
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vote
  labels:
    app: votelabel
spec:
  replicas: 2
```

### Configuration d'une règle d’autoscaling à 70% de CPU, max 8 instances

Création d'un HorizontalPodScaler visant le déployment de l'application de vote

```console
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: vote-autoscaler
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: vote
  minReplicas: 1
  maxReplicas: 8
  targetCPUUtilizationPercentage: 70
```

Vérification du fonctionnement de l'autoscaler : 

``kubectl get hpa``

![](https://i.imgur.com/ui03HNd.png)

### Test d'une montée en charge

Test de montée en charge du [Brief 4](https://github.com/simplon-paul-lion/Brief4) réutilisé : script en bash construisant un tableau et appelant un script en R construisant un graphique de ce tableau pour le nombre de requêtes cumulées en fonction du temps (en secondes) par instance créée. Ici, 8 instances (le maximum possible) ont été créées.

![](https://i.imgur.com/Mr4rHxW.png)

<div id=Kube>
    
# Fonctionnement de Kubernetes

Kubernetes : outil open-source permettant d'automatiser le déploiement, la mise à l'échelle et la gestion des conteneurs, qui contiennent les ressources nécessaires pour faire tourner l’application. L'état des applications conteneurisés est défini par des fichiers de configuration constitués de fichiers JSON ou YAML, puis maintenu par Kubernetes (controller-manager).
Pour interagir avec le cluster, on peut utiliser la ligne de commande (kubectl) ou l'API (interface de programmation d'application).

Kubernetes aide donc à gérer facilement et efficacement des clusters (ensemble de nœuds qui permettent d'exécuter des applications conteneurisées). Les noeuds sont des serveurs physiques ou virtuels qui exécutent les applications. Un pod correspond à un groupe de conteneurs qui partagent des ressources et qui sont exécutés sur un seul noeud. Le pod est l'objet Kubernetes le plus petit et le plus simple. Tout cela est contrôlé par le plan de contrôle (contrôle les nœuds Kubernetes et assigne les tâches).

Pendant l'écriture des fichiers de configuration, le serveur API permet la configuration des objets (Deployment, Service, Volume, Secrets...).

![](https://i.imgur.com/YX5A9wj.png)
