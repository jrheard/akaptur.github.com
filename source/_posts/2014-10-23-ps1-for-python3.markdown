---
layout: post
title: "PS1 for Python3"
date: 2014-10-23 14:38
comments: true
categories: python
---

I spend a lot of time flipping back and forth between Python 2.x and 3.x: I use different versions for different projects, talk to people about different versions, explore differences between the two, and paste the output of REPL sessions into chat windows.  I also like to keep long-running REPL sessions.  These two activities in combination became quite confusing, and I'd often forget which version I was using.

``` python
>>> print some_var
  File "<stdin>", line 1
    print some_var
                 ^
SyntaxError: invalid syntax
>>> # *swears*
```

After the hundredth time I made this mistake, I decided to modify my prompt to make it always obvious which version was which, even in long-running REPL sessions.  You can do this by creating a file to be run when Python starts up.  Add this line to your `.bashrc`:

```
export PYTHONSTARTUP=~/mystartupscript.py
```

Then in `mystartupscript.py`:

``` python mystartupscript.py
import sys
if sys.version_info.major == 3:
    sys.ps1 = "PY3 >>> "
    sys.ps2 = "PY3 ... "
```

This makes it obvious when you're about to slip up:

``` python
PY3 >>> for value in giant_collection:
PY3 ...     print(value)
PY3 ...
```

I've also add this line to `mystartupscript.py` to bite the bullet and start using print as a function everywhere:

``` python mystartupscript.py
from __future__ import print_function
```

This has no effect in Python3.x, but will move 2.x to the new syntax.
