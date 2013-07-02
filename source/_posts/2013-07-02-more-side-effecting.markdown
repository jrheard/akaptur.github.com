---
layout: post
title: "More side effecting"
date: 2013-07-02 15:47
comments: false
categories: 
---

[Nick Coghlan](https://twitter.com/ncoghlan_dev) added another interesting note in response to my last post about creating side-effecting assignments in python.  Here's Nick:

<blockquote class="twitter-tweet" data-conversation="none" data-cards="hidden"><p><a href="https://twitter.com/brandon_rhodes">@brandon_rhodes</a> <a href="https://twitter.com/akaptur">@akaptur</a> You can do better if you&#39;re the one controlling the namespace creation (e.g. an importer): <a href="https://t.co/gA8LD121cD">https://t.co/gA8LD121cD</a></p>&mdash; Nick Coghlan (@ncoghlan_dev) <a href="https://twitter.com/ncoghlan_dev/statuses/350971160659378176">June 29, 2013</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

Let's look at [that gist](https://gist.github.com/ncoghlan/5891123#file-crazy-namespace-python-2):

``` python
>>> class Madness(dict):
...     def __setitem__(self, attr, value):
...         if isinstance(value, type):
...             value.__name__ = attr
...         dict.__setitem__(self, attr, value)
... 
>>> code = """\
... class Example(object): pass
... NewName = Example
... print(Example.__name__)
... """
>>> exec code in Madness()
NewName
```

This is really fun stuff, and nicely illustrates a core feature of python - that namespaces are basically just dictionaries.

So what's going on here?  In this case, I think it's easiest to work backwards from the end.

Last we have `exec code in Madness()`. The `exec` keyword in python 2[^1] executes a string as code.

``` python
>>> a = "hello"
>>> exec "print a"
hello
```

`exec` optionally takes a context in which to execute the code.  A dictionary is a perfectly legal context to use:

``` python
>>> exec "print b" in {'b' : "hi there"}
hi there
```

All the code is shown here - `b` is not defined elsewhere in the REPL session.

If a dictionary is provided, it's presumed to contain both the global and the local variables.  This dictionary is the **only** context in which the code will be executed (it won't check the scope from which the `exec` statement was made).

``` python
>>> a = "hello"
>>> exec "print a" in {'b': "hi there"}
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "<string>", line 1, in <module>
NameError: name 'a' is not defined
```
So without knowing anything about the `Madness()` object, we'd expect it to be a dictionary-like thing.

And sure enough, it is:
``` python
>>> class Madness(dict):
...     def __setitem__(self, attr, value):
```

`__setitem__` is the standard way to override setting a key:value pair in a dictionary.  It works just like you'd expect:
``` python
>>> d = {}
>>> d.__setitem__('hello', 2)
>>> d
{'hello': 2}
```

We're intercepting any attempt to write to the Madness dictionary-like object.
``` python
...     def __setitem__(self, attr, value):
...         if isinstance(value, type):
...             value.__name__ = attr
...         dict.__setitem__(self, attr, value)
```

If the value of the key:value pair that we're setting is an instance of the `type` type - that is, if the object in question is a class - then grab that class's __name__ and set it to the key.  (This step will blow up if the key in question isn't a legal name in the first place.)  Then use the parent class (`dict`)'s `__setitem__` to actually write to the Madness object.

We can execute the code string one line at a time to better see what's happening.

``` python
>>> m = Madness()
>>> m.keys()
[]
>>> exec "class Example(object): pass" in m
>>> m.keys()
['__builtins__', 'Example']
>>> m['Example']
<class 'Example'>
>>> exec "NewName = Example" in m
>>> m.keys()
['__builtins__', 'NewName', 'Example']
>>> m['Example']
<class 'NewName'>
>>> m['NewName']
<class 'NewName'
>>> exec "OtherName = Example" in m
>>> m['Example']
<class 'OtherName'>
```

(I'm using `m.keys()` to examine the state of the namespace instead of just printing `m` because the reference to `__builtins__` pukes a bunch of tiresome definitions and copyright statements into the REPL.  `exec` adds a reference to `__builtins__` on the execution of any statement.)

We're now quite close to the original Ruby behavior - with a deeper understanding of python namespaces!




[^1]: All of this works in python 3, too, with slightly different syntax because `exec` is a function, not a keyword.  Nick's gist includes the python 3 version too.
