
<h1>Mini-projet</h1>
<h2>Déployez wordpress à l’aide des étapes suivantes</h2>

<h2>Deploiement de MYSQL</h2>
Créez un deployment mysql avec 1 réplica.

Le manifeste ci dessous créé un déploiement avec une stratégie Recreate car de mon point de vue, travailler sur une base de données, c'est préférable de toute éteindre et tout rallumer pour éviter des écarts

Ensuite je défini mon selector, le pod que j'installe (le template) et je m'assure bien de monter le répertoire contenant la base de donnée.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-clusterip
spec:
  type: ClusterIP
  selector:
    app: mysql
  ports:
    - port: 3306
ubuntu@minikube-daniel:~/miniProjet$ cat mysqlDeploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dp-mysql
  labels:
    env: backend
spec:
  strategy:
    type:  Recreate
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      name: mysql-backend
      labels:
        app: mysql
        type: pod
        formateur: Frazer
    spec:
      containers:
        - image: mysql:5.7
          name: mysql-backend
          ports:
            - containerPort: 3306
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: userdb
                  key: password
            - name: MYSQL_USER
              valueFrom:
                secretKeyRef:
                  name: userdb
                  key: password
            - name: MYSQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: secret
                  key: password
            - name: MYSQL_DATABASE
              value: wordpress
          volumeMounts:
            - mountPath: /var/lib/mysql
#Nous voulons sauvegarder la base de donnée de mysql, il faut aller sur la documentation pour savoir ou se trouve la bdd
              name: mysql-data
#Ce volume doit exister sur le POD et il est donc impérative de le faire dans la partie volume ci dessous
      volumes:
        - name: mysql-data
          persistentVolumeClaim:
            claimName: mysqlpvc

```

Pour appliquer le manifeste il faudra le sauvegarder dans un fichier par exemple mysqlDeploy.yaml et lancer la commande !

```shell
#Lancer la création
kubectl apply -f wpManifest.yaml

#Verification du deployement
kubectl get deploy -o wide

#Verification des pods
kubectl get pods -o wide
```

On constate que la création est en PENDING, et j'ai ma petite idée derrière ça, nous allons y revenir plutard dans le projet.
Si vous êtes curieux vous pourrez faire un describe sur un pods pour voir dans les Events pourquoi il est en pending!
Vous constaterez qu'en réalité le Persistent Volume Claim n'existe pas encore et c'est donc du au manque de cette ressource que notre Deployement restera en PENDING ! 

![[Pasted image 20211221114923.png]]

Pour le moment nous allons dérouler le projet, jusqu'au moment où toutes les ressources seront créées pour relancer les manifests de déploiements

Créez un service de type clusterIP pour exposer votre pod mysql, j'ai choisi de ne pas attribuer d'IP à ce service car je trouve que cela complique les choses.
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-clusterip
spec:
  type: ClusterIP
  selector:
    app: mysql
  ports:
    - protocol: TCP
      port: 3306
      targetPort: 3306
  clusterIP: None
```
On peut vérifier que le service est bien créer avec la commande suivante :
```shell
kubectl get svc -o wide
```
![[Pasted image 20211221115248.png]]


<h2>Deploiement de Wordpress</h2>
Créez un deployment wordpress avec les bonnes variables d’environnement pour se connecter à la base de données mysql.

J'ai ajouté des secrets, ce sont les identifiants de connexion de root, depuis que l'utilisateur que je génère précédemment n'a pas assez de droit... et j'ai ajouté pour des raisons de debug la variable WORDPRESS_DEBUG
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dp-wp
  labels:
    env: frontend
spec:
  strategy:
    type:  Recreate
  replicas: 1
  selector:
    matchLabels:
      app: wp
  template:
    metadata:
      name: wp-frontend
      labels:
        app: wp
        type: pod
        formateur: Frazer
    spec:
      containers:
        - image: wordpress:latest
          name: wp-frontend
          ports:
            - containerPort: 80
          env:
            - name: WORDPRESS_DB_HOST
              value: mysql-clusterip
            - name: WORDPRESS_DB_USER
              valueFrom:
                secretKeyRef:
                  name: secret
                  key: user
            - name: WORDPRESS_DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: secret
                  key: password
            - name: WORDPRESS_DB_NAME
              value: wordpress
            - name: WORDPRESS_DEBUG
              value: "1"
          volumeMounts:
            - mountPath: /var/www/html
#Nous voulons sauvegarder notre frontend, il faut aller sur la documentation pour savoir ou se trouve la bdd
              name: wp-data
#Ce volume doit exister sur le POD et il est donc impérative de le faire dans la partie volume ci dessous
      volumes:
        - name: wp-data
          persistentVolumeClaim:
            claimName: wordpresspvc
```


<h2>Création de volume</h2>
Votre deployment devra stocker les données de wordpress et de mysql sur un volume mounté dans le (wordpress : /data) et (mysql : /db data) de votre noeud

On créé les volumes persistent qui contiendront la base de donnée mysql puis les données du frontend. Pourquoi fait on cela ? Tout simplement pour avoir plusieurs replica et 1 donnée partagée entre ces réplica. Si jamais un des replicas tombent, les données sont toujours accessible quand le container sera reconstruit par le deploy.

Manifeste pour la création du volume persistent pour sauvegarder les données de wordpress :

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: wordpresspv
spec:
  storageClassName: wordpressPv
  accessModes:
    - ReadWriteOnce
  capacity: 
    storage: 1Gi 
  hostPath:
    path: /data
```

