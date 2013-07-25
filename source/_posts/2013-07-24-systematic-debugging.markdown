---
layout: post
title: "Systematic Debugging"
date: 2013-07-24 13:54
comments: true
categories: python
---

We've asked many of our residents at Hacker School what qualities all great programmers share.  There's very little agreement - clearly, there are a multitude of ways to be a great programmer, and you can think of a counter-example for almost every quality you can name.  One of the rare non-controversial statements came first from Jessica McKellar, who identified systematic debugging as a key skill.

What does systematic debugging look like?  At this point, I'm focused on two aspects: asking a clear question, and keeping track of my mental "stack".  I had a particularly fun and interesting bug yesterday that I think illustrates this nicely.

The problem at hand was a brainteaser from Jessica: 
>Using the official SOWPODS Scrabble dictionary, what letters, if any, never appear doubled? (By that I mean -- "AA" does appear doubled because it is in "AARDVARK", "BB" does appear doubled because it is in "BUBBLE" -- are there any letters that never appear doubled in a word?)

I came up with a solution, but it was kind of slow, so I was trying to find a faster one.  Also, my original solution used a regular expression to match a doubled letter in a word, but it would only find the first pair (e.g. in BOOKKEEPER only the 'OO' would be caught).

I decided to try taking the dictionary as a single string, rather than a list of words, and consuming it one letter-pair at a time, as follows.

``` python
def find_all_doubles_as_word_mash():
    words = open('sowpods.txt').read()
    letters = set(string.uppercase)
    dub_re = re.compile(r"([A-Z])(\1)")
    end = 0
    dubbed = re.search(dub_re, words[end:])

    while dubbed and letters:
        letters -= set(dubbed.group())
        end += dubbed.end()
        dubbed = re.search(dub_re, words[end:])
        print end, '/', len(words)

    return letters
```

If you're thinking, "Hmm, that looks like it might be slow," congratulations!  It is indeed quite slow.  I had two guesses for why it was slow. (It wasn't the print statement.)  First, maybe I was copying the string over and over, and that was slowing down the function.  Second, maybe I misunderstodd the regular expression - for example, maybe it was taking a longer time to operate on a longer string. I decided to investigate the second possibility first, since regular expressions were more of a mystery to me.

At this point in the problem solving, my mental stack looks something like this:

<pre>
 ---------------------------
| regex taking longer on a  | (newest)
| longer-length string?     |
 ---------------------------
| reg exp? or string copy?  |
 ---------------------------
| function is so slow, why? |
 ---------------------------
|   find a faster solution  |
 ---------------------------
|  solve wordplay problem   | (oldest)
 ---------------------------
</pre>

The next step was to test my theory that my regular expression should not take longer on a longer string.  Importantly, the regex had no "greedy" elements - it should match and return the first time it encounters two identical characters, regardless of how long the string is after those matching characters.  Here was the experiment I wrote:

``` python test_with_timeit.py
import timeit

SETUP = """import re;
import string;
double = re.compile(r"([a-z])(\1)");
short_test = "hee" + string.lowercase;
long_test = "hee" + string.lowercase*100
"""

N_TIMES = 10000

def test():
    short_time = timeit.timeit(stmt="double.search(short_test)", setup=SETUP, number=N_TIMES)
    long_time = timeit.timeit(stmt="double.search(long_test)", setup=SETUP, number=N_TIMES)
 
    print "short:", short_time
    print "long:", long_time
    print "time ratios:", long_time / short_time, "x"

if __name__ == '__main__':
    test()
```
In both cases, I thought, the regex should check "he", fail, check "ee", succeed, and return.  To my great surprise, the long string test took 70-80x as long to run as the short string test!  
``` text test_with_timeit output
short: 0.0243809223175
long: 1.77788496017
time ratios: 72.9211527366 x
```

