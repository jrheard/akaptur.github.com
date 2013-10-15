---
layout: post
title: "Bug hunting"
date: 2013-10-15 08:01
comments: false
categories: 
---

I've managed to encounter three different bugs with the same obscure source in the last week.  I think Hacker School might be cursed.  Here's a blog post attempting to rid us of the curse.

### Bug 1: Flask app on Heroku can't find images and stylesheets
The first bug was a Flask app deployed to Heroku.  It worked fine locally, but when deployed, none of the images or stylesheets rendered.

### Bug 2: A project fails to build
A Hacker Schooler cloned into a project and tried to build it with `python setup.py install`.  The build failed with the error `Supposed package directory '[project]' exists but is not a directory.`

### Bug 3: Heroku-deployed app crashes
I deployed a new feature to the Hacker School site (which is a Rails app), and crashed the application.  Again, everything worked fine locally on my machine and my colleague's machines.  

The solution and explanations are below the fold.  If you'd like to try to guess, you can ask me debugging questions on twitter [(@akaptur)](https://twitter.com/akaptur), and I'll respond within 24 hours until Friday, October 18th, 2013. If you don't like guessing, or your own bugs are plenty for you, you can click through now.

<!-- more -->

## PSA: Mac OS X is a case-insensitive file system, y'all
Remarkably enough, all these bugs were caused by converting from a case-insensitve file system, like Mac OS X, to a case-sensitive one, like Ubuntu.

### The flask app
In the flask app, the author had named his static folder `Static`.  Flask looks for static assets in `static` by default. No problem on his MacBook - but a big problem on Heroku.

### The build
This was my personal favorite, and a new bit of trivia: when can `git clone` result in a local repo state that's different from the remote repo?  If the project was developed on a case-sensitive file system, and there exists both a `project` script and a `Project` folder, git will first clone the script and then **silently** fail to write the identically-named folder or the files that belong in it.  Interestingly, all the blobs are still there in your local repo: `git status` will show all the files in `Project/` and the `Project/` folder itself as deleted.  Simple demo [here](https://github.com/paulvstheworld/case-sensitive-test). (Thanks, Paul!)

### The Hacker School deploy
Same story as #1 with different details: my colleague and I had done `require 'JSON'` instead of `require 'json'`, which worked fine locally and brought the entire app down on heroku.

I've finally learned to look for this bug when something mysterious happens between my machine and some other machine - which I'm sure means I'll never encounter it again!