Manifeste pour la création du PVC afin que nos pods wordpress utilisent notre volume persistent créé précédemment : 
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wordpressPvc
spec:
  storageClassName: wordpresspv
  accessModes:
    - ReadWriteOnce
  resources: 
    requests: 
	  storage: 1Gi
	
```

Manifeste pour la création du volume persistent pour sauvegarder notre base de donnée mysql :
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysqlPv
spec:
  storageClassName: mysqlpv 
  accessModes:
    - ReadWriteOnce 
  capacity: 
    storage: 1Gi 
  hostPath:
    path: /db-data
```

Manifeste pour la création du PVC afin que nos pods mysql utilisent notre volume persistent créé précédemment : 
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysqlpvc
spec:
  storageClassName: mysqlpv
  accessModes:
    - ReadWriteOnce
  resources: 
    requests: 
	  storage: 1Gi
	
```

Lors de la création de ces manifestes j'ai rencontré plusieurs souci que j'ai pu troubleshooté:

Le nom des PV et PVC ne doivent pas contenir des majuscules mais uniquement les critères suivants :

Lowercase RFC 1123 subdomain must consist of lower case alphanumeric characters, '-' or '.', and must start and end with an alphanumeric character (e.g. 'example.com', regex used for validation is '[a-z0-9]([-a-z0-9]*[a-z0-9])?(\.[a-z0-9]([-a-z0-9]*[a-z0-9])?)*')

Ensuite nous pouvons vérifier avec les commandes suivantes que nos PV et PVC soient bien là :
```shell
kubectl get pvc -o wide
kubectl get pv -o wide
```
![[Pasted image 20211221114851.png]]


A partir d'ici nous pouvons supprimer nos précédents deploy car les volumes n'étaient pas encorer créer et relancer leur création ! Et voila tout marche beaucoup mieux
![[Pasted image 20211221120337.png]]

<h2>Creation d'un service NodePort</h2>
Créez un service de type nodeport pour exposer le frontend wordpress

Dans ce manifeste nous allons exposé le port 30080 à l'extérieur pour que nos pods ayant le labbel app: wp soit accessible sur leur port 80.
```yaml
apiVersion: v1
kind: Service
metadata:
  name: wp-nodeport
spec:
  type: NodePort
  selector:
    app: wp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30080

```

Une fois le manifeste créer, on le lance et on vérifie qu'il existe à l'aide des commandes suivantes :
```shell
kubectl apply -f wpNIP.yaml
kubectl get svc -o wide
kubectl describe svc wp-nodeport
```
![[Pasted image 20211221121844.png]]

<h2>Troubleshooting</h2>
Bon petit problème rien ne fonctionne ! Troubleshooting time :

![[Pasted image 20211221135523.png]]

ubuntu@minikube-daniel:~/miniProjet$ kubectl create secret generic dbpassword  --from-literal=MYSQL_ROOT_PASSWORD=password
secret/dbpassword created
ubuntu@minikube-daniel:~/miniProjet$

![[Pasted image 20211221135537.png]]

Au début je n'utilisais pas le secret pour le mot de passe de la base de donnée, de plus la création de base de donnée n'est pas possible via un déploiement kubernetes avec des variables d'environnement, il faudra scripter et utiliser le LifeCycle management pour lancer un script de création au démarrage des machines.

Bon apres 1 heure de debuggage j'ai enfin trouvé mon erreur...

Cela fonctionne très bien maintenant !

![[Pasted image 20211221151659.png]]

J'ai ajouté un variable au déploiement de WORDPRESS pour avoir plus de verbosité dans l'erreur.

WORDPRESS_DEBUG = "1"

Pendant la phase de débug aussi, pensez à vider et supprimer les répertoires créer sur la machine quand on lance les déploiements, sinon les modifications ne sont pas pris en compte et on se retrouve avec les mêmes erreurs à chaque fois.

```shell

#Clean Deploy+Volume
kubectl delete -f mysqlDeploy.yaml
kubectl delete -f wpDeploy.yaml
sudo rm -Rf /data
sudo rm -Rf /db-data

#Relancer
kubectl apply -f mysqlDeploy.yaml
kubectl apply -f wpDeploy.yaml

ou

kubectl apply -f .
```

<h2>Résultat final</h2>

Vous remarquerez à gauche mon site vu public, et a droite le tableau de bord de l'admin.
![[Pasted image 20211221153938.png]]


<h2>BONUS - Ingress</h2>


Nous vous conseillons d’utiliser les manifests pour réaliser cet exercice

Rédigez un rapport retraçant l’ensemble des taches que vous avez réalisez (captures
+ explications) et à la fin de votre travail, poussez vos manifests sur github et ajoutez
le lien de votre repo à votre rapport et sauvegarder ledit rapport dans l’intranet
