---
title: Développement continu à la maison (partie 2)
description: Installation du cluster Kubernetes
date: "2018-05-21"
featured: "devops.jpg"
featuredalt: "devops"
featuredpath: "img/blog"
linktitle: ""
categories: ["DevOps"]
tags: ["Kubernetes"]
type: post
---

Et voilà nous y sommes.  Après une longue introduction la fois précédente, aujourd’hui, on s’attaque enfin à la pratique ! Cet article a pour but d’installer notre cluster Kubernetes. C’est la première pierre à notre édifice (et non des moindres !) qui nous permettra ensuite d’installer tous nos services. Il s’agit donc de ne pas se rater !

<!-- more -->

Ça tombe bien parce que Kubernetes possède une documentation très complète. Je n’ai pas vraiment envie de copier-coller la documentation officielle dans cet article c’est pourquoi je vais abuser de liens vers celle-ci. De cette façon cet article sera à jour plus longtemps et évitera les redits.

Pour commencer détaillons les différents choix d’installation qui s’offrent à nous.  La documentation officielle nous fournit une [page](https://kubernetes.io/docs/setup/pick-right-solution/#local-machine-solutions) nous permettant de choisir la bonne solution dans cet océan de possibilités. Je vous conseille de la parcourir, mais je vais résumer la situation à la lumière de notre ce que l’on souhaite réaliser. Notez bien que je ne vais pas parler de toutes les solutions possibles, juste celles qui me semblent intéressantes (et pas trop underground) pour notre usage.

## Les solutions « cloud »

C’est la solution la plus simple, ici rien à faire ! Il y a des [tonnes](https://kubernetes.io/docs/setup/pick-right-solution/#hosted-solutions) de cloud providers qui offrent des solutions Kubernetes tout cuit dans le bec. Je pense que cela peut être utile à notre niveau pour tester mais le réel avantage est à mon avis pour les entreprises qui n’ont pas forcément les moyens ou la stratégie de monter une architecture complexe pour leur business. Avec cette solution ils profitent de la mise à l’échelle, de la maintenabilité et des infrastructures de professionnels et ils payent à la carte ce dont ils ont besoin.

C’est plutôt cool mais dans notre cas ça ne nous intéresse pas vraiment. Eh oui le titre de cette série est tout de même **Développement continu à la maison** alors ce serait un peu tricher que d’utiliser un cloud provider vous ne trouvez pas ?

## Les solutions en localhost

Ici c’est votre propre ordinateur qui fait office de cluster Kubernetes. C’est pratique car il n’y a pas besoin de dévouer une machine exprès pour le cluster. De plus, ces solutions sont souvent assez rapides à mettre en place.  C’est vraiment parfait pour faire des tests sans vouloir se prendre la tête. Évidemment l’inconvénient ce sont les performances.  Je vais tout de même faire un tour d’horizon de ces solutions pour ceux qui souhaiteraient suivre cette série dans un but davantage pédagogique que « productif ».  C’est aussi un bon moyen pour continuer à bosser sur sa machine quand on n'est pas sur le réseau de production, par exemple dans le train ou dans l’avion.

### Les machine virtuelles toutes faite

C’est [ici](https://kubernetes.io/docs/setup/pick-right-solution/#on-premises-vms) que ça se passe. Le but est d’utiliser une image, souvent une base CoreOS, pour rapidement monter une ou plusieurs machines virtuelles. Comme vous pouvez le voir plein de solutions existent, mais je pense que le plus simple et le plus répandu à l’heure actuelle est d’utiliser Vagrant.
Vagrant est un logiciel open source qui aide à créer et manager des instances virtuelles. C’est très souvent utilisé pour monter rapidement et simplement des architectures de développement ou de test. Par exemple, on peut monter un cluster multi-node sur sa machine en suivant ce [lien](https://kubernetes.io/docs/getting-started-guides/coreos/).

### Minikube

Minikube est un assistant permettant l’installation de Kubernetes en single-node (le master est aussi un worker) sur des environnements virtuels. En fait c’est encore plus facile que les VM toutes faite car il suffit de spécifier à Minikube l’hyperviseur utilisé (Virtualbox, KVM, VMware, …) et il se charge ensuite de faire tout le travail. C’est à mon sens la meilleure solution pour monter rapidement un environnement de test. Suivez ce [guide](https://kubernetes.io/docs/getting-started-guides/minikube/) pour l’installation.

## Les solutions sur machine(s) dédié(s)

On se rapproche d’un environnement de production. Même si les solutions en localhost sont pratiques pour commencer, nous on voudrait un vrai setup de développement continu à la maison comme les pros, non ? Là encore plusieurs solutions s’offrent à nous.

### Kubeadm

Kubeadm est un assistant d’installation pour un cluster multi-node. C’est vraiment cool car cela permet d’installer notre cluster sans prise de tête tout ayant la possibilité de choisir les options les plus significatives (notamment la gestion du réseau).  Je pense que cette solution est parfaite pour quelqu’un qui souhaite monter un cluster en gardant un minimum de contrôle, sans forcément devoir y passer trop de temps. Personnellement c’est la solution que j’ai choisie pour mon cluster. Un autre avantage d’utiliser cet assistant est qu’il simplifie la gestion des mises à jour du cluster et permet de profiter assez simplement des dernières fonctionnalités (comprenez ici que, comme c’est plus simple de maintenir le cluster, on aura moins la flemme de faire les mises à jour). Suivez donc ce [guide](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/) si vous partez dans cette direction.

### Bare metal

Dernière façon de faire présentée [ici](https://kubernetes.io/docs/setup/pick-right-solution/#bare-metal), celle du barbu. Et oui il n’y pas nécessairement besoin d’utiliser un assistant comme Minikube ou Kubeadm pour installer le cluster, on peut tout aussi bien le faire à la main sur une machine / VM muni de Fedora ou CoreOS. Cette solution est celle qui permettra d’être le plus précis dans ce qu’on veut faire mais ce sera aussi la plus complexe. À vous de voir si cette solution vous convient et si vous souhaitez passer du temps supplémentaire là dessus. Je pense sincèrement que c’est utile pour approfondir sa connaissance du produit, c’est un peu le bonus « pour finir le jeu à 100% ».


## Astuces

Comme je vous l’ai dit je ne vais pas détailler l’installation du cluster pour chaque solution. En informatique, tout ce qu’on écrit à une date de péremption donc je préfère vous laisser des liens vers une documentation officielle qui sera maintenue à jour.
Mais bon, pour pas vous laisser comme ça je vous mets ci-dessous quelques astuces que j’ai employé quand j’ai installé mon cluster.

> Ces astuces ne sont pas obligatoires pour monter votre cluster. Ce sont juste certains choix que j’ai fait, mais je ne dis pas que ce sont les meilleurs. Je les mets là plutôt comme exemple / bookmark à prendre avec des pincettes et qui pourraient éventuellement vous aider à debugger en cas de soucis.

### Mes choix techniques

Personnellement j’ai utilisé 3 VM sous CentOS 7 installées via VirtualBox avec l'ISO Minimal. N’ayant pas la joie d’avoir de vrais serveurs à la maison j’ai utilisé mon ancien laptop qui fait très bien l’affaire (comptez au moins 8Go de RAM tout de même).  Une VM est **master** et ne portera pas d’applications de production et les deux autres sont des **workers**. Ça me donne un peu de redondance sans avoir  besoin de trop de RAM (2Go / VM). Si vous avez 16Go sous la main (et un bon processeur au passage) je vous conseille de faire 4 VM de 3Go de RAM avec 1 master et 3 workers, avec ça vous serez plutôt bien. Enfin sachez qu’il est possible de faire de la redondance sur les masters, ce qui peut être intéressant également.

Pour la gestion du réseau j’ai utilisé [Flannel](https://github.com/coreos/flannel#flannel), mais vous pouvez choisir celui qui vous chante bien évidemment (un petit comparatif [ici](https://kubernetes.io/docs/concepts/cluster-administration/networking/)). Flannel est très simple à mettre en place et fait le travail tout seul. Si vous souhaitez aller plus loin (et faire du full L3 par exemple) vous pouvez regarder du côté de [Calico](https://www.projectcalico.org/).

Sachez enfin que sur mon réseau j’ai un DHCP et un DNS. Ce dernier est assez obligatoire pour avoir vos propres nom en `*.home.local` par exemple. Il y a pleins de bon tutoriels sur comment monter un serveur DNS à la maison donc je n’aborderais pas ce sujet ici mais je ferais un article prochainement sur tous les services *enablers* que j’utilise à côté de mon cluster. Dans le même registre vous pouvez également monter un serveur NFS pour gérer les volumes persistants de vos applications mais ce n’est pas une nécessité pour avoir un cluster fonctionnel. Vous pourrez toujours le faire plus tard.

### Bien préparer ses machines

> Notez que les astuces de ce paragraphe sont à faire sur chaque noeud du cluster (en root).

- Modifier l'hostname de la machine si besoin pour par exemple coller avec son nom DNS.

```bash
hostnamectl --set-hostname worker
```

- Vérifier que les machines sont dans le même réseau (ou pas si vous savez ce que vous faite).

- Les VM doivent avoir des MAC différentes. Vous pouvez le vérifier avec la commande  `ip addr`.

- Les VM doivent avoir des product_uuid différents. Vous pouvez le vérifier avec cette commande :

```bash
cat /sys/class/dmi/id/product_uui
```

- Désactiver le swap:

Voir l’état du swap :

```bash
cat /proc/swaps
```

Supprimer le swap :

```bash
swapoff -a
```

 Et commenter la ligne de swap dans `/etc/fstab`

- Désactiver SElinux

Ce n’est pas recommandé mais cela peut vous éviter bien des soucis. Vous pouvez sauter cette étape si vous avez l’habitude d’utiliser SElinux ou si vous êtes obligé par mesure de sécurité.

Pour voir l'état de SElinux :

```bash
sestatus
```

Désactiver pour la session courante:

```bash
setenforce 0
```

Désactiver définitivement (ne faites pas un copier-coller) :

```bash
vi /etc/sysconfig/selinux + SELINUX=disabled
```

- Pour ne pas s’embêter avec le par-feux système et utiliser iptable à la place :

```bash
yum install iptables-services.x86_64 -y
systemctl stop firewalld.service
systemctl disable firewalld.service
systemctl mask firewalld.service
systemctl start iptables
systemctl enable iptables
systemctl unmask iptables
iptables -F
service iptables save
```

* Passer le trafic du bridge IPV4 dans la chaine iptables :

```bash
modprobe br_netfilter
sysctl net.bridge.bridge-nf-call-iptables=1
```

Et même pour être serein :

```bash
cat <<EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```

* Installer ntpd

```bash
yum install ntp
systemctl enable ntpd
systemctl start ntpd
```

### Installation du cluster avec Kubeadm

Ici je vous laisse suivre la documentation pour installer Kubeadm [ici](https://kubernetes.io/docs/setup/independent/install-kubeadm/).
Ensuite pour la création du cluster il suffit de suivre cette documentation [là](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/).

Avec ça vous devriez vous en sortir sans problème ! :) N’hésitez pas à poser des questions en commentaire de cet article si vous êtes bloqué.

### Tester notre cluster

On peut vérifier que notre cluster est fonctionnel avec une application de test fournie par Kubernetes :

```bash
kubectl create namespace sock-shop
kubectl apply -n sock-shop -f "https://github.com/microservices-demo/microservices-demo/blob/master/deploy/kubernetes/complete-demo.yaml?raw=true"
```

L'application est normalement installée. On peut récupérer le port exposé avec la commande suivante :

```bash
kubectl -n sock-shop get svc front-end
```

```
NAME CLUSTER-IP EXTERNAL-IP PORT(S) AGE
front-end 10.110.250.153 <nodes> 80:30001/TCP 59s
```

On voit ici que le port est le 30001 mais bien sûr chez vous il sera peut-être différent.

On attend que tout est bien en running (cela peut prendre quelques secondes à quelques minutes). Vous pouvez vérifier l’état du déploiement avec la commande :

```bash
kubectl get pods -n sock-shop
```

Une fois que le pod est à l’état *running* on peut tester de s’y connecter avec un navigateur :

```
http://<master_ip>:<port>
```

Si ça marche c'est cool on peut supprimer l’application de test :

```bash
kubectl delete namespace sock-shop
```

## Conclusion

Eh beh, je sais pas pour vous mais moi je suis tout de même assez fier d’avoir mon cluster Kubernetes multi-node qui tourne à la maison ! Peu importe le temps que ça vous a pris pour le monter, l’objectif est double : avoir une base pour nos développements futurs et apprendre quelque chose. Personnellement j’ai pris mon temps pour faire tout ça mais j’ai l’impression d’avoir une base assez solide sur laquelle je peux me reposer. Cela reste un MVP (Minimum Viable Product) mais nous allons étoffer tout cela au fur et à mesure.

Dans le prochain article on déploiera notre première vraie application sur notre cluster ! Peut-Être que vous l’avez déjà deviné mais ce sera [Træfik](https://traefik.io/), qui nous permettra de gérer l’interface entre notre cluster tout frais et le reste du monde (ou du moins le reste de notre réseau local).

Les articles de la série :

* [Développement continu à la maison (partie 1)](https://matthieugouel.github.io/blog/2018-05-14-developpement-continu-a-la-maison-partie-1/)
* [Développement continu à la maison (partie 2)](https://matthieugouel.github.io/blog/2018-05-20-developpement-continu-a-la-maison-partie-2/)
