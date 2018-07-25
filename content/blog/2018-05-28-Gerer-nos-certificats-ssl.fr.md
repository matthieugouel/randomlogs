---
title: Gérer nos certificats SSL
description: Gratuitement bien sûr !
date: "2018-05-28"
featured: "devops.jpg"
featuredalt: "devops"
featuredpath: "img/blog"
linktitle: ""
categories: ["Security"]
tags: ["Certificates"]
type: post
---

Dans la série [Développement continu à la maison (partie 1)](https://matthieugouel.github.io/blog/2018-05-14-developpement-continu-a-la-maison-partie-1/) nous allons mettre en place des services avec des endpoints en HTTPS sur Kubernetes. Pour ce  faire nous avons bien évidemment besoin d’un certificat valide, celui-ci issu d’une autorité de certification valide.  
Il y a plusieurs moyens de se procurer de tels certificats.

<!-- more -->

### Acheter un certificat

Alors oui on peut acheter un certificat à une autorité de certification ou bien à des revendeurs. Je le mentionne car cela existe mais franchement, franchement, n’achetez pas de certificat. Depuis plusieurs années maintenant on peut obtenir des certificats gratuitement de façon parfaitement légale et légitime alors pourquoi s’en priver.

### Obtenir un certificat gratuitement avec Let’s encrypt

Donc je disais qu’il existe des autorités de certification qui délivrent des certificats gratuitement. L’autorité de loin la plus connue est [Let’s Encrypt](https://letsencrypt.org/) qui provient de l’*Internet Security Research Group*  et est supportée par la *Linux Foundation*. Autant dire que ce n’est pas un vieux service lugubre utilisé par trois geeks au fond de leur garage. Ce certificat est reconnu par une grande majorité de navigateurs (voir [ici](https://letsencrypt.org/docs/certificate-compatibility/)) et a délivré à l’heure où j’écris ces lignes plus de 50 millions de certificats (voir [là](https://letsencrypt.org/stats/)).

Le mieux est que le service est totalement automatisé donc obtenir et manager des certificats se fait vraiment simplement. Et, cerise sur le gâteau, depuis le 13 mars 2018 Let’s encrypt supporte les certificats wildcard (détails [ici](https://community.letsencrypt.org/t/acme-v2-production-environment-wildcards/55578)) !

Pour commencer à s’amuser avec Let’s encrypt je vous propose de lire la [documentation officielle](https://letsencrypt.org/getting-started/) Vous commencez à me connaitre je ne vais pas vous faire un tutoriel pas à pas alors qu’une belle documentation maintenue existe.

### Créer son propre certificat racine

Lorsque j’ai commencé à créer mon système de développement continu à la maison Let’s encrypt ne permettait pas encore de faire des certificats wildcard. Or j’en avais besoin car je voulais qu’un seul certificat sécurise toutes les applications de mon cluster Kubernetes. Ce n’est pas obligatoire, on peut générer un certificat pour chaque service mais le nombre de certificats que l’on peut demander à Let’s encrypt est limité dans le temps et surtout je ne voulais pas trop me prendre la tête à manager plein de certificats.

De plus mes services sont sur mon réseau local et on va pas se mentir je suis le seul à intervenir dessus. Il me suffit donc de créer un certificat racine (CA certificate), sans passer par une autorité de certification, l’installer sur mon laptop et le tour est joué ! Cette CA va être en mesure de signer un ou plusieurs certificats, qui pourront tout à fait être wildcard. Aucun problème.

> Aucun problème … pas tout à fait. Effectivement je n’aurais pas de problème à accéder à mes ressources en HTTPS depuis ma machine. Il faudra installer le certificat racine sur chaque machine mais peu importe je n’ai qu’un seul laptop. Par contre là où ça peut poser problème c’est quand les services ont besoin de communiquer entre eux de façon sécurisée. Je vous rappelle que ces services sont sur Kubernetes dans des conteneurs Docker. Souvent, et c’est d’ailleurs un des avantages de Docker, on a très envie d’utiliser directement l’image officielle qui elle ne connaît pas votre super autorité de certification. Dans ces cas-là, donc, il faut soit se débrouiller pour que les services communiquent de façon *insecure* soit installer le certificat racine sur le où les conteneurs concernés.

En fait créer une CA n’est pas très compliqué, de multiples tutoriels existent à ce sujet sur le web. Le souci est plutôt créer une CA qui est acceptée par les navigateurs, en particulier par Chrome qui est très tatillon. Ainsi il n’est pas rare d’avoir une erreur du type `missing_subjectAltName` alors que nous avons un certificat tout a fait valide. Après quelques recherches je suis tombé sur ce [post](https://stackoverflow.com/questions/7580508/getting-chrome-to-accept-self-signed-localhost-certificate/43666288#43666288) qui donne des petits scripts bash pour générer une CA et un certificat signé par celle-ci. PS: N’oubliez pas d’installer la CA sur votre laptop !

En bonus je vous mets en bookmark les instructions à rajouter dans votre Dockerfile pour installer votre CA dans vos images Docker (avec une base type Debian).
Dans l’exemple le certificat racine s’appelle `home.local-ca.crt` en local et on souhaite qu’il apparaisse en tant que `home.local.crt` dans le conteneur.  Évidement votre CA doit être présente dans le même répertoire que votre Dockerfile et s’appeler correctement.

```  
RUN apt-get update && apt-get install -y ca-certificates
RUN mkdir -p /usr/local/share/ca-certificates/home.local
COPY home.local-ca.crt /usr/local/share/ca-certificates/home.local/home.local.crt
RUN update-ca-certificates
```

Notez que l’extension `.crt` est importante. Pour convertir un `.pem`en `.crt`il suffit de faire :

```bash
openssl x509 -outform der -in home.local-ca.pem -out home.local-ca.crt
```
