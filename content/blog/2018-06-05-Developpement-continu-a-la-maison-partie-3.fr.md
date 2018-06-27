---
title: Développement continu à la maison (partie 3)
description: Un reverse proxy avec Traefik
date: "2018-06-05"
featured: "devops.jpg"
featuredalt: "devops"
featuredpath: "img/blog"
linktitle: ""
categories: ["DevOps"]
tags: ["Kubernetes"]
type: post
---

Si vous avez suivi la partie précédente vous avez désormais un cluster Kubernetes tout neuf prêt à accueillir toutes nos applications. C’est génial mais il manque quelque chose.

<!-- more -->

On l’a vu quand on a testé le cluster, l’application est accessible depuis le navigateur via l’adresse IP du **master** et un port aléatoire. Il faut avouer que ce n’est pas très pratique, nous on aimerait bien avoir un nom DNS associé à ce service pour pouvoir y accéder en HTTPS par exemple. De plus on voudrait profiter de la possibilité de multiplier les instances de l’application et ainsi répartir la charge entre ces plusieurs instances, tout en ayant qu’un seul point d’entrée.

Eh bien le but d’aujourd’hui est de faire exactement ça. Et pour cela nous allons avoir besoin de:

* un serveur DNS
* un certificat SSL pour signer nos connexions HTTPS
* un reverse proxy


### Le serveur DNS

Alors celui-là on en a déjà fait mention dans la partie précédente. Je vous disais que j’ai un serveur DNS à côté de mon cluster qui me permet d’atteindre les noeuds avec un nom (comme par exemple `master.home.local`, `worker1.home.local`, …). Je vous disais également que ce n’était pas très dur d’installer un serveur DNS chez soi et que j’en reparlerais une autre fois. Aujourd’hui je vais … vous dire exactement la même chose. Je ferais un article pour présenter tous les services d’infrastructure qui tournent sur mon réseau, mais ce sera pour une autre fois. Pour l’instant Il faudra donc vous débrouiller pour mettre en place un serveur DNS chez vous.

Quelques pistes tout de même :

* BIND est de loin le serveur DNS le plus utilisé sous Linux
* Peut-être que votre box internet peut également faire office de serveur DNS
* Si vous avez un NAS style Synology, ils permettent parfois de mettre en place un serveur DNS très simplement

En tout cas pour chaque nouveau service on voudra donc ajouter un nom DNS propre à celui-ci. Ce nom DNS pointera à chaque fois sur l’adresse IP du noeud **master** de Kubernetes.

### Le certificat SSL

