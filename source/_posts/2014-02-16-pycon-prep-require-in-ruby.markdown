---
layout: post
title: "PyCon prep: `require` in Ruby"
date: 2014-02-16 14:18
comments: false
categories: ruby, pycon
---

I'm talking about `import` at PyCon in April. In the talk, we'll imagine that there is no `import` and will reinvent it from scratch. I hope this will give everyone (including me!) a deeper understanding of the choices `import` makes and the ways it could have been different. Ideally, the structure will be a couple of sections of the form "We could have made [decisions].  That would mean [effects].  Surprise - that's how it works in [language]!"[^1]

This is the first of (probably) several posts with notes of things I'm learning as I prepare my talk.  Feedback is welcome.

Today I'm looking into Ruby's `require` and `require_relative`[^2] to see if aspects of them would be interesting to Python programmers. So far, here's what I think is most relevant:

- Unlike Python, `require` won't load all objects in the required file. There's a concept of local versus global variables in the file scope that doesn't exist in Python.

- Unlike Python, one file does not map to one module. Modules are created by using the keyword `module`.

- Unlike Python, namespace collisions are completely possible. Consider the following simple files:

``` ruby one.rb
puts "one!"

def foo
  :hello
end
```

``` ruby two.rb
puts "two!"

def foo
  :world
end
```

``` ruby main.rb
require_relative 'one'
require_relative 'two'

puts foo
```

And the output from running `main.rb`:

``` ruby output
one!
two!
world
```

- Like Python's `import`, `require` will only load a file once. This can interact interestingly with namespace collisions - to take a contrived example:
``` ruby main.rb
require_relative 'one'
require_relative 'two'
require_relative 'one'

puts foo
```

Because `one.rb` isn't reloaded, `foo` is still `'world'`:

``` ruby output
one!
two!
world
```

### Questions for further investigation / thought
My talk should *not* convince people that Python is Right and other languages are Wrong.  I'm trying to overcome my bias towards the system I'm most used to. (I think I've written roughly equal amounts of Python and Ruby, but the vast majority of the Ruby I've written is Rails, where all the `require`ing and namespacing happens by magic.) Here are some questions I'd like to research more.

1. Python's namespacing feels much better to me, although I'm sure that's partly because I'm used to it. What's the advantage to doing namespacing this way?

2. Why have both `require` and `require_relative`? Why not have `require` check the relative path as well before raising a `LoadError`?

3. What's the advantage of uncoupling a module from a file?

[^1]: I [asked on twitter](https://twitter.com/akaptur/status/433281929199095809) for suggestions of languages that make interesting decisions about `import` equivalents. So far the suggestions are R, Go, Rust, Ruby, JavaScript, and Clojure. If you have others, let me know.

[^2]: As far as I can tell, the only difference between `require` and `require_relative` is the load path searched.