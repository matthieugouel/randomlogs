---
title: Présentation de Gibica
description: Au coeur d’un langage
date: "2018-07-24"
featured: "dev.jpg"
featuredalt: "dev"
featuredpath: "img/blog"
linktitle: ""
categories: ["Développement"]
tags: ["Gibica"]
type: post
---

Depuis quelques semaines je m’intéresse beaucoup au fonctionnement des langages informatiques. Les utilisant quotidiennement depuis plusieurs années, j’ai rarement pris le temps d’étudier précisément comment ces langages sont pensés, construits et interprétés par nos machines. Bien sûr j’ai eu des cours de compilation en école d’ingénieur, mais j’ai beaucoup sous-estimé cette matière à l’époque, donc il ne m’en reste pas grand-chose (bouh !). 

<!-- more --> 

Quand je souhaite vraiment apprendre quelque chose sur un sujet, ma technique est de trouver rapidement un objectif de réalisation qui me permette de valider mes compétences (vous remarquerez que j’ai rien inventé). Je me suis donc mis en tête d’écrire un *interpréteur* (ou bien un interprète, mais on va rester sur interpréteur) pour exécuter un langage inventé par mes soins. C’est ainsi que le langage Gibica est né ! 

Quand on parle de langage informatique on fait parfois l’amalgame entre le langage en lui-même et le programme permettant de convertir ce langage en une suite d’instruction compréhensible par la machine. C’est souvent le cas parce que la commande pour exécuter le programme a le même nom que le langage lui-même (exemple Python ou Java). Mais finalement quand on dit « je veux créer un langage », il y a deux choses distinctes à accomplir : définir le langage avec sa syntaxe et sa grammaire, et écrire le programme qui interprète ce langage. 

Le programme qui va exécuter un langage informatique peut être de différente natures. On parle souvent de *compilateur* et/ou d’*interpréteur* qui sont deux types de programmes avec des stratégies différentes.
La stratégie d’un compilateur est de traduire un langage dans un autre plus proche du langage machine. Un compilateur digne de ce nom passe souvent par plusieurs étapes de compilation pour arriver à la fin à un code en langage binaire, formé uniquement de 0 et de 1 et compréhensible par la machine. C’est par exemple le cas de GCC, le compilateur officiel du langage C, qui, après plusieurs étapes de compilation, transforme le C en un fichier binaire. Ce fichier est ensuite exécutable directement sans l’aide du compilateur. 
La stratégie d’un interpréteur est différente : il va lire le langage source et va interpréter celui-ci à la volée pour effectuer des instructions. À la différence du compilateur il ne traduit pas le programme en un fichier binaire mais interprète directement les différentes instructions et fait ce travail de traduction à chaque exécution. On a donc besoin de l’interpréteur à chaque fois que l’on veut exécuter le programme. En guide d’exemple on peut imaginer un interpréteur lisant par exemple du Python et analyse, traduit et exécute les instructions dans un langage de plus bas niveau comme du C.

> Petit aparté. Beaucoup de gens pensent que c’est comme cela que fonctionne l’interpréteur Python sauf que … pas tout à fait. En fait, comme pour beaucoup de langages modernes, l’interpréteur officiel de Python (CPython) va d’abord compiler le Python en un langage intermédiaire puis va interpréter ce dernier via le langage C. Et oui rien n’empêche de compiler un langage en autre chose que du langage machine !

Gibica est donc un langage unique, avec sa syntaxe et sa grammaire. C’est également le nom du programme qui l’interprète. Le mot « Gibica » peut donc aussi bien faire référence au langage qu’à l’interpréteur qui permet à la machine de comprendre ce langage.

Mais à quoi ressemble donc le langage Gibica ? Voici un petit aperçu :

```
#
# This is a Gibica program ! 8-)
#

def process(n) {

    let mut i = 0;

    while i < n {
        if i == 2 {
            return i + 2;
        }
        i = i + 1;
    }
    return 0;
}

let result = process(5)
print(result);

```

Les plus observateurs pourront remarquer mes inspirations qui sont principalement le Python, le Rust et le JavaScript. Et oui, c’est difficile de créer un langage parfaitement original ! :-p

En écrivant ce bout de code dans un fichier `test.gbc`, on peut exécuter celui-ci avec la commande suivante :

```
$ gibica test.gbc
```

Et on obtient bien le résultat attendu : 

```
4 
```


Yeah ! (Ouf !…)

Il y encore beaucoup à dire sur les fonctionnalités du langage, sur comment fonctionne l’interpréteur et sur le chemin qu’il reste à parcourir… mais je réserve cela pour d’autres articles ! Néanmoins, si cela vous intéresse, le code de l’interpréteur est open source et est disponible [ici](https://github.com/matthieugouel/gibica). J’ai également rapidement écrit une documentation du langage [ici](http://gibica.readthedocs.io/en/latest/index.html).

À très vite pour plonger au cœur d’un langage informatique !