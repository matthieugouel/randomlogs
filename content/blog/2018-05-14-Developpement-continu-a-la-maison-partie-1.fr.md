---
title: Développement continu à la maison (partie 1)
description: Introduction
date: "2018-05-14"
featured: "devops.jpg"
featuredalt: "devops"
featuredpath: "img/blog"
linktitle: ""
categories: ["DevOps"]
tags: ["Kubernetes"]
type: post
---

Premier article du thème DevOps. Ce thème regroupera les articles parlant des sujets d’automatisation et de déploiement d’applications dans un environment de production. C’est vaste !

<!-- more -->

Je vous propose dans cette série d’articles de construire pas à pas notre écosystème moderne de développement continu.  Nous commencerons par installer un orchestrateur qui nous servira de fondations pour notre système. Sur ces fondations nous déploierons les services nécessaires pour faire de l’intégration continue comme un logiciel de gestion de version, un système de développement continu, des dépôts etc … Enfin, une fois tout ça fait il ne nous restera plus qu’à mettre en place notre workflow afin de déployer nos propres applications au sein même de notre orchestrateur. La boucle est bouclée !

## Kubernetes comme orchestrateur

[Kubernetes](https://fr.wikipedia.org/wiki/Kubernetes) ( ou « k8s » pour les intimes) est un orchestrateur de conteneurs open source développé par Google.
Le but de Kubernetes est de permettre le déploiement, le maintien et la mise à l’échelle d’applications conteneurisés. Au lieu d’avoir une grosse application monolithique en production, nous pourrons segmenter les différentes parties de notre application en conteneurs et Kubernetes sera en charge de les manager.

Les conteneurs sont vraiment pratique pendant la phrase de développement. ils permettent d’avoir les même conditions système pour tous les développeurs, de ne pas polluer son environnement de travail avec des tonnes d’applications et inversement, de segmenter les différentes parties critiques, de gérer les versions, etc …
Par contre quand il s’agit de mettre tout ça en production, ça devient tout de suite plus compliqué. On va avoir tendance à vouloir faire une machine virtuelle par conteneurs mais les conteneurs et les VM c’est pas tout à fait pareil.  Que se passe-t-il quand une VM tombe ? Ou un hyperviseur ? Comment mettre à échelle efficacement ?

C’est là que Kubernetes entre en scène.  Les conteneurs qui servent pour le développement et le staging servent également en production. Cela permet d’avoir à tout moment exactement les mêmes conditions en dev et en prod (donc plus de sueur froide la veille d’un déploiement majeur) . Si un conteneur tombe, Kubernetes se chargera de le reinstancier et comme celui-ci fonctionne en cluster, si un node du cluster tombe alors il reinstanciera tous les conteneurs de ce node autre part. Besoin de plus de ressources ? Il suffit d’ajouter un node et / ou multiplier les conteneurs qui sont trop chargés.

Cette stratégie de conteneurisation unifiée sera de plus incroyablement efficace pour faire du development continu. Eh oui pourquoi s’embêter à justement faire des déploiements majeurs qui change la moitié de la codebase alors que l’on peut le faire de façon controlée et automatisée par petites touches (comme à la fin de chaque sprint par exemple).

Voilà j’espère vous avoir un peu convaincu sur l’utilisation de cette technologie comme base de notre super système de production à la maison, mais laissez-moi tout de même résumer :

* Mêmes conditions en dev et en prod
* Gestion du maintien et de la mise à l’échelle des conteneurs qui peut même se faire automatiquement (auto scaling)
* Facilitation du développement continu
* Gestion du réseau et de tout ce qui doit être mis en place pour la communication inter-conteneurs (DNS, DHCP, …)
* Gestion des volumes persistants, des secrets, …

Ah oui, Kubernetes est un orchestrateurs de conteneurs. Ok, mais quoi comme conteneurs exactement ?
Même si ce n’est pas obligatoire, dans la grande majorité des cas les gens utilisent Kubernetes conjointement avec [Docker](https://www.docker.com/).  D’autres systèmes de conteneurisation existent (je pense notamment à Rocket) mais bon pour notre cas il n’y a pas vraiment d’intérêt à choisir autre chose que Docker.

## Les services de base au développement continu

Voilà donc on a les fondations de notre système DevOps mais malheureusement ça ne va pas suffire pour créer notre workflow. Pour cela il va nous falloir des services dans notre cluster qui permettront de répondre à ces problématiques :

* Gérer l’accès depuis l’extérieur de notre cluster et la sécurité ce ces accès
* Gérer les dépôts de nos images docker voire de nos libraires (nos package Python par exemple)
* Gérer le versioning de notre code et l’intégration continue

Pour tout ce qui est accès à nos conteneurs depuis l’extérieur nous allons utiliser [Træfik](https://traefik.io/). C’est un reverse proxy couramment utilisé pour remplir cette tâche et il fera l’intermédiaire entre nos domaines DNS et nos conteneurs. Ce sera donc l’entrée et la sortie de notre cluster, qui se fera de façon sécurisée avec notre propre CA.

Ensuite nous mettrons en place un dépôt d’image Docker (Docker registry). Cela nous permettra de stocker nos images et Kubernetes ira piocher dedans quand il en aura besoin.

Enfin pour le versioning on va bien évidemment utiliser Git. A la base j’étais parti pour utiliser Gitlab avec Gitlab-CI comme système d’intégration continue. Le soucis c’est que Gitlab demandait trop de resources pour mon pauvre petit cluster. Je me suis donc rabattu sur [Gogs](https://gogs.io/) comme gestionnaire de version et [Drone ](https://drone.io/) comme système d’intégration continue. Et franchement je ne suis pas déçu !

## Un Workflow DevOps

Le principe sera donc de développer sur notre machine, puis d’avoir la possibilité de simuler la prod avec Docker et Docker Compose en local, avant d’envoyer notre code sur notre gestionnaire de version préféré. Le code envoyé entrainera le déclenchement d’un processus d’intégration continue (ou CI/CD pour Continuous Integration / Continuous Deployment)  qui se chargera de lancer les tests et les builds.  Si l’on push sur la « master » alors cela déclenchera en plus le déploiement en production.
A chaque nouvelle fonctionnalité on code donc sur une branche spécifique ou sur la « develop », et quand on est content on merge notre code tout neuf sur la « master » et le déploiement se fait automatiquement sur Kubernetes.

Le déploiement automatisé est donc composé de plusieurs parties :

* Tests de non-régression (unitaires, fonctionnels, …)
* Builds (quand il y’en a besoin)
* Création et déploiement de l’image sur le Docker registry
* Remplacement de l’image docker de production par la nouvelle sur Kubernetes

## Conclusion

On a donc fait le tour des technologies que l’on va mettre en place dans cette série.  C’était beaucoup de mots pour pas beaucoup d’action mais il me semblait nécessaire de planter le décor avant de se lancer tête baissée.

Dans le prochain article de ce thème on mettra enfin la main à la pâte. Il s’agira de créer notre propre cluster Kubernetes !
A bientôt donc pour poser la première brique de notre écosystème DevOps.

Les articles de la série :

* [Développement continu à la maison (partie 1)](https://matthieugouel.github.io/blog/2018-05-14-developpement-continu-a-la-maison-partie-1/)
* [Développement continu à la maison (partie 2)](https://matthieugouel.github.io/blog/2018-05-21-developpement-continu-a-la-maison-partie-2/)
* [Développement continu à la maison (partie 3)](https://matthieugouel.github.io/blog/2018-06-04-developpement-continu-a-la-maison-partie-3/)
* [Développement continu à la maison (partie 4)](https://matthieugouel.github.io/blog/2018-06-27-developpement-continu-a-la-maison-partie-4/)
