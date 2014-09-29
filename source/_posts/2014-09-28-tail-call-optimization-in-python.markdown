---
layout: post
title: "Tail-call optimization in Python by hacking bytecode"
date: 2014-09-28 17:53
comments: false
categories:
---

### Tail-call recursion
Tail-call recursion refers to a type of recursion where no information needs to be stored on the call stack to reach the correct answer.

This recursive function is not tail-recursive, because the multiplication happens as the call stack unwinds:

``` python
def factorial(n):
    if n < 2:
        return 1
    else:
        return n * factorial(n-1)
```

We could rewrite the function to be tail-recursive as follows:

``` python
def factorial_tail(n, accumulator = 1):
    if n < 2:
        return accumulator
    else:
        return factorial_tail(n-1, accumulator * n)
```

In some programming languages, the compiler notices functions that are tail-recursive and rewrites them to avoid actually building up `n` stack frames while recursing.  This is known as tail-call optimization, or TCO.  Python doesn't do TCO, so both versions of this factorial calculator will give us trouble for values of `n` greater than Python's recursion limit (1000 by default):

``` python
>>> factorial(1000)
  File "<stdin>", line 5, in factorial
  File "<stdin>", line 5, in factorial
  ...
  File "<stdin>", line 5, in factorial
  File "<stdin>", line 5, in factorial
RuntimeError: maximum recursion depth exceeded
```

#### Brief politcal side note
Python doesn't have TCO for non-technical reasons - it would be possible, but core developers have chosen not to implement it.  I'm fine with that decision. This fun implementation of TCO isn't an argument that the language should have TCO, and it certainly isn't how it would be done if Python chose to really implement it.

### Implementing tail-call optimization in Python

