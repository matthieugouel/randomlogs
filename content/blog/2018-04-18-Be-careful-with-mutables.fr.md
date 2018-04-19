---
title: Les mutables sont fourbes
description: Vous pourriez être surpris.
date: "2018-04-18"
featured: "dev.png"
featuredalt: "dev"
featuredpath: "img/blog"
linktitle: ""
categories: ["Développement"]
tags: ["Python"]
type: post
---

En Python il y a deux types d'objets :les mutables et les immutables. Pourquoi pas un seul ? Parce que les deux sont bien pratiques selon la situation et permettent d'aborder le code de façon différentes. Voyons tout ça plus en détail.

<!--more-->

# Mutable vs immutable

Un mutable peut être modifié après son initialisation.
Autrement dit, vous pouvez changer le contenu de cet objet sans changer son identité.

Par exemple, les objets `int` sont mutables.

```python
In [1]: integer = 2

In [2]: integer.__hash__
Out[2]: <method-wrapper '__hash__' of int object at 0x10fc01a90>

In [3]: integer += 2

In [4]: integer
Out[4]: 4

In [5]: integer.__hash__
Out[5]: <method-wrapper '__hash__' of int object at 0x10fc01ad0>
```

Vous pouvez remarquer que le hash ne diffère pas avant et après avoir changé la valeur de l'objet (0x10fc01ad0).


De l'autre côté du ring on a les immutables. Ceux-ci, une fois instanciés, ne peuvent plus être modifiés.

Par exemple, les objets `float` sont immutables.

```python
In [1]: float = 2.0

In [2]: float.__hash__
Out[2]: <method-wrapper '__hash__' of float object at 0x104d7ed08>

In [3]: float += 2.0

In [4]: float
Out[4]: 4.0

In [5]: float.__hash__
Out[5]: <method-wrapper '__hash__' of float object at 0x104d7ee88>
```

Cette fois-ci vous pouvez remarquer que le hash a changé. Et oui, la modification de l'objet à la ligne 3 a recréé un nouvel objet `float` avec la valeur actualisée.

```python
In [3]: float += 2.0
```

Les gens pensent souvent à tort, que l'on ne peut pas utiliser d'opérateurs avec les immutables. On a vu ici que c'était tout à fait possible, c'est juste que Python va recréer un objet tout frais à chaque fois.

A ce moment on peut se demander quel peut bien être l'avantage d'avoir implémenté certains objets en mode immutables en python. Pourquoi ne pas avoir tout mis mutable et terminé bonsoir ?

En fait le concept d'immutabilité à quelque chose à voir avec la programmation fonctionnelle. Je pense que j'en reparlerais plus en détail une autre fois mais en clair le but des immutables est d'éviter les effets de bord.
Quels effets me direz-vous ? Ça tombe bien j'ai un petit exemple sous la main qui m'a déjà causé quelques temps de debug.

# Un exemple d'effet de bord des mutables

Alors imaginons que l'on souhaite écrire un bout de code qui crée une liste de dictionnaires  partageant la même base. Ici, seul la valeur d'une seul clé diffère d'un dictionnaire à l'autre.

```python
# Incorrect way
container = []
base = {'index': 0, 'anything': 'else'}
for i in range(2):
    buffer = base
    buffer['index'] = i
    container.append(buffer)
```

```
[{'anything': 'else', 'index': 1}, {'anything': 'else', 'index': 1}]
```

Ah ? Bon ok ça ne fonctionne pas. On dirait que le second dictionnaire à remplacé le premier à la seconde itération.   
En fait ce que l'on voit est deux _références_ à un même objet car les  `dict` sont mutables. Les deux dictionnaires ont le même hash.
Du coup quand on change la valeur de _buffer_ à la seconde itération, on change aussi la première référence, qui est le premier dictionnaire.

Une solution est de créer un nouvel objet _buffer_ à chaque itération.
Petit truc pratique,  `dict` permet de créer un nouveau dictionnaire avec les mêmes valeurs grâce à la méthode `copy()`.

```python
# Correct way
container = []
base = {'index': 0, 'anything': 'else'}
for i in range(2):
    buffer = base.copy()
    buffer['index'] = i
    container.append(buffer)
```
```
[{'anything': 'else', 'index': 0}, {'anything': 'else', 'index': 1}]
```

C'est mieux ! A chaque itération un nouvel objet _buffer_ est créé indépendant de celui d'avant.

Pour finir, une façon _pythonic_ de faire la même chose en utilisant une compréhension de liste et une astuce pour concaténer deux dictionnaires (Python 3.5+).

```python
base = {'index': 0, 'anything': 'else'}
container = [{ **base, **{'index': i}} for i in range(2)]
```
```
[{'anything': 'else', 'index': 0}, {'anything': 'else', 'index': 1}]
```

Mais bon cette méthode, même si je la trouve stylé, est tout de même moins explicite que la première, donc à éviter si vous ne voulez pas perdre votre lecteur.

# Conclusion

C'est important de faire la différence entre mutables et immutables notamment pour éviter ce genre d'effet de bord.
Pour des questions de performances et de simplicité, les mutables restent très pratiques et doivent à mon sens rester dans Python. Par contre, choisir d'utiliser des immutables dans certaines situations constitue une bonne pratique afin d'éviter d'être surpris.
