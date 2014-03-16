---
layout: post
title: "Reading EBNF"
date: 2014-03-16 15:39
comments: true
categories:
---

One of the fun parts of [pairing with Amy on Nagini](http://mathamy.com/import-accio-bootstrapping-python-grammar.html) was modifying the grammar of Python. You should try this - it's easier than you think! Eli Bendersky has a [great post](eli.thegreenplace.net/2010/06/30/python-internals-adding-a-new-statement-to-python/) with step-by-step instructions.[^1]

Modifying Python's grammar starts in the Grammar/Grammar file. I've recently learned how to read this (which, for me, mostly means learning how to pronounce the punctuation), so I want to walk through the `import` example in some detail. The syntax here is [Extended Backus-Naur Form](http://en.wikipedia.org/wiki/Extended_Backus%E2%80%93Naur_Form), or EBNF. You read it like a tree, and your primary verb is "consists of":

* `import_stmt` consists of one of two forms, `import_name` or `import_from`.
* `import_name` consists of the literal word `import` followed by `dotted_as_names`.
* `dotted_as_names` consists of a `dotted_as_name` (note the singular), optionally followed by one or more pairs of a comma and another `dotted_as_name`.
* `dotted_as_name` consists of a `dotted_name`, optionally followed by the literal word 'as' and a NAME.
* Finally, `dotted_name` consists of a `NAME`, maybe followed by pairs of a dot and another `NAME`.

You can walk the other branches in a similar way.

```
import_stmt: import_name | import_from
import_name: 'import' dotted_as_names
# note below: the ('.' | '...') is necessary because '...' is tokenized as ELLIPSIS
import_from: ('from' (('.' | '...')* dotted_name | ('.' | '...')+)
              'import' ('*' | '(' import_as_names ')' | import_as_names))
import_as_name: NAME ['as' NAME]
dotted_as_name: dotted_name ['as' NAME]
import_as_names: import_as_name (',' import_as_name)* [',']
dotted_as_names: dotted_as_name (',' dotted_as_name)*
dotted_name: NAME ('.' NAME)*
```

To `accio`-ify Python, we had to replace the occurences of `'import'` with `'accio'`. There are only two - we were only interested in the literal string `import`, not all the other names. `import_as_name` and so on are just nodes in the tree, and only matter to the parser and compiler.

Every other keyword and symbol that has special meaning to the Python parser also appears in Grammar as a string.

Perusing the grammar is (goofy) way to learn about corner cases of Python syntax, too!  For example, did you know that `with` can take more than one context manager? It's right there in the grammar:

```
with_stmt: 'with' with_item (',' with_item)*  ':' suite
with_item: test ['as' expr]
```

``` python
>>> with open('foo.txt','w') as f, open('bar.txt','w') as g:
...     f.write('foo')
...     g.write('bar')
...
>>> f
<closed file 'foo.txt', mode 'w' at 0x10bf71270>
>>> g
<closed file 'bar.txt', mode 'w' at 0x10bf71810>
```

Now go ahead and add your favorite keyword into Python!

[^1]: Like Eli, I'm not advocating for Python's actual grammar to change - it's just a fun exercise.
