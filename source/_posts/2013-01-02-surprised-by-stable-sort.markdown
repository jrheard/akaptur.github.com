---
layout: post
title: "Surprised by Stable Sort"
date: 2013-01-02 00:02
comments: false
categories: python
---

I encountered some surprising behavior in python's list.sort() method.  I was calling list.sort() with a function, then list.reverse().  This was silly - I forgot I could call list.sort(reverse=True).  As it turns out, these are not the always the same, or even usually the same.  

Here's the code:

```
def two_sort(ex):
    ex_copy = ex[:]
 
    ex.sort(key=lambda tup: tup[1])
    ex.reverse()
    print "Ex:  ", ex
 
    ex_copy.sort(key=lambda tup: tup[1], reverse=True)
    print "Copy:", ex_copy
 
    assert ex == ex_copy # fails
 
ex = [('a', 0), ('b', 0), ('c', 2), ('d', 3)]
two_sort(ex)
# Ex:   [('d', 3), ('c', 2), ('b', 0), ('a', 0)]
# Copy: [('d', 3), ('c', 2), ('a', 0), ('b', 0)]
```

What's happening here is that .sort() and sorted() in python are "stable sorts", meaning that it'll preserve the existing order when faced with a tie.  This leads to a nice side effect, as described in the [python wiki](http://wiki.python.org/moin/HowTo/Sorting/): you can sort by multiple criteria.  So if we wanted to sort primarily by number (ascending), and secondarily by letter, we could do that quite easily.  Just sort by the secondary key first, then by the primary key.  Since the sort is stable, you know that a tie in the second (primary) sort will preserve the order you had going in.  

```
>>> a = [('d', 3), ('c', 5), ('b', 0), ('a', 0)]
>>> a.sort(key=lambda tup: tup[0])
>>> a
[('a', 0), ('b', 0), ('c', 5), ('d', 3)]
>>> a.sort(key=lambda tup:tup[1])
>>> a
[('a', 0), ('b', 0), ('d', 3), ('c', 5)]
```