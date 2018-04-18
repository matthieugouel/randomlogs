---
title: Be careful with mutables.
description: Strange things may happen.
date: "2018-04-18"
featured: "banner_blog1.png"
featuredalt: "Init"
featuredpath: "date"
linktitle: ""
categories: ["Development"]
tags: ["Python"]
type: post
---

<!--more-->

# Mutable vs immutable

In Python there is two type of objects : mutable and immutable.

A mutable object can be modified after its initialization.
You can change the content of theses objects without changing their identity.

For instance, the `int` objects are mutable.

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

You may notice that the hash doesn't differ before and after changing the content of the object (0x10fc01ad0).

On the other hand, immutables are objects that cannot be changed after their initialization.

For instance, `float` objects are immutables.

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
This time you may notice that the hash changed because a new object has been created when we made the following command.

```python
In [3]: float += 2.0
```

People often think that we cannot use operators (like `+`) with immutables but it's not true.
It's just that a new object will be created out of this.

Ok, so what is the advantage of using immutables object ? It seems to be just a way to complicate things right ?  
The mutable / immutable concept has to do with functional programming.
I will get into this another time but in functional programming every object is immutable.
The main reason to do that is to avoid side effects dealing with mutables.

# An example of mutables side effects

Let's try to better understand the problem with mutables by example.  
Imagine you want to write a piece of code that create a list with some dictionaries which share the same base.
For example, only the value of one key differs from an object to another.

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

Oh ? It seems like the first item has been overrided when we added the second one.  
In fact what we see in the list is two _references_ of the same object because `dict` objects are mutables.
So when we changed the object _buffer_ the second time, we also changed the first reference of the object.

A solution would be to create a _buffer_ object each time we get in the loop.
Hopefully `dict` provide a way to copy the content of a dictionary to an other with the `copy()` method.

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

Ok it's better ! Now for each loop, a new buffer object is created.

Finally, we can think about a _pythonic_ way to do the same thing using a list comprehension and a little trick to concatenate dicts (Python 3.5+).

```python
base = {'index': 0, 'anything': 'else'}
container = [{ **base, **{'index': i}} for i in range(2)]
```
```
[{'anything': 'else', 'index': 0}, {'anything': 'else', 'index': 1}]
```

But actually this method is maybe less explicit than the previous for someone reviewing your code.

# Conclusion

It's important to know the difference between mutable and immutable object in order to avoid theses type of side effects.
For performance and simplicity, I think using mutables are very convenient and must stay in the Python language, but using immutables in some part of your code may be a _best practice_ to prevent strange things to happen.
