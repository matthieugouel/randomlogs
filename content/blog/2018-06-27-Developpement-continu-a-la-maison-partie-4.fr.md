---
title: Développement continu à la maison (partie 4)
description: Docker registry, SCM et CI/CD
date: "2018-06-27"
featured: "devops.jpg"
featuredalt: "devops"
featuredpath: "img/blog"
linktitle: ""
categories: ["DevOps"]
tags: ["Kubernetes"]
type: post
---

Après avoir installé notre cluster Kubernetes nous avons configuré un reverse proxy et tout le nécessaire pour déployer convenablement des applications. Aujourd’hui nous allons installer ce qu’il faut pour enfin faire du déploiement continu. On va enfin pouvoir justifier le titre de la série !

<!-- more -->

Notre stratégie d’intégration continue se doit d’être assez simple, du moins pour commencer. Exit Jenkins & Cie qui demandent beaucoup de ressources et une configuration non-négligeable. Nous on a juste besoin d’un logiciel de gestion de version (ou SCM pour Source Code Management) et d’un logiciel de déploiement continu qui s’enclenchera à chaque fois que l’on poussera du code sur notre SCM. Nous allons aussi avoir besoin d’un *registry* Docker pour stocker les images crée par la chaîne CI/CD et déployé dans notre cluster.

Nous en avons déjà parlé, j’ai choisi *Gogs* comme SCM car il est simple et léger. Pour l’intégration continue j’ai choisi *Drone* qui est également très simple à mettre en place et ne demande pas beaucoup de ressources (rappelez-vous mon cluster est loin d’être une flèche). La mission d’aujourd’hui : installer et faire fonctionner ces deux outils ensemble !

### Installation du docker registry

Cette partie est vraiment très simple. Docker fourni une image Docker registry déjà toute prête à utiliser.

Le stockage ici est fait via un *persistant volume* qui pointe sur un dossier local du node. Ce n’est PAS l’idéal du tout : si le node tombe bye-bye les images. Mais bon dans un premier temps c’est plus simple que monter un serveur NFS. Vous verrez que c’est aussi cette méthode qui est utilisée pour Gogs (ce qui est encore plus dangereux !)