Clearly, I was missing something about regular expressions.  I googled around and learned some details about backreferences, the `(\1)` in my code (which makes my regular expression not actually ["regular"](http://en.wikipedia.org/wiki/Regular_expression#Formal_language_theory)), but that wasn't very illuminating.  (You can picture "something about Finite State Automata" being pushed onto and quickly popped back off the stack.)

My next step was to run timing experiments with a variety of regular expressions to see if I could find an element that made the difference.  At this point yesterday, I was still trying to pass strings to `timeit`, which was getting unwieldy.  (Stay tuned to learn two better ways to do timing.)  I switched to a simpler timing script, just taking `time.time()` at the start and end of the algorithm.

``` python test_without_timeit.py
import re
import string
import time
from collections import defaultdict

everybody_stand_back = {
    "double" : re.compile(r"([a-z])(\1)"),
    "double_var" : re.compile(r"([a-z])\1"),
    "single" : re.compile(r"([a-z])"),
    "verbose" : re.compile("|".join(let*2 for let in string.lowercase)),
}

test_strings = {
    "short" : "hee",
    "long" : "hee" + string.lowercase*100,
}

outcomes = defaultdict(dict)

N_TIMES = 100000

def test():
    for name, regex in everybody_stand_back.iteritems():
        for string_name, test_string in test_strings.iteritems():
            start = time.time()
            for _ in range(N_TIMES):
                re.search(regex, test_string)
            end = time.time()
            delta = end - start
            outcomes[name][string_name] = delta

    return outcomes

if __name__ == '__main__':
    outcomes = test()
    for name, regex in outcomes.iteritems():
        print name, regex["long"]/regex["short"]
``` 

Hopefully the differences would start to become clear, and I could easily extend this to add more variations on the regular expression.

It printed:
```
double 0.953508019384
single 1.15989795352
double_var 1.04188410251
verbose 0.861312402506
```

Uh-oh, wait a minute, I thought this was identical code.  Where did the 70x time difference go?

I couldn't see any obvious errors, so I went back to the `timeit` code.  Could I reproduce these results while still using the `timeit` module?

Let's look back at the problem stack.

<pre>
 ---------------------------
| what's up with timeit?    |    
 ---------------------------
| is this code different?   |    
 ---------------------------
| timing programs give      |
| different results, uh-oh  |
 ---------------------------
| regex taking longer on a  |
| longer-length string      |
 ---------------------------
| reg exp? or string copy?  |
 ---------------------------
| function is so slow, why? |
 ---------------------------
|   find a faster solution  |
 ---------------------------
|  solve wordplay problem   | (oldest)
 ---------------------------
</pre>

At this point [Brandon Rhodes](https://twitter.com/brandon_rhodes) joined me in pair-debugging, and we made a couple of discoveries.  The most interesting of these was that two different ways of calling timeit generated wildly different results. 

``` python test_with_timeit.py
import re
import timeit
import string
 
 
double = re.compile(r"([a-z])(\1)")

 
SETUP = """import re;
import string;
double = re.compile(r"([a-z])(\1)");
short_test = "hee" + string.lowercase;
long_test = "hee" + string.lowercase*200
"""

N_TIMES = 10000

short_test = "hee" + string.lowercase
long_test = "hee" + string.lowercase*200

def test():
    short_time = timeit.timeit(stmt="double.search(short_test)", setup=SETUP, number=N_TIMES)
    long_time = timeit.timeit(stmt="double.search(long_test)", setup=SETUP, number=N_TIMES)
 
    print "short:", short_time
    print "long:", long_time
 

def short_string_test():
    double.search(short_test)

def long_string_test():
    double.search(long_test)
 
if __name__ == '__main__':
    print "method 1: strings"
    test()

    print "method 2: functions"
    print timeit.timeit(short_string_test, number=N_TIMES)
    print timeit.timeit(long_string_test, number=N_TIMES)
    
    print "method 3: build timer explicitly"
    s = timeit.Timer(stmt="double.search(long_test)", setup=SETUP)
    print "short:", s.timeit(number=N_TIMES)
    t = timeit.Timer(stmt="double.search(long_test)", setup=SETUP)
    print "long:", t.timeit(number=N_TIMES)
```

``` python test_with_timeit output
method 1: strings
short: 0.0235750675201
long: 3.41090488434
method 2: functions
0.0102179050446
0.00959801673889
method 3: build timer explicitly
short: 0.0263011455536
long: 3.42960190773
```
So now we know something is up with passing strings to timeit.  

<pre>
 ---------------------------
| timeit string version - ? |        
 ---------------------------
| what's up with timeit?    |    
 ---------------------------
| is this code different?   |    
 ---------------------------
          ....
</pre>

Ok, we've narrowed it down this far - it's time to take a look at the source.  `timeit` is a module written in python (yay!), and the relevant parts look pretty straightforward:

``` python timeit.py
template = """
def inner(_it, _timer):
    %(setup)s
    _t0 = _timer()
    for _i in _it:
        %(stmt)s
    _t1 = _timer()
    return _t1 - _t0
"""

class Timer:
    """Class for timing execution speed of small code snippets.

    The constructor takes a statement to be timed, an additional
    statement used for setup, and a timer function.  Both statements
    default to 'pass'; the timer function is platform-dependent (see
    module doc string).

    To measure the execution time of the first statement, use the
    timeit() method.  The repeat() method is a convenience to call
    timeit() multiple times and return a list of results.

    The statements may contain newlines, as long as they don't contain
    multi-line string literals.
    """

    def __init__(self, stmt="pass", setup="pass", timer=default_timer):
        """Constructor.  See class doc string."""
        self.timer = timer
        ns = {}
        if isinstance(stmt, basestring):
            stmt = reindent(stmt, 8)
            if isinstance(setup, basestring):
                setup = reindent(setup, 4)
                src = template % {'stmt': stmt, 'setup': setup}
            elif hasattr(setup, '__call__'):
                src = template % {'stmt': stmt, 'setup': '_setup()'}
                ns['_setup'] = setup
            else:
                raise ValueError("setup is neither a string nor callable")
            self.src = src # Save for traceback display
            code = compile(src, dummy_src_name, "exec")
            exec code in globals(), ns
            self.inner = ns["inner"]
        elif hasattr(stmt, '__call__'):
            self.src = None
            if isinstance(setup, basestring):
                _setup = setup
                def setup():
                    exec _setup in globals(), ns
            elif not hasattr(setup, '__call__'):
                raise ValueError("setup is neither a string nor callable")
            self.inner = _template_func(setup, stmt)
        else:
            raise ValueError("stmt is neither a string nor callable")

    def timeit(self, number=default_number):
        """Time 'number' executions of the main statement.

        To be precise, this executes the setup statement once, and
        then returns the time it takes to execute the main statement
        a number of times, as a float measured in seconds.  The
        argument is the number of times through the loop, defaulting
        to one million.  The main statement, the setup statement and
        the timer function to be used are passed to the constructor.
        """
        if itertools:
            it = itertools.repeat(None, number)
        else:
            it = [None] * number
        gcold = gc.isenabled()
        gc.disable()
        timing = self.inner(it, self.timer)
        if gcold:
            gc.enable()
        return timing

def timeit(stmt="pass", setup="pass", timer=default_timer,
           number=default_number):
    """Convenience function to create Timer object and call timeit method."""
    return Timer(stmt, setup, timer).timeit(number)
```

Let's trace the part that's giving us the long execution string.  We call the module-level function `timeit.timeit`, which creates a Timer object and returns the result of calling its `timeit` method.  No surprise that we got identical results between our method #1 and method #3 above.  

The `Timer` object checks to see if you've passed it strings or callables as its `stmt` and `setup` parameters.  In our case both are strings.  The `init` method splices the strings into the source code here:

```
src = template % {'stmt': stmt, 'setup': setup}
```

We can isolate this in the REPL if we want to be extra-sure of how it works:
```
>>> template = """
... def inner(_it, _timer):
...     %(setup)s
...     _t0 = _timer()
...     for _i in _it:
...         %(stmt)s
...     _t1 = _timer()
...     return _t1 - _t0
... """
>>> stmt = "pass"
>>> setup = "print 'here we go'"
>>> print template % {'stmt': stmt, 'setup': setup}

def inner(_it, _timer):
    print 'here we go'
    _t0 = _timer()
    for _i in _it:
        pass
    _t1 = _timer()
    return _t1 - _t0
```

Got it - simple, legal python code is generated. So where is our regular expression getting screwed up?  Let's take a look at the code generated in our actual test by inserting a print statement into the `timeit` module.  (First we'll copy the module over to our current working directory, to avoid modifying our actual standard library.)

```
src = template % {'stmt': stmt, 'setup': setup}
print src
```

And now calling `test_with_timeit.py` again, it prints:

``` python 
def inner(_it, _timer):
    import re;
    import string;
    double = re.compile(r"([a-z])()");
    short_test = "hee" + string.lowercase;
    long_test = "hee" + string.lowercase*200
    
    _t0 = _timer()
    for _i in _it:
        double.search(short_test)
    _t1 = _timer()
    return _t1 - _t0
```

This looks mostly reasonable ... but hey, wait, what happened to the regular expression?

``` python
double = re.compile(r"([a-z])()");
```

That's not right!  Changing `print src` to `print repr(src)` gives us a better idea what's up - the relevant line is now:

``` python
double = re.compile(r"([a-z])(\x01)");
```

And looking back at the `SETUP` string ... uh-oh, this isn't a raw string, and it contains an escape character (`(\1)`).  We can tell we're in trouble immediately in the REPL:
``` python
>>> print "\1"

>>> repr("\1")
"'\\x01'"
```

Why does this happen?  We can consult the [docs on string literals](http://docs.python.org/2/reference/lexical_analysis.html#string-literals) to find out.  Unless you're working with a raw string (one prefixed with `r` or `R`), escape characters are interpreted the same way they are in C.  Our text, a backslash followed by a number, matches the pattern for an octal digit: a backslash followed by 1-3 integers. `\ooo` indicates a character with the octal value `ooo`.  And that's exactly what we've done:

``` python
>>> ord("\1")
1
>>> chr(1)
'\x01'
```

`\x01` is a legal (though unprintable) character, and **it doesn't appear in our long string to test the regular expression.** So _of course_ the longer one took a longer time - it had to traverse the entire string looking for `\x01`, which it never found.

The fix is quite easy - one character, in fact.  Make the `SETUP` string raw:
``` python 
SETUP = r"""import re;
import string;
double = re.compile(r"([a-z])(\1)");
short_test = "hee" + string.lowercase;
long_test = "hee" + string.lowercase*200
"""
```

... and the problem is solved.  So we can pop a bunch of frame off our problem stack and go investigate the string copying, which is in fact the issue.

<pre>
 ---------------------------
| reg exp? or string copy?  |
 ---------------------------
| function is so slow, why? |
 ---------------------------
|   find a faster solution  |
 ---------------------------
|  solve wordplay problem   | (oldest)
 ---------------------------
</pre>

## Lessons learned
Interesting bugs are a ton of fun!  Now for a bit of introspection: what could have gone better?

- Most obviously, I could have spent less time on the regular-expression branch of why the function was slow, especially as the stack grew.  Whether I should have done this depends on your perspective: at Hacker School, I'm almost 100% learning motivated and about 0% get-it-done motivated, so my time to solve the bug is virtually unbounded.

- I was slightly overconfident about understanding how the `timeit` code splicing happened.  The code looked so simple that I didn't bother to check my understanding at first.

- Passing `timeit.timeit` strings is an unwieldy way of doing timing.  I should have been using either the callable option, or just timing with `time python test_file.py` from the command line.  Both of these would have been easier to work with and would have sidestepped the bug.

- My mental model of regular expressions was largely correct.  That's satisfying.

- Strings!  Ouch.  This may not seem like much of a lesson, but it's useful to add to my mental list of Places That Bugs Lurk.

