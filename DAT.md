# Document d’Architecture Technique

## Contexte

Projet : Déploiement d'une application de vote et de sa base de données Redis

### Besoins fonctionnels

L'application de vote permet aux utilisateurs lambdas, connaissant le DNS ou l'adresse IP, de voter pour deux options (Chiens/Chats), ou de remettre le compte à zéro.

![](https://i.imgur.com/PT9lgtI.png)

### Contraintes techniques

L'application doit être résiliente : dans le cas où un grand nombre d'utilisateurs se connectent en même temps à l'application, elle doit rester fonctionnelle.
L'accès à la base de données REDIS nécessite un mot de passe, stocké dans un secret Kubernetes.
Le site doit être en HTTPS : un certificat TLS, fourni par Gandi, est nécessaire. Il est également stocké dans un secret Kubernetes.
Les VMs créées nécessitent une clé SSH afin d'y accéder.

## Topologie

![](https://i.imgur.com/A6Xb4M6.png)

### Représentation opérationnelle

Les données seront sauvegardées sur un disque externe relié à la base de données REDIS. Afin d'accéder à cette base de données, un mot de passe sera nécessaire.

### Représentation fonctionnelle

L'utilisateur se connectera via le Web en utilisant un nom de domaine authentifié par un certificat TLS (https://vote.simplon-celia.space), traduit par le DNS en adresse IP, afin d'arriver à la Gateway qui le conduira à l'application de vote.

### Représentation technique

- L'image utilisée est la dernière image REDIS. Cela est à changer hors phase de test. REDIS sera déployé et défini en tant que Cluster. Le port du container REDIS sera 6379.
- L'image de l'application de vote sera trouvée à : whujin11e/public:azure_voting_app. L'application sera déployée et définie en tant que Cluster. Le port du container de l'application sera 80.
- L'Ingress sera configuré avec Azure Application Gateway en Ingress controller, le certificat TLS en Secret et le cluster de l'application de vote en backend.
- Le PersistentVolumeClaim et le StorageClass définiront un stockage en autorisation Read/Write, qui peut s'étendre durant un auto scaling.
- L'autoscaler permet un scale-in et scale-out de l'application de vote automatique.

## Choix de l’architecture

- Afin de tester la résilience de l'infrastructure, une variable d'environnement STRESS_SECS va permettre de stresser l'application et un test de charge va envoyer, à l'aide d'un script bash et d'un script R, un grand nombre de requête.
- Le CPU de l'application aura a disposition 250 millicpu, avec un maximum de 500 millicpu.
- Le pod de l'application de vote aura 2 réplicas.
- Le stockage, de type disque, Standard LRS, fera 1Gi.
- Jusqu'à 8 VMs peuvent être déployées durant un scale-out lorsqu'on atteint 70% du CPU.
