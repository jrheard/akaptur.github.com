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

What's going on here?  In this case, I think it's easiest to work backwards from the end.

Last we have `exec code in Madness()`. The `exec` keyword in python 2 [^1]

[^1]: All of this works in python 3, too, with slightly different syntax because `exec` is a function, not a keyword.  Nick's included the python 3 version in his gist.