[Paul Tagliamonte](https://twitter.com/paultag), [Luida Nikolaeva](https://github.com/lohmataja), and I sat down to implement this a few weeks ago and had it Totally Working (tm) a few hours later. Here's the basic strategy:

1. Walk through the bytecode looking for recursive calls
2. When you find one, remove it.
3. Store the arguments to the recursive call to overwrite the original arguments.
4. Replace the instruction to make the recursive call with an instruction to jump back to the beginning of the bytecode.

Let's go through this in detail. Here's an example of where we're trying to get:

``` python
@make_tail_recursive
def fact(n, accum):
    if n <= 1:
        return accum
    else:
        return fact(n-1, accum*n)

fact(1000, 1) # will totally work
```

First, we implement the decorator piece. We take a function, rewrite its bytecode, use the new bytecode to build a new code object, and then glue the new code object back on the original function object. We have to make a new code object because the code object's attributes are read-only. Surprisingly, it is possible to just replace a function's code object!

``` python
from types import CodeType

def make_tail_recursive(fn):
    c = fn.__code__
    bytecode = tail_recurse(fn)
    new_code = CodeType(c.co_argcount, c.co_nlocals, c.co_stacksize, c.co_flags,
                        bytecode, # where the magic happens
                        c.co_consts, c.co_names, c.co_varnames, c.co_filename,
                        c.co_name, c.co_firstlineno, c.co_lnotab)
    fn.__code__ = new_code
    return fn
```

Before we dive into `tail_recurse`, let's examine the bytecode of the function we're monkeying with. You can see a visualization of the Python virtual machine's stack [here](link).

``` python
>>> def fact(n, accum):
...     if n <= 1:
...         return accum
...     else:
...         return fact(n-1, accum*n)
...
>>> dis.dis(fact)
  2           0 LOAD_FAST                0 (n)
              3 LOAD_CONST               1 (1)
              6 COMPARE_OP               1 (<=)
              9 POP_JUMP_IF_FALSE       16

  3          12 LOAD_FAST                1 (accum)
             15 RETURN_VALUE

  5     >>   16 LOAD_GLOBAL              0 (fact) <-- loads the `fact` function
             19 LOAD_FAST                0 (n)    <-- loads n
             22 LOAD_CONST               1 (1)    <-- loads 1
             25 BINARY_SUBTRACT                   <-- subtracts & pushes result onto stack
             26 LOAD_FAST                1 (accum)<-- loads accum
             29 LOAD_FAST                0 (n)    <-- loads n
             32 BINARY_MULTIPLY                   <-- multiples & pushes result onto stack
             33 CALL_FUNCTION            2        <-- calls function with two args on top of stack
             36 RETURN_VALUE                      <-- return the result
             37 LOAD_CONST               0 (None)
             40 RETURN_VALUE
```

The instruction `CALL_FUNCTION` expects a number of arguments on the top of the stack and the function object itself beneath them, like this:

``` text
 ----------
|  arg 2   |
 ----------
|  arg 1   |
 ----------
| func_obj |
 ----------
```

In our example, it looks like this:

``` text
 ---------------------
|  value of accum * n |
 ---------------------
|    value of n - 1   |
 ---------------------
|     fact function   |
 ---------------------
```

Notice that the calculations of `accum * n` and `n - 1` are sitting on the stack, ready to be passed as arguments to the recursive call.  Instead of doing that, we'll replace the instruction `CALL_FUNCTION` with instructions to store the calculated values as the new values of `n` and `accum`. Then we'll insert an instruction to jump to the beginning of the bytecode.  When we're done, we want the bytecode to look like this:

``` python
>>> dis.dis(new_fact)
  2     >>    0 LOAD_FAST                0 (n)
              3 LOAD_CONST               1 (1)
              6 COMPARE_OP               1 (<=)
              9 POP_JUMP_IF_FALSE       16

  3          12 LOAD_FAST                1 (accum)
             15 RETURN_VALUE                       <-- only reachable exit point

                                                   <-- loading the function is gone
  5     >>   16 LOAD_FAST                0 (n)
             19 LOAD_CONST               1 (1)
             22 BINARY_SUBTRACT
             23 LOAD_FAST                1 (accum)
             26 LOAD_FAST                0 (n)
             29 BINARY_MULTIPLY
             30 STORE_FAST               1 (accum) <-- store new val of accum
             33 STORE_FAST               0 (n)     <-- store new val of n
             36 JUMP_ABSOLUTE            0         <-- jump back to start
             39 RETURN_VALUE                       <-- never reached!
             40 LOAD_CONST               0 (None)
             43 RETURN_VALUE                       <-- also never reached
```

Here's the code to do the bytecode transformation:

``` python
def tail_recurse(fn):
    new_bytecode = []
    code_obj = fn.__code__
    for byte, arg in consume(code_obj.co_code):
        name = opcode.opname[byte]
        if name == "LOAD_GLOBAL" and code_obj.co_names[arg] == fn.__name__:
            pass # TODO: fix jump targets
        elif name == "CALL_FUNCTION":
            for i in range(arg):
                new_bytecode.append(opmap["STORE_FAST"])
                new_bytecode += integer_as_two_bytes(arg - i - 1)
            new_bytecode.append(opmap["JUMP_ABSOLUTE"])
            new_bytecode += integer_as_two_bytes(0) # jump to beginning of bytecode
        else:
            new_bytecode.append(byte)
            if arg is not None:
                new_bytecode += integer_as_two_bytes(arg)

    return "".join([chr(b) for b in new_bytecode])

def integer_as_two_bytes(num):
    """ Return an integer as two bytes"""
    return divmod(num, 255)[::-1]

def consume(bytecode):
    bytecode = [ord(b) for b in bytecode]
    i = 0
    while i < len(bytecode):
        op = bytecode[i]
        if op > opcode.HAVE_ARGUMENT:
            args = bytecode[i+1:i+3]
            arg = args[0] + (args[1] << 8)
            yield op, arg
            i += 3
        else:
            yield op, None
            i += 1

```

And we're done! Let's calculate the factorial of 1,000!

``` text
$ python byteme/replace_code.py
402387260077093773543702433923003985719374864210714632543799910429938512398629020592044208486969404800479988610197196058631666872994808558901323829669944590997424504087073759918823627727188732519779505950995276120874975462497043601418278094646496291056393887437886487337119181045825783647849977012476632889835955735432513185323958463075557409114262417474349347553428646576611667797396668820291207379143853719588249808126867838374559731746136085379534524221586593201928090878297308431392844403281231558611036976801357304216168747609675871348312025478589320767169132448426236131412508780208000261683151027341827977704784635868170164365024153691398281264810213092761244896359928705114964975419909342221566832572080821333186116811553615836546984046708975602900950537616475847728421889679646244945160765353408198901385442487984959953319101723355556602139450399736280750137837615307127761926849034352625200015888535147331611702103968175921510907788019393178114194545257223865541461062892187960223838971476088506276862967146674697562911234082439208160153780889893964518263243671616762179168909779911903754031274622289988005195444414282012187361745992642956581746628302955570299024324153181617210465832036786906117260158783520751516284225540265170483304226143974286933061690897968482590125458327168226458066526769958652682272807075781391858178889652208164348344825993266043367660176999612831860788386150279465955131156552036093988180612138558600301435694527224206344631797460594682573103790084024432438465657245014402821885252470935190620929023136493273497565513958720559654228749774011413346962715422845862377387538230483865688976461927383814900140767310446640259899490222221765904339901886018566526485061799702356193897017860040811889729918311021171229845901641921068884387121855646124960798722908519296819372388642614839657382291123125024186649353143970137428531926649875337218940694281434118520158014123344828015051399694290153483077644569099073152433278288269864602789864321139083506217095002597389863554277196742822248757586765752344220207573630569498825087968928162753848863396909959826280956121450994871701244516461260379029309120889086942028510640182154399457156805941872748998094254742173582401063677404595741785160829230135358081840096996372524230560855903700624271243416909004153690105933983835777939410970027753472000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
```

### Complexities from here
This implementation is a very minimal and brittle TCO. Liuda went on to fix a bunch of the remaining problems (like loading other globals in the middle of a recursive call, rewriting the jump targets, and so on). Check out her code [here](link).

### Aside: Fun facts about the recursion limit
Attentive readers may have been surprised above that a recursion limit of 1000 starts throwing an error on `factorial(1000)` rather than `factorial(1001)`. This is because the "recursion" limit is actually a cap on size of the call stack, regardless of whether or not the function calls on it are recursive. The first stack frame is the top-level module frame, so only 999 more are available. We can test this by nesting a recursive function in a couple of wrapper functions:

``` python
>>> def one(n):
...     def two(n):
...         def countdown(n, times_called=[0]):
...             times_called[0] += 1
...             if n == 0:
...                 return times_called
...             else:
...                 try:
...                     return countdown(n-1)
...                 except RuntimeError:
...                     return times_called
...         return countdown(n)
...     return two(n)
...
>>> one(10)
[11]
>>> one(100)
[101]
>>> one(1000)
[997]
```

We could verify this further by writing nested functions programmatically:

``` python
def write_nested_functions(depth):
    with open('/usr/share/dict/words','r') as name_file:
        fn_names = [line.strip() for line in name_file.readlines()][:depth]
    with open("empty_nest.py", 'w') as f:
        for depth, func_name in enumerate(fn_names):
            f.write("    " * depth + "def %s():\n" % func_name)
        f.write("    " * (depth + 1) + "print 'yay'\n")
        for depth, func_name in reversed(list(enumerate(fn_names))):
            f.write("    " * depth + "%s()\n" % func_name)
```

... which outputs this for a depth of 10:

``` python empty_nest.py
def A():
    def a():
        def aa():
            def aal():
                def aalii():
                    def aam():
                        def Aani():
                            def aardvark():
                                def aardwolf():
                                    def Aaron():
                                        print 'yay'
                                    Aaron()
                                aardwolf()
                            aardvark()
                        Aani()
                    aam()
                aalii()
            aal()
        aa()
    a()
A()
```

Sadly, we can't get even close to the maximum allowable recursion depth using this tactic, because Python won't compile a file with more than 20 levels of indentation.  When we run the function above with a depth of 1000 and try to run the resulting file, we get this failure:

``` text
$ python empty_nest.py
  File "empty_nest.py", line 101
    print 'yay'
              ^
IndentationError: too many levels of indentation
```

Undettered, we can write a [script](link) that generates chained functions rather than nested functions, along the lines of the below:

``` python chain.py
def A():
    a()

def a():
    aa()

...

def accordancy():
    accordant()

def accordant():
    accordantly()

def accordantly():
    print 'yay'

A()
```

Attempting to run this script on 1000 chained functions shows that we do indeed throw a `RuntimeError` when the call stack reaches 1000, despite having no actual recursion in the traditional sense of the word.

``` text
$ python empty_nest.py
Traceback (most recent call last):
  File "empty_nest.py", line 2053, in <module>
    A()
  File "empty_nest.py", line 41, in A
    a()
  ...
  File "empty_nest.py", line 2993, in accordance
    accordancy()
  File "empty_nest.py", line 2996, in accordancy
    accordant()
RuntimeError: maximum recursion depth exceeded
```