Voici le [manifest](https://github.com/matthieugouel/kubernetes-manifests/blob/master/registry/registry.yml) qui me sert à déployer le Docker registry sur ma configuration. Comme d’habitude à adapter selon vos besoins.

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: registry
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: registry-volume
  namespace: registry
  labels:
    type: local
    app: registry
spec:
  capacity:
    storage: 5Gi
  accessModes:
  - ReadWriteOnce
  hostPath:
    path: /tmp/data/registry-volume
  persistentVolumeReclaimPolicy: Delete
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: registry-claim
  namespace: registry
  labels:
    app: registry
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: registry
  namespace: registry
  labels:
    app: registry
spec:
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: registry
    spec:
      containers:
      - image: registry:2
        name: registry
        env:
        - name: REGISTRY_HTTP_ADDR
          value: :5000
        - name: REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY
          value: /var/lib/registry
        ports:
        - containerPort: 5000
          name: registry
          protocol: TCP
        volumeMounts:
        - name: registry
          mountPath: /var/lib/registry
      volumes:
      - name: registry
        persistentVolumeClaim:
          claimName: registry-claim
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: registry
  name: registry
  namespace: registry
spec:
  ports:
  - name: registry
    port: 5000
    targetPort: 5000
    protocol: TCP
  selector:
    app: registry
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: registry
  namespace: registry
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  tls:
  - secretName: home
  rules:
  - host: registry.home.local
    http:
      paths:
      - path: /
        backend:
          serviceName: registry
          servicePort: 5000
```

Pour l’installation, toujours pareil :

```bash
kubectl apply -f https://raw.githubusercontent.com/MatthieuGouel/k8s-manifests/master/registry/registry.yml
```

Une fois que le pod est en état *running*  (en faisant un `kubectl get pods -n registry` pour vérifier) on peut faire un petit test d’upload d’une image manuellement sur le Docker registry.

```bash
docker pull hello-world
docker tag hello-world registry.home.local/hello-world
docker push registry.home.local/hello-world
```
```bash
The push refers to repository [registry.home.local/hello-world]
2b8cbd0846c5: Pushed
latest: digest: sha256:d5c74e6f8efc7bdf42a5e22bd764400692cf82360d86b8c587a7584b03f51520 size: 524
```

Parfait on a maintenant un endroit pour stocker les images qui résulteront de notre chaîne CI/CD. On passe au SCM maintenant ?!

### Installation de Gogs

Pour la suite je vais utiliser une façon alternative et plus pratique pour installer des programmes dans notre cluster. Les manifests c’est sympa mais la communauté s’est vite rendu compte que c’était un peu lourd. Finalement  d’un utilisateur à l’autre la structure d’un manifest est toujours la même, seul les variables changent. C’est ainsi que **Helm** est né. Ce soft prend en entrée des templates d’installation (charts) pour chaque programme à installer et prend également un yaml de variables correspondant à ce template (pour ceux qui connaissent Jinja2 c’est exactement le même principe).

Ce qui est bien c’est qu’il existe déjà moulte [charts](https://github.com/kubernetes/charts) qui attentent juste d’être utilisé avec vos propres variables … cool non ? :-)

Pour l’installation de Gogs et Drone, donc, je vais vous donner l’exemple du yaml de variables que j’ai utilisé ainsi que la commande qui utilise le chart associé à ces variables.  Ce qui est bien en plus c’est que l'on a pas besoin non plus d'inventer la structure yaml car la communauté qui fait le chart fourni étalement un exemple de yaml avec des valeurs par défaut ! Pratique du coup si ça se trouve vous n’aurez même pas à le changer (n’y croyez pas trop quand même) !

Pour installer Helm, allez donc faire un tour sur le [site officiel](https://docs.helm.sh/using_helm/#quickstart-guide) ou sur leur [repository Github](https://github.com/kubernetes/helm). Voilà. Ah oui n'oubliez pas d'ajouter le repertoire "incubator" de helm une fois l'installation terminée avec la commande :

```bash
helm repo add incubator https://kubernetes-charts-incubator.storage.googleapis.com/
```

Pour Gogs on va utiliser le chart officiel `incubator` disponible [ici](https://github.com/kubernetes/charts/tree/master/incubator/gogs). Vous n’avez pas besoin de  cloner quoi que ce soit Helm va le faire pour vous. Par contre vous allez éventuellement avoir besoin du fichier `values.yaml` qui est là où on va mettre nos variables custom.

Pour ma part mon fichier `values.yaml` ressemble à ça :

```yaml
## Override the name of the Chart.
##
# nameOverride:

## Kubernetes configuration
## For minikube, set this to NodePort, elsewhere use LoadBalancer
##
serviceType: NodePort

replicaCount: 1

image: gogs/gogs
imageTag: latest
imagePullPolicy: IfNotPresent

service:
  ## Override the components name (defaults to service).
  ##
  # nameOverride:

  ## HTTP listen port.
  ## ref: https://gogs.io/docs/advanced/configuration_cheat_sheet
  ##
  httpPort: 80

  ## SSH listen port.
  ## ref: https://gogs.io/docs/advanced/configuration_cheat_sheet
  ##
  sshPort: 2222

  ## Gogs configuration values
  ## ref: https://gogs.io/docs/advanced/configuration_cheat_sheet
  ##
  gogs:

    ## Application name, can be your company or team name.
    ##
    appName: 'Gogs'

    ## Either "dev", "prod" or "test".
    ##
    runMode: 'prod'

    ## Force every new repository to be private.
    ##
    forcePrivate: false

    ## Indicates whether or not to disable Git clone through HTTP/HTTPS. When
    ## disabled, users can only perform Git operations via SSH.
    ##
    disableHttpGit: false

    ## Indicates whether or not to enable repository file upload feature.
    ##
    repositoryUploadEnabled: true

    ## File types that are allowed to be uploaded, e.g. image/jpeg|image/png.
    ## Leave empty means allow any file typ
    ##
    repositoryUploadAllowedTypes:

    ## Maximum size of each file in MB.
    ##
    repositoryUploadMaxFileSize: 3

    ## Maximum number of files per upload.
    ##
    repositoryUploadMaxFiles: 10

    ## Enable this to use captcha validation for registration.
    ##
    serviceEnableCaptcha: true

    ## Users need to confirm e-mail for registration
    ##
    serviceRegisterEmailConfirm: false

    ## Weather or not to allow users to register.
    ##
    serviceDisableRegistration: false

    ## Weather or not sign in is required to view anything.
    ##
    serviceRequireSignInView: false

    ## Mail notification
    ##
    serviceEnableNotifyMail: false

    ## Either "memory", "redis", or "memcache", default is "memory"
    ##
    cacheAdapter: memory

    ## For "memory" only, GC interval in seconds, default is 60
    ##
    cacheInterval: 60

    ## For "redis" and "memcache", connection host address
    ## redis: network=tcp,addr=:6379,password=macaron,db=0,pool_size=100,idle_timeout=180
    ## memcache: `127.0.0.1:11211`
    ##
    cacheHost:

    ## Enable this to use captcha validation for registration.
    ##
    serverDomain: git.home.local

    ## Full public URL of Gogs server.
    ##
    serverRootUrl: https://git.home.local/

    ## Landing page for non-logged users, can be "home" or "explore"
    ##
    serverLandingPage: home

    ## Either "mysql", "postgres" or "sqlite3", you can connect to TiDB with
    ## MySQL protocol.  Default is to use the postgresql configuration included
    ## with this chart.
    ##
    databaseType: postgres

    ## Database host.  Unused unless `postgresql.install` is false.
    ##
    databaseHost:

    ## Database user.  Unused unless `postgresql.install` is false.
    ##
    databaseUser:

    ## Database password.  Unused unless `postgresql.install` is false.
    ##
    databasePassword:

    ## Database password.  Unused unless `postgresql.install` is false.
    ##
    databaseName:

    ## Hook task queue length, increase if webhook shooting starts hanging
    ##
    webhookQueueLength: 1000

    ## Deliver timeout in seconds
    ##
    webhookDeliverTimeout: 5

    ## Allow insecure certification
    ##
    webhookSkipTlsVerify: true

    ## Number of history information in each page
    ##
    webhookPagingNum: 10

    ## Can be "console" and "file", default is "console"
    ## Use comma to separate multiple modes, e.g. "console, file"
    ##
    logMode: console

    ## Either "Trace", "Info", "Warn", "Error", "Fatal", default is "Trace"
    ##
    logLevel: Trace

    ## Undocumented, but you can take a guess.
    ##
    otherShowFooterBranding: false

    ## Show version information about Gogs and Go in the footer
    ##
    otherShowFooterVersion: true

    ## Show time of template execution in the footer
    ##
    otherShowFooterTemplateLoadTime: true

    ## Change this value for your installation.
    ##
    securitySecretKey: "changeme"

    ## Number of repositories that are showed in one explore page
    ##
    uiExplorePagingNum: 20

    ## Number of issues that are showed in one page
    ##
    uiIssuePagingNum: 10

    ## Number of maximum commits showed in one activity feed.
    ## NOTE: This value is also used in how many commits a webhook will send.
    ##
    uiFeedMaxCommitNum: 5

  ## Ingress configuration.
  ## ref: https://kubernetes.io/docs/user-guide/ingress/
  ##
  ingress:
    ## Enable Ingress.
    ##
    enabled: true

    ## Annotations.
    ##
    annotations:
      kubernetes.io/ingress.class: traefik
    #   kubernetes.io/tls-acme: 'true'

    ## Hostnames.
    ## Must be provided if Ingress is enabled.
    ##
    hosts:
      - git.home.local

    ## TLS configuration.
    ## Secrets must be manually created in the namespace.
    ##
    tls:
      - secretName: home
        hosts:
          - git.home.local


## Persistent Volume Storage configuration.
## ref: https://kubernetes.io/docs/user-guide/persistent-volumes
##
persistence:
  ## Enable persistence using Persistent Volume Claims.
  ##
  enabled: true

  ## gogs data Persistent Volume Storage Class
  ## If defined, storageClassName: <storageClass>
  ## If set to "-", storageClassName: "", which disables dynamic provisioning
  ## If undefined (the default) or set to null, no storageClassName spec is
  ##   set, choosing the default provisioner.  (gp2 on AWS, standard on
  ##   GKE, AWS & OpenStack)
  ##
  # storageClass: "-"

  ## Persistent Volume Access Mode.
  ##
  accessMode: ReadWriteOnce

  ## Persistent Volume Storage Size.
  ##
  size: 1Gi

## Configuration values for the postgresql dependency.
## ref: https://github.com/kubernetes/charts/blob/master/stable/postgresql/README.md
##
postgresql:

  ### Install PostgreSQL dependency
  ##
  install: true

  ### PostgreSQL User to create.
  ##
  postgresUser: gogs

  ## PostgreSQL Password for the new user.
  ## If not set, a random 10 characters password will be used.
  ##
  postgresPassword: gogs

  ## PostgreSQL Database to create.
  ##
  postgresDatabase: gogs

  ## Persistent Volume Storage configuration.
  ## ref: https://kubernetes.io/docs/user-guide/persistent-volumes
  ##
  persistence:
    ## Enable PostgreSQL persistence using Persistent Volume Claims.
    ##
    enabled: true

    accessMode: ReadWriteOnce

    size: 5Gi
```

Vient le moment de l’installation. Pour le stockage comme je vous l’ai dit j’utilise un *persistant volume* local, il faut donc commencer par le créer. On peut le faire en poussant ce genre manifest :

```yaml
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: git-gogs-volume
  labels:
    type: local
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  hostPath:
    path: /data/git
  persistentVolumeReclaimPolicy: Delete
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: git-postgres-volume
  labels:
    type: local
spec:
  capacity:
    storage: 5Gi
  accessModes:
  - ReadWriteOnce
  hostPath:
    path: /data/git
  persistentVolumeReclaimPolicy: Delete
```

Vous pouvez changer la taille des espaces de stockage à votre convenance (moi j'ai toujours un "hyperviseur" avec une configuration matérielle ridicule).

Pour appliquer ce manifest, on fait classiquement :

```bash
kubectl apply -f peristantvolume.yml
```

On peut vérifier que tout s’est bien passé avec la commande (le status doit être `Available` pour l’instant ) :

```bash
kubectl get pv
```

```bash
NAME                  CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                     STORAGECLASS   REASON    AGE
git-gogs-volume       1Gi        RWO            Delete           Available                                                      13s
git-postgres-volume   5Gi        RWO            Delete           Available
```

Très bien on a notre persistant volume et on a près de nous (dans le dossier courant) notre fichier `values.yaml` avec nos paramètres favoris. C’est le moment de lancer le Helm !

```bash
helm install -f values.yaml  incubator/gogs --namespace git -n git-gogs
```

Cette commande utilise donc le fichier `values.yaml`, le repository `incubator/gogs` et installe tout dans le namespace `git`. Enfin le nom de l’application sera `git-gogs`.

Le résultat de la commande Helm doit vous faire un petit retour sur ce qu’il a fait. Notez que le status peut ne pas être tout de suite en `Running` car l’image doit être téléchargé et initialisé.

En théorie quelques minutes plus tard vous devriez pouvoir vous connecter sur l’interface de Gogs sur le FQDN que vous avez précisé dans le fichier `values.yaml`. Et de un.

### Installation de Drone

Pour l’installation de Drone c’est sensiblement la même chose. Vous renseignez votre fichier `values.yaml` et l’avantage c’est qu’il n’y a pas besoin de *persistant volume*.  Encore plus simple.

Voici donc mon fichier `values.yaml` :

```yaml
image:
  registry: docker.io
  org: drone
  server: drone
  agent: drone
  tag: 0.7
  pullPolicy: IfNotPresent

service:
  http:
    externalPort: 80
    internalPort: 8000
  type: ClusterIP

ingress:
  enabled: true
  # enable TLS via kube-lego
  tls: true
  hostname: drone.home.local
  secretName: home
  annotations:
    kubernetes.io/ingress.class: traefik

server:
  # Drone server configuration. Values in here get injected as environment variables.
  # See http://readme.drone.io/admin/installation-reference#server-options for a list of possible values.
  env:
    DRONE_DEBUG: "false"
    DRONE_DATABASE_DRIVER: "sqlite3"
    DRONE_DATABASE_DATASOURCE: "drone.sqlite"
    # Drone requires some environment variables to bootstrap the git service or it won't start up.
    # Uncomment this and add your own custom configuration.
    #
    # See http://readme.drone.io/admin/installation-reference/ for more info on these envvars.
    DRONE_PROVIDER: "gogs"
    DRONE_OPEN: "true"
    DRONE_GOGS: true
    DRONE_GOGS_URL: https://git.home.local
    DRONE_GOGS_SKIP_VERIFY: true
    DRONE_ADMIN: "matthieu"
  resources:
    requests:
      memory: 32Mi
      cpu: 40m
    limits:
      memory: 1Gi
      cpu: 1

agent:
  # Drone agent configuration. Values in here get injected as environment variables.
  # See http://readme.drone.io/admin/installation-reference#server-options for a list of possible values.
  env:
    DRONE_DEBUG: "false"
  resources:
    requests:
      memory: 32Mi
      cpu: 40m
    limits:
      memory: 1Gi
      cpu: 1

# Uncomment this if you want to set a specific shared secret between
# the agents and servers, otherwise this will be auto-generated.
#shared_secret: supersecret
```

Pour l’installer j’utilise la commande :

```bash
helm install -f values.yaml incubator/drone --namespace git -n git-drone
```

Et de deux.

### Configuration

Dernière étape, la configuration de Gogs et de Drone ensemble. En fait une bonne partie de la configuration a été faite durant l’installation de Drone. Il ne reste plus qu’à se logger sur Drone avec vos credentials de Gogs (Drone ne supporte pas encore Oauth2)  et le tour est joué. Enfin, vous devriez voir vos répositories Git dans l’interface de Drone.

### Conclusion

Enfin ! On a tout pour créer nos chaines CI/CD. Pfiou on va pas se mentir c’est pas mal de boulot pour arriver jusqu’ici mais tout de même on en a fait du chemin ! Désormais on va pouvoir vraiment facilement déployer de façon continue nos applications sur Kubernetes, de façon simple, scalable, et surtout automatique.
Malheureusement l’application pratique du déploiement sera pour un autre article ! Stay tuned.

Les articles de la série :

* [Développement continu à la maison (partie 1)](https://matthieugouel.github.io/blog/2018-05-14-developpement-continu-a-la-maison-partie-1/)
* [Développement continu à la maison (partie 2)](https://matthieugouel.github.io/blog/2018-05-21-developpement-continu-a-la-maison-partie-2/)
* [Développement continu à la maison (partie 3)](https://matthieugouel.github.io/blog/2018-06-04-developpement-continu-a-la-maison-partie-3/)
* [Développement continu à la maison (partie 4)](https://matthieugouel.github.io/blog/2018-06-27-developpement-continu-a-la-maison-partie-4/)
