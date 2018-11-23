---
title: Oops!... I Did It Again
description: Another side project interpreter.
date: "2018-11-22"
featured: "dev.jpg"
featuredalt: "dev"
featuredpath: "img/blog"
linktitle: ""
categories: ["Développement"]
tags: ["Gibica", "Bjørn"]
type: post
---

# Oops!... I Did It Again

Il y a quelques mois je vous ai parlé de [Gibica](https://github.com/matthieugouel/gibica), mon tout premier interpréteur en Python.  Eh bien en fait j’en ai fait un autre, cette fois ci en *Rust*, et il s’appelle [Bjørn](https://github.com/matthieugouel/bjorn). Voilà. 

Mais enfin pourquoi s’infliger une telle peine ?! Sachant  que ce second interpréteur m’a pris encore plus de temps à coder que le premier  ?! C’est ce qu’on va voir aujourd’hui. 

<!-- more --> 

## Revenons un instant sur Gibica

Pour rappel, le langage ressemble à cela : 

```
#
# Get the nth number of the fibonacci sequence.
#

def fibonacci(n) {

    if n <= 1 {
        return 1;
    }

    return fibonacci(n - 2) + fibonacci(n - 1);
}

print(fibonacci(10));
```

Au moment  de l’article de présentation j’étais arrivé à une sorte de jalon. Assez de fonctionnalités étaient implémentées pour jouer un peu avec … et pour se rendre compte que les performances n’étaient pas tops.

Les raisons de ces performances médiocres sont multiples mais les deux raisons principales étaient prévisibles et sont lié à l’utilisation de Python, le langage avec lequel *Gibica* est codé.

* Python est lui-même un langage interprété. Ce qui fait que quand on lance un programme en *Gibica* avec cette commande :

```
$ gibica program.gbc
```

En fait on lance une commande qui ressemble à celle-ci : 

```
$ python gibica.py program.gbc
```

Grossièrement, on a donc Python (*CPython* pour être précis) qui interprète un programme écrit en Python, *Gibica*, qui lui-même interprète un langage écrit en *Gibica*, ` program.gbc`.  On a donc une cascade d’interprétation qui est loin d’être performante.

> Pour être tout à fait précis, Python ne fait pas la totalité de son interprétation à chaque fois que l’on lance *Gibica*. En effet, celui-ci a été codé sous forme de package, ce qui fait qu’il est précompilé en *Bytecode*. Python n’a donc plus qu’à interpréter ce *Bytecode* à chaque exécution, mais il n’empêche que cela reste une interprétation !

* Python n’est juste pas fait pour ce genre de projets. On l’a vu c’est un langage interprété et comme beaucoup de ses semblables c’est un langage *high level*. Cela ne signifie pas que c’est un langage compliqué hein mais plutôt que le langage abstrait fortement ce qu’il se passe au niveau des composants physiques. Cela à l’avantage de simplifier l’utilisation du langage et de se concentrer sur le code en lui-même, mais par conséquent Python fait beaucoup de choix à notre place. Le souci c’est que quand on fait un langage on a avant tout besoin de contrôle, car, plus on a accès aux couches basses du système, et plus on va pouvoir faire des choix poussés quant à la façon dont va être implémenté notre langage. Et bien sûr, moins le langage est abstrait, plus on va pouvoir optimiser le code en fonction de nos besoins, et ainsi gagner en performance. 

Alors pourquoi ai-je fait *Gibica* en Python alors que je connaissais pertinemment ses limitations ? Tout simplement car créer un langage n’est pas une mince affaire, du moins au début. Je souhaitais donc commencer avec un langage que je maitrise pour justement me concentrer sur ce qui est important, c’est-à-dire le langage lui-même. 

Je savais dès le début que *Gibica* ne serait pas le prochain interpréteur à la mode qui détrônerait *CPython* mais qu’il serait plutôt un interpréteur à usage éducatif,  et qu’il me permettrait également de tester rapidement de nouvelles fonctionnalités ou de nouveaux design. Enfin, je souhaitais également voir dans quelles mesures les raisons cités plus haut seraient importantes dans les performances du langage.

À la suite de l’article précédent j’avais donc plusieurs choix :

1. Continuer de développer les fonctionnalités de *Gibica*, sachant que, même en passant du temps à essayer de l’optimiser, je ferais toujours face aux limitations discutées précédemment.
2. Tenter de transformer *Gibica* en utilisant un langage ressemblant au Python mais qui serait compilé afin de gagner en performances. 
3. Tenter de transformer *Gibica* en compilateur.

Pour moi l’option 1 n’était pas vraiment envisageable car ça ne ferait que retarder (ou ignorer) le problème de fond. Il arrivera forcément, en ajoutant de plus en plus de fonctionnalités, que l’interpréteur devienne complètement inutilisable et que le projet finisse en *Meh*. 

L’option 2 est intéressante. Il existe plusieurs langages qui ont une syntaxe qui ressemble au Python, mais qui se compile, ce qui améliorerait les performances tout en réutilisant une bonne partie du code. Ces langages offrent souvent des fonctionnalités de plus bas niveau ce qui ajouterait du contrôle. J’ai tenté deux approches.  

La première, la plus simple, était de compiler *Gibica* en *Cython*. *Cython* est un langage qui étend les possibilités de Python pour écrire facilement  des extensions  de plus bas niveau Au passage, il améliore les performances du code écrit en Python en le compilant en C ou C++, et fourni en plus la possibilité de gérer le typage explicite des objets. J’ai fait le bourrin en passant tout mon code en *Cython*, mais après quelques tests, je me suis aperçu que cela n’améliorait pas beaucoup les performances globales. Je pense que *Cython* n’est pas fait pour optimiser ce genre de code, mais est plutôt pertinent pour optimiser certains types d’algorithmes qui manipulent par exemples de gros tableaux ou effectuent des calculs répétitifs. 

J’ai également tenté de compiler *Gibica* en *RPython*. C’est un langage qui reprends la majeure partie des fonctionnalités de Python, mais compile le code pour directement créer un exécutable. Cela paraît être la meilleure alternative surtout que *Pypy*, un interpréteur Python très utilisé (qui est au passage plus rapide que *CPython*)  est codé en *RPython*. Sauf que *RPython* n’implémente qu’une partie de Python et il aurait fallu faire beaucoup de modifications  pour rendre le code compatible. C’est un projet intéressant, mais j’ai le sentiment que j’aurais perdu le côté « simple » et «  éducatif » de *Gibica*.

L’option 3 est selon moi la plus intéressante. *Gibica* est lent, certes, mais en le transformant en compilateur, on déplacerait cette lenteur non pas à l’exécution du programme mais à la compilation, ce qui est somme toute bien plus acceptable. Seulement, modifier proprement *Gibica* en compilateur n’est pas évident et mérite un article à lui tout seul.

Au final je n’ai choisi aucune de ces solutions. En explorant l’option 2 je me suis rendu compte qu’il serait vraiment plus propre d’utiliser un langage complet compilé et bas niveau plutôt que de tenter de tordre *Gibica* dans tous les sens… Et puis j’avais envie de faire du *Rust*.

## On reprend le même design, et on recommence !

Je trouvais vraiment intéressant l’idée de reprendre exactement la même architecture logiciel que *Gibica*, mais dans un langage adapté au problème, afin de voir l’impact des deux limitations discutées plus haut sur les performances.  

J’ai donc pas trop cherché à innover de ce côté-là ce qui m’a permis de me concentrer sur mon apprentissage d’un nouveau langage. 

Pour le choix de ce nouveau langage justement, j’ai un peu hésité entre deux possibilités. 

1. [Go](https://fr.wikipedia.org/wiki/Go_(langage)), un langage compilé, performant et sûr. Il est également rapide à prendre en main car se veut simple et efficace. 
2. [Rust](https://fr.wikipedia.org/wiki/Rust_(langage)), langage compilé, un peu plus performant et sûr. Il offre plus de contrôle que *Go* à mon sens car plus bas niveau. Par contre, bien qu’il soit très formateur d’apprendre ce langage, celui-ci est assez difficile à prendre en main (je fais encore des cauchemars impliquant un borrow checker et des méthodes mutables) .

Comme vous l’avez deviné j’ai choisi de coder ce nouvel interpréteur en *Rust*. Et franchement je ne le regrette pas !

Me voilà quelques mois après cette décision avec un second interpréteur, *Bjørn*. Le langage associé est un peu différent de *Gibica* pour simplifier l’expérience utilisateur. Pour ceux qui codent en python je pense que vous n’allez pas être dépaysé ! 

Un petit exemple : 

```
#
# This is a Bjørn program.
# Get the nth number of the fibonacci sequence.
#

def fibonacci(n):
    if n <= 1:
        return 1
    return fibonacci(n - 2) + fibonacci(n - 1)

print(fibonacci(20))
```

Les différences principales avec *Gibica* sont les blocs indentés (fini  les accolades !) et la disparition de la mutabilité explicite (fini le mot clé `mut` !).  Également, comme on peut le voir dans le commentaire de l’exemple ci-dessus, *Bjørn* gère les caractères [unicode](https://fr.wikipedia.org/wiki/Unicode) (contrairement à *CPython*). Enfin les point-virgule à la fin des expressions ont disparu. 

## Alors, finalement, c’était utile ?
Maintenant que *Gibica* et *Bjørn* ont le même nombre de fonctionnalités on va enfin pouvoir comparer leurs performances et ainsi vérifier si le choix judicieux d’un langage de programmation peut changer la donne. Je vais également confronter les résultats avec les performances de *CPython 3.7* afin d’avoir une base de comparaison.

> Note: Les benchmarks suivants ont été réalisés avec [hyperfine](https://github.com/sharkdp/hyperfine), une application codée en Rust qui permet de faire des tests de performances sur des applications en ligne de commande avec une bonne précision. __Sur tous les graphiques, la valeur la plus basse est la meilleure.__

Commençons avec la déclaration simple d’une variable.  C’est une instruction basique qui va nous permettre d’avoir une bonne idée de l’empreinte des différents interpréteurs. 

{{< img-post path="date" file="variable_declaration.png" alt="Variable Declaration" type="center" >}}

On remarque que *Gibica* est déjà  loin derrière ! Par contre *Bjørn* est plus rapide que *CPython*, cool non ? 

On poursuit avec la déclaration et l’appel d’une fonction simple : 

{{< img-post path="date" file="simple_function.png" alt="Variable Declaration" type="center" >}}

On observe des résultats similaires. *Gibica* est toujours dans les choux mais *Bjørn* tiens bon !

Enfin on va tenter de stresser un peu le langage en effectuant un peu de calculs. Classiquement on peut utiliser une fonction récursive pour calculer les n premiers éléments de la suite de Fibonacci. Ce qui est bien avec cette méthode c’est que cette fonction a une complexité algorithmique exponentielle donc une petite augmentation de n va augmenter fortement le nombre de calculs. 

D’abord les performances pour `n=10` et `n=20` : 

{{< img-post path="date" file="recursive_function.png" alt="Variable Declaration" type="center" >}}

Même pas besoin de mentionner que *Gibica* est en train de cracher ses poumons sur le côté de la route. Non la vraie information c’est que *Bjørn* commence à montrer des signes de faiblesses. Alors qu’il est plus rapide que *CPython* pour `n=10`, cela commence déjà à se gâter pour `n=20` même si le temps d’exécution reste raisonnable. 

Par contre pour `n=30` ce n’est plus la même : 

{{< img-post path="date" file="recursive_function_expensive.png" alt="Variable Declaration" type="center" >}}

Ici *Gibica* prends plus d’une minute et demi pour s’exécuter, et *Bjørn* prends tout de même 11 secondes ! On dirait que ça va si on le compare à *Gibica* mais en fait non car*CPython* ne prends lui que 400 millisecondes … Damn, le flibustier !

Bon, avec ces tests on peut faire quelques petites conclusions :

* Le passage de *Python* à *Rust* a été plus qu’efficace. Cela montre que l’utilisation d’un langage approprié à son besoin peut vraiment faire toute la différence dans un projet. 
* Le choix d’un langage n’est pas suffisant. *CPython* est codé en C (on aurait pu s’en douter), qui est un langage plus ou moins aussi performant que *Rust* à l’heure actuelle. Pourtant on voit bien des grosses différences dans certains cas. *Bjørn* est très rapide la plupart du temps, mais quand il s’agit de faire des calculs récursifs  il ne monte pas à l’échelle. Mais pourquoi ?!

Pour être honnête, *Bjørn* n’est pas plus rapide que *CPython* parce que je suis un développeur de génie … non c’est tout simplement qu’il n’a, ni le design, ni le nombre de fonctionnalités de *CPython*. 

Et là il faut se rappeler comment fonctionne *CPython* : Après avoir transformé le programme utilisateur en un arbre syntaxique abstrait (ou AST), il compile celui-ci en un langage intermédiaire : le *Bytecode*. Ainsi, ce n’est pas directement l’AST que *CPython* interprète mais bien ce fameux *Bytecode*, ce qui a pour conséquences les points suivant.

* Il est nécessaire d’initialiser pas mal de choses pour compiler et interpréter le *Bytecode* (la fameuse et si mal nommée VM)  donc *CPython* dépense un temps fixe à chaque exécution (30 millisecondes à la louche sur mon laptop). 
* Une fois le *Bytecode* compilé, son interprétation est beaucoup plus rapide que l’interprétation direct de l’AST. Donc dans notre exemple, une fois la fonction `fibonacci` compilé en *Bytecode*, *Cpython* n’a plus qu’à l’utiliser autant de fois qu’il faut sans avoir à recopier l’AST à chaque fois. 

On comprend donc mieux pourquoi *Bjørn* et souvent plus rapide que *CPython* : il interprète directement l’AST donc ne perd pas du temps à générer de *Bytecode*. De plus, du fait qu'il possède moins de fonctionnalités, il a moins d’objets à charger en mémoire à l'initialisation. Par contre quand il s’agit de faire plein de calculs répétitifs, là *CPython* explose tout le monde !

###  Et la suite maintenant ?

J’ai vraiment appris énormément de choses durant ces derniers mois. D’abord avec *Gibica* où j’ai pu comprendre le processus de création d’un langage et d’un interpréteur. Ensuite avec *Bjørn* où j’ai appris un nouveau langage, le *Rust*, avec toute ses subtilités et sa profondeur. 

Selon moi deux projets se dégagent des différentes conclusions faites dans cet article. 

* *Gibica* a besoin de gros changements. Je pense que le plus sympa et logique serait d’en faire un compilateur. 
* *Bjørn* pourrait utiliser un langage intermédiaire pour se rapprocher des performances de *CPython*. Je pense que ce serait intéressant de le faire à ce moment du projet pour ne pas avoir à trop modifier le code plus tard. Même aujourd’hui cela va demander un travail conséquent donc autant le faire tant que le nombre de fonctionnalités n’est pas trop élevé. 

Dans ces deux projets je compte utiliser *LLVM* (Low Level Virtual Machine), que ce soit pour convertir l’AST en un langage intermédiaire ou bien pour interpréter / compiler celui-ci. Ces deux projets seront donc utiles l’un pour l’autre ce qui accélérera peut-être un peu les développements. Encore de beaux challenges en perspective!