Ici vous l’aurez compris on va utiliser [l'article précédent](https://matthieugouel.github.io/blog/2018-05-28-gerer-nos-certificats-ssl/) qui explique justement comment créer et gérer nos certificats. Si ce n’est pas déjà fait je vous retrouve donc après la lecture… c’est bon ? Super.

### Le reverse proxy

C’est ici que cela devient intéressant. On va donc utiliser [Træfik](https://traefik.io/) qui va nous servir de reverse proxy et de load balancer.  Il va prendre en entrée la requête HTTP/HTTPS de l’utilisateur, la désencapsuler et rediriger celle-ci vers le *pod* adéquat. Si l’application possède plusieurs pods pour faire de la redondance, Traefik va répartir la charge entre ces différents pods.

![Traefik](https://docs.traefik.io/img/architecture.png)
<p style="text-align:right";>*(source : [https://docs.traefik.io/](https://docs.traefik.io/))*</p>

On peut très bien installer Traefik à l’extérieur du cluster, mais bon on en a un alors pourquoi ne pas en profiter ? On va donc installer Traefik via un manifest comme n’importe quelle application.

> Une raison de ne pas installer Traefik dans notre cluster serait d’optimiser les performances. Traefik va être assez chargé car il doit traiter toutes les requêtes qui atteignent le cluster. C’est donc une bonne idée de le mettre sur une machine costaude et indépendante du cluster. Après, comment vous allez le voir tout de suite, Traefik lui-même peut être réparti sur plusieurs instances dans le cluster, et peut donc aussi profiter des fonctionnalités de scaling, de résilience et  d’automatisation de Kubernetes. C’est un choix !



Avant d’installer Traefik sur notre cluster nous allons avoir besoin de rajouter le certificat généré dans le paragraphe précédent en tant que secret dans notre cluster. Et pour faire cela nous allons devoir créer le namespace *traefik*.

```bash
kubectl create namespace traefik
```

Ensuite, il faut créer le secret en lui-même. Ça se fait en une ligne avec cette commande (attention vous devez être dans le même répertoire que votre couple clé/certificat et évidement que les noms correspondent).


```bash
kubectl create secret tls home --cert=home.local.crt --key=device.key -n traefik
```

Il est aussi possible de faire en sorte que Traefik génère un certificat let's encrypt à l'installation. Pour cela il faut modifier la configuration du `traefik.toml` dans le *configmap* du manifest afin de rajouter les options ACME (voir [ici](https://docs.traefik.io/configuration/acme/) pour plus d’info).

Ensuite c’est bon on va pouvoir installer Traefik. Je vous propose de partir de ce manifest qui est celui qui correspond à ma propre configuration. A vous de l’adapter à la vôtre !

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: traefik
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller
rules:
- apiGroups:
  - ""
  resources: ["configmaps","secrets","endpoints","events","services"]
  verbs: ["list","watch","create","update","delete","get"]
- apiGroups:
  - ""
  - "extensions"
  resources: ["services","nodes","ingresses","pods","ingresses/status"]
  verbs: ["list","watch","create","update","delete","get"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: traefik-ingress-controller
subjects:
- kind: ServiceAccount
  name: traefik-ingress-controller
  namespace: traefik
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: traefik-ingress-controller
  namespace: traefik
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: traefik-conf
  namespace: traefik
data:
  traefik.toml: |-
    defaultEntryPoints = ["http","https"]
    [entryPoints]
      [entryPoints.http]
      address = ":80"
        [entryPoints.http.redirect]
        entryPoint = "https"
      [entryPoints.https]
      address = ":443"
        [entryPoints.https.tls]
          [[entryPoints.https.tls.certificates]]
          CertFile = "/ssl/tls.crt"
          KeyFile = "/ssl/tls.key"
    [web]
    address = ":8080"
---
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: traefik-ingress-controller
  namespace: traefik
  labels:
    k8s-app: traefik-ingress-lb
    kubernetes.io/cluster-service: "true"
spec:
  template:
     metadata:
       labels:
         k8s-app: traefik-ingress-lb
         name: traefik-ingress-lb
     spec:
       hostNetwork: true
       serviceAccountName: traefik-ingress-controller
       terminationGracePeriodSeconds: 60
       volumes:
       - name: ssl
         secret:
           secretName: home
       - name: config
         configMap:
           name: traefik-conf
       tolerations:
       - key: node-role.kubernetes.io/master
         effect: NoSchedule
       containers:
       - image: traefik:v1.3.1
         name: traefik-ingress-lb
         imagePullPolicy: Always
         volumeMounts:
         - mountPath: "/ssl"
           name: "ssl"
         - mountPath: "/config"
           name: "config"
         resources:
           requests:
             cpu: 100m
             memory: 20Mi
         ports:
         - containerPort: 80
         - containerPort: 443
         args:
         - --kubernetes
         - --configfile=/config/traefik.toml
---
apiVersion: v1
kind: Service
metadata:
  name: traefik-web-ui
  namespace: traefik
spec:
  selector:
    k8s-app: traefik-ingress-lb
  ports:
  - name: http
    port: 80
    targetPort: 8080
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: traefik-web-ui
  namespace: traefik
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  tls:
  - secretName: home
  rules:
  - host: "traefik.home.local"
    http:
      paths:
      - backend:
          serviceName: traefik-web-ui
          servicePort: http
```

Si rien ne change, vous pouvez utiliser directement ce manifest en faisant la commande suivante :

```bash
kubectl apply -f https://raw.githubusercontent.com/MatthieuGouel/k8s-manifests/master/traefik/traefik.yml
```

Enfin on va tout de même avoir envie de tester tout ça avec une « vraie » application, non ? Je vous ai préparé un second manifest qui met en place un serveur Nginx.

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: health
---
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: nginx-health
  namespace: health
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 1
  template:
    metadata:
        labels:
          app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-health
  namespace: health
  labels:
    app: nginx
spec:
  ports:
  - name: http
    port: 80
  selector:
    app: nginx
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nginx-health
  namespace: health
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  tls:
  - secretName: home
  rules:
  - host: health.home.local
    http:
      paths:
      - path: /
        backend:
          serviceName: nginx-health
          servicePort: http
```

Vous pouvez voir à la fin du manifest qu’il y a également un *ingress* qui pointe vers l’adresse `health.home.local` mais vous pouvez bien sûr modifier ce nom à votre convenance.

Après avoir appliqué la configuration avec la commande (toujours si rien ne change pour vous) :

```bash
kubectl apply -f https://raw.githubusercontent.com/matthieugouel/kubernetes-manifests/master/nginx-health/nginx-health.yml
```

Vous devriez normalement voir cette application sur le dashboard Traefik à l’adresse que vous avez renseignée lors de l’installation de celui-ci dans l’ingress (pour moi c’est `traefik.home.local`). Enfin vous pourrez accéder à l’application via le nom DNS de l’application nginx-health (pour moi `health.home.local`). N’oubliez pas de déclarer ces noms dans le DNS bien sûr.

## Conclusion

On ne dirait pas comme ça mais c’est un article assez dense ! Le jeu en vaut la chandelle car maintenant on a vraiment tout ce qu’il faut pour déployer des applications sur Kubernetes. On aurait pu s’arrêter là si la série s’appelait « installation d’un cluster kubernetes à la maison » mais on va aller encore plus loin car désormais le but va être d’automatiser le déploiement de nos applications de façon continue sur notre cluster. On va faire du vrai DevOps quoi !

Les articles de la série :

* [Développement continu à la maison (partie 1)](https://matthieugouel.github.io/blog/2018-05-14-developpement-continu-a-la-maison-partie-1/)
* [Développement continu à la maison (partie 2)](https://matthieugouel.github.io/blog/2018-05-21-developpement-continu-a-la-maison-partie-2/)
* [Développement continu à la maison (partie 3)](https://matthieugouel.github.io/blog/2018-06-04-developpement-continu-a-la-maison-partie-3/)
* [Développement continu à la maison (partie 4)](https://matthieugouel.github.io/blog/2018-06-27-developpement-continu-a-la-maison-partie-4/)
