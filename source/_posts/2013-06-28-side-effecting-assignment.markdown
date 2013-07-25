---
layout: post
title: "Side-effecting Assignment"
date: 2013-06-28 14:35
comments: false
categories: python
---

##Surprises

My colleague <a href="https://github.com/davidbalbert">Dave</a> told me about an interesting bit of Ruby the other day.  If you make a new `Class` object, it initially has no name. If you then assign a name to it, with your new class on the right-hand side of the assignment, the name is attached to the Class object.  

``` ruby
[1] pry(main)> c = Class.new
=> #<Class:0x007ff19bc46988>
[3] pry(main)> c.name
=> nil
[4] pry(main)> Foo = c
=> Foo
[5] pry(main)> Foo
=> Foo
[6] pry(main)> c
=> Foo
[7] pry(main)> c.name
=> "Foo"
```

I found this shockingly non-intuitive. Really, an assignment statement permanently modified the object on the right-hand side?

In a word, yes.

``` ruby
[8] pry(main)> Bar = c
=> Foo
[9] pry(main)> c
=> Foo
[10] pry(main)> Foo
=> Foo
[11] pry(main)> Bar
=> Foo

[12] pry(main)> Baz = Foo
=> Foo
[13] pry(main)> Baz
=> Foo
```

To recap:

Step 1: `c` is some object

Step 2: set `Foo = c`

Step 3: `c` is permanently altered.  

Wow.

Luckily for our intuition, this is a pretty special case, and not a way that Ruby generally behaves.  But it got me wondering if there's something similar in python.

##What about python?

The simplest answer is "no".  In python, [assignment is a simple statement](http://docs.python.org/2/reference/simple_stmts.html#assignment-statements), not an operator, so you can't do things like operator overloading.  This also means we can't somehow add a hook to assignment.

Ok, fine.  But wait - we're programmers!  We don't give up that easily.  

What about hooking on `__getattribute__`?  (Defining your own `__getattribute__` is "the bad way" to do it.  You almost always want to define your own `__getattr__`, unless you're doing something silly and/or malicious, like we are.  Note that we have to use the parent class (`object`)'s `__getattribute__` to extract and save the attribute we want.  If we didn't do that, we'd trigger an infinitely recursive lookup.)

``` python
class Foo(object):
    def __init__(self):
        self.some_attr = "sup foo"

    def __getattribute__(self, attr):
        attribute = object.__getattribute__(self, attr)
        object.__setattr__(self, attr, "you're too late!")
        return attribute
```

This looks like it works!

``` python
>>> f = Foo()
>>> f
<obj.Foo object at 0x10253dd10>
>>> word = f.some_attr
>>> print word
sup foo
>>> print f.some_attr
you're too late!
```

Unfortunately, this behavior has nothing to do with assignment.  The object was mutated when `__getattribute__` was called, which happened to be in an assignment statement in this code above.  We'd get exactly the same behavior without the assignment: 

``` python
>>> g = Foo()
>>> g
<obj.Foo object at 0x10253dd10>
>>> g.some_attr
sup foo
>>> g.some_attr
you're too late!
```

Darn.

But we're programmers, right?  We don't give up that easily.

## Trace functions to the rescue

There's a `settrace` function in python's `sys` module that we can use to examine stack frames, events, and lots of other code data while our program is running.  `sys.settrace(fn)` takes a trace function as an argument, and that trace function must take `frame, event, args` as arguments.  It will then get called every time an event happens.  A line of code, a function call, a function return, and an exception are all "events".  We can use this to inspect each line of code before it runs and check manually if it's an assignment statement. 

(I know this is silly, but isn't it fun?)

``` python
import linecache

class Tracer(object):
    def __init__(self, program):
        self.program = program

    def traceit(self, frame, event, arg):
        if event == 'line':
            linenum = frame.f_lineno
            linetext = linecache.getline(self.program, linenum)
            vprint( 'line', linenum, linetext )

            is_assignment = "=" in linetext and "==" not in linetext

            if is_assignment:
                self.mess_up_on_assignment(frame, linetext)

        return self.traceit
```

(The trace function returns a reference to itself to indicate that tracing should continue.)

Now let's write our code to mess up assignment.

``` python
class Tracer(object):
    [...]

    def mess_up_on_assignment(self, frame, linetext):
        rhs = linetext[linetext.index("=")+1:]
        names = rhs.split()
        local_objs = [name for name in names if name in frame.f_locals.keys() + frame.f_globals.keys()]
        for name in local_objs:
            if name not in JANKY_NAMESPACE_MANAGER:
                box = Box(frame.f_locals[name], name)
                frame.f_locals[name] = box
            else:
                JANKY_NAMESPACE_MANAGER[name].assignments += 1

JANKY_NAMESPACE_MANAGER = {}

class Box(object):
    def __init__(self, obj, name):
        self.obj = obj
        self.name = name
        self.assignments = 1
        JANKY_NAMESPACE_MANAGER[name] = self

    def __repr__(self):
        raise NameError("this message has self-destructed.")
```

Here our "box" object is pretty silly.  It takes the original object and wraps it in a box.  (You could implement `__getattr__` and `__setattr__` so that the box behaves basically like the original object, which I haven't done.)  The box then explodes when you try to print it.  We're also tracking which names we've seen with the aptly-named `JANKY_NAMESPACE_MANAGER` (which has all kinds of scope issues, but whatever).

Meanwhile, in the `mess_up_on_assignment` code, we check to see if any of the whitespace-separated words on the right-hand side of the equation are names we recognize from `frame.f_locals` or `frame.f_globals`.  If so, and if we haven't seen them before, throw them in a self-destructing box!

Add a simple helper to get things set up.

``` python
def assignment(orig_file):
    trace_obj = Tracer(orig_file)
    sys.settrace(trace_obj.traceit)
```

We can now sabotage simple-looking programs!

This one runs fine:
``` python
import sabotage
sabotage.assignment(__file__)

def greet():
    name = raw_input("Enter your name: ")
    print "hello,", name
    
    print "goodbye, ", name

if __name__ == '__main__':
    greet()
```

And this one blows up:
``` python
import sabotage
sabotage.assignment(__file__)

print "done sabotaging"
raw_input("Hit enter to continue")

def greet():
    name = raw_input("Enter your name: ")
    print "hello,", name
    
    name_copy = name
    
    print "goodbye, ", name

if __name__ == '__main__':
    greet()
```

And there you have it - a (incredibly goofy) side-effecting assignment that mutates the object on the right-hand side of the assignment statement.
