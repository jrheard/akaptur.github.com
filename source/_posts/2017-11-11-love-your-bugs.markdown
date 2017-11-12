---
layout: post
title: "Love your bugs"
date: 2017-11-12 18:23
comments: true
categories: 
---

In early October I gave a keynote at [Python Brasil](//2017.pythonbrasil.org.br/#) in Belo Horizonte. Here is an aspirational and lightly edited transcript of the talk. There is also a video available [here](//www.youtube.com/watch?v=h4pZZOmv4Qs).

### I love bugs

I'm currently a senior engineer at [Pilot.com](//www.pilot.com), working on automating bookkeeping for startups. Before that, I worked for [Dropbox](//www.dropbox.com) on the desktop client team, and I'll have a few stories about my work there. Earlier, I was a facilitator at the [Recurse Center](//www.recurse.com), a writers retreat for programmers in NYC. I studied astrophysics in college and worked in finance for a few years before becoming an engineer.

But none of that is really important to remember - the only thing you need to know about me is that I love bugs. I love bugs because they're entertaining. They're dramatic. The investigation of a great bug can be full of twists and turns. A great bug is like a good joke or a riddle - you're expecting one outcome, but the result veers off in another direction.

Over the course of this talk I'm going to tell you about some bugs that I have loved, explain why I love bugs so much, and then convince you that you should love bugs too.

## Bug #1

Ok, straight into bug #1. This is a bug that I encountered while working at Dropbox. As you may know, Dropbox is a utility that syncs your files from one computer to the cloud and to your other computers.

```
            +--------------+     +---------------+
            |              |     |               |
            |  METASERVER  |     |  BLOCKSERVER  |
            |              |     |               |
            +-+--+---------+     +---------+-----+
              ^  |                         ^
              |  |                         |
              |  |     +----------+        |
              |  +---> |          |        |
              |        |  CLIENT  +--------+
              +--------+          |
                       +----------+
```

Here's a vastly simplified diagram of Dropbox's architecture. The desktop client runs on your local computer listening for changes in the file system. When it notices a changed file, it reads the file, then hashes the contents in 4MB blocks. These blocks are stored in the backend in a giant key-value store that we call blockserver. The key is the digest of the hashed contents, and the values are the contents themselves.

Of course, we want to avoid uploading the same block multiple times. You can imagine that if you're writing a document, you're probably mostly changing the end - we don't want to upload the beginning over and over. So before uploading a block to the blockserver the client talks to a different server that's responsible for managing metadata and permissions, among other things. The client asks metaserver whether it needs the block or has seen it before. The  "metaserver" responds with whether or not each block needs to be uploaded.

So the request and response look roughly like this: The client says, "I have a changed file made up of blocks with hashes `'abcd,deef,efgh'`". The server responds, "I have those first two, but upload the third." Then the client sends the block up to the blockserver.

```
                    +--------------+     +---------------+
                    |              |     |               |
                    |  METASERVER  |     |  BLOCKSERVER  |
                    |              |     |               |
                    +-+--+---------+     +---------+-----+
                      ^  |                         ^
                      |  | 'ok, ok, need'          |
    'abcd,deef,efgh'  |  |     +----------+        | efgh: [contents]
                      |  +---> |          |        |
                      |        |  CLIENT  +--------+
                      +--------+          |
                               +----------+
```

That's the setup. So here's the bug.

```
                    +--------------+
                    |              |
                    |  METASERVER  |
                    |              |
                    +-+--+---------+
                      ^  |
                      |  |   '???'
    'abcdldeef,efgh'  |  |     +----------+
         ^            |  +---> |          |
         ^            |        |  CLIENT  +
                      +--------+          |
                               +----------+
```

Sometimes the client would make a weird request: each hash value should have been sixteen characters long, but instead it was thirty-three characters long - twice as many plus one. The server wouldn't know what to do with this and would throw an exception. We'd see this exception get reported, and we'd go look at the log files from the desktop client, and really weird stuff would be going on - the client's local database had gotten corrupted, or python would be throwing MemoryErrors, and none of it would make sense.

If you've never seen this problem before, it's totally mystifying. But once you'd seen it once, you can recognize it every time thereafter. Here's a hint: the middle character of each 33-character string that we'd often see instead of a comma was `l`. These are the other characters we'd see in the middle position:

```
l \x0c < $ ( . -
```

The ordinal value for an ascii comma - `,` - is 44. The ordinal value for `l` is 108. In binary, here's how those two are represented:

```
bin(ord(',')): 0101100  
bin(ord('l')): 1101100  
```

You'll notice that an `l` is exactly one bit away from a comma. And herein lies your problem: a bitflip. One bit of memory that the desktop client is using has gotten corrupted, and now the desktop client is sending a request to the server that is garbage.

And here are the other characters we'd frequently see instead of the comma when a different bit had been flipped.

```
,    : 0101100
l    : 1101100
\x0c : 0001100
<    : 0111100
$    : 0100100
(    : 0101000
.    : 0101110
-    : 0101101
```

### Bitflips are real!

I love this bug because it shows that bitflips are a real thing that can happen, not just a theoretical concern. In fact, there are some domains where they're more common than others. One such domain is if you're getting requests from users with low-end or old hardware, which is true for a lot of laptops running Dropbox. Another domain with lots of bitflips is outer space - there's no atmosphere in space to protect your memory from energetic particles and radiation, so bitflips are pretty common.

You probably really care about correctness in space - your code might be keeping astronauts alive on the ISS, for example, but even if it's not mission-critical, it's hard to do software updates to space. If you really need your application to defend against bitflips, there are a variety of hardware & software approaches you can take, and there's a [very interesting talk](//www.youtube.com/watch?v=ETgNLF_XpEM) by Katie Betchold about this.

Dropbox in this context doesn't really need to protect against bitflips. The machine that is corrupting memory is a user's machine, so we can detect if the bitflip happens to fall in the comma - but if it's in a different character we don't necessarily know it, and if the bitflip is in the actual file data read off of disk, then we have no idea. There's a pretty limited set of places where we could address this, and instead we decide to basically silence the exception and move on. Often this kind of bug resolves after the client restarts.

### Unlikely bugs aren't impossible

This is one of my favorite bugs for a couple of reasons. The first is that it's a reminder of the difference between unlikely and impossible. At sufficient scale, unlikely events start to happen at a noticable rate. 

### Social bugs

My second favorite thing about this bug is that it's a tremendously social one. This bug can crop up anywhere that the desktop client talks to the server, which is a lot of different endpoints and components in the system. This meant that a lot of different engineers at Dropbox would see versions of the bug. The first time you see it, you can _really_ scratch your head, but after that it's easy to diagnose, and the investigation is really quick: you look at the middle character and see if it's an `l`.

### Cultural differences

One interesting side-effect of this bug was that it exposed a cultural difference between the server and client teams. Occasionally this bug would be spotted by a member of the server team and investigated from there. If one of your _servers_ is flipping bits, that's probably not random chance - it's probably memory corruption, and you need to find the affected machine and get it out of the pool as fast as possible or you risk corrupting a lot of user data. That's an incident, and you need to respond quickly. But if the user's machine is corrupting data, there's not a lot you can do.

### Share your bugs

So if you're investigating a confusing bug, especially one in a big system, don't forget to talk to people about it. Maybe your colleagues have seen a bug shaped like this one before. If they have, you might save a lot of time. And if they haven't, don't forget to tell people about the solution once you've figured it out - write it up or tell the story in your team meeting. Then the next time your teams hits something similar, you'll all be more prepared.


## How bugs can help you learn

### Recurse Center

Before I joined Dropbox, I worked for the Recurse Center. The idea behind RC is that it's a community of self-directed learners spending time together getting better as programmers. That is the full extent of the structure of RC: there's no curriculum or assignments or deadlines. The only scoping is a shared goal of getting better as a programmer. We'd see people come to participate in the program who had gotten CS degrees but didn't feel like they had a solid handle on practical programming, or people who had been writing Java for ten years and wanted to learn Clojure or Haskell, and many other profiles as well. 

My job there was as a facilitator, helping people make the most of the lack of structure and providing guidance based on what we'd learned from earlier participants. So my colleagues and I were very interested in the best techniques for learning for self-motivated adults.

### Deliberate Practice

There's a lot of different research in this space, and one of the ones I think is most interesting is the idea of deliberate practice. Deliberate practice is an attempt to explain the difference in performance between experts & amateurs. And the guiding principle here is that if you look just at innate characteristics - genetic or otherwise - they don't go very far towards explaining the difference in performance. So the researchers, originally Ericsson, Krampe, and Tesch-Romer, set out to discover what did explain the difference. And what they settled on was time spent in deliberate practice.

Deliberate practice is pretty narrow in their definition: it's not work for pay, and it's not playing for fun. You have to be operating on the edge of your ability, doing a project appropriate for your skill level (not so easy that you don't learn anything and not so hard that you don't make any progress). You also have to get immediate feedback on whether or not you've done the thing correctly.

This is really exciting, because it's a framework for how to build expertise. But the challenge is that as programmers this is really hard advice to apply. It's hard to know whether you're operating at the edge of your ability. Immediate corrective feedback is very rare - in some cases you're lucky to get feedback ever, and in other cases maybe it takes months. You can get quick feedback on small things in the REPL and so on, but if you're making a design decision or picking a technology, you're not going to get feedback on those things for quite a long time.

But one category of programming where deliberate practice is a useful model is debugging. If you wrote code, then you had a mental model of how it worked when you wrote it. But your code has a bug, so your mental model isn't quite right. By definition you're on the boundary of your understanding - so, great! You're about to learn something new. And if you can reproduce the bug, that's a rare case where you can get immediate feedback on whether or not your fix is correct.

A bug like this might teach you something small about your program, or you might learn something larger about the system your code is running in. Now I've got a story for you about a bug like that.

## Bug #2

This bug also one that I encountered at Dropbox. At the time, I was investigating why some desktop client weren't sending logs as consistently as we expected. I'd started digging into the client logging system and discovered a bunch of interesting bugs. I'll tell you only the subset of those bugs that is relevant to this story.

Again here's a very simplified architecture of the system.

```
                                       +--------------+
                                       |              |
                   +---+  +----------> |  LOG SERVER  |
                   |log|  |            |              |
                   +---+  |            +------+-------+
                          |                   |
                    +-----+----+              |  200 ok
                    |          |              |
                    |  CLIENT  |  <-----------+
                    |          |
                    +-----+----+
                          ^
                          +--------+--------+--------+
                          |        ^        ^        |
                       +--+--+  +--+--+  +--+--+  +--+--+
                       | log |  | log |  | log |  | log |
                       |     |  |     |  |     |  |     |
                       |     |  |     |  |     |  |     |
                       +-----+  +-----+  +-----+  +-----+
```

The desktop client would generate logs. Those logs were compress, encrypted, and written to disk. Then every so often the client would send them up to the server. The client would read a log off of disk and send it to the log server. The server would decrypt it and store it, then respond with a 200.

If the client couldn't reach the log server, it wouldn't let the log directory grow unbounded. After a certain point it would start deleting logs to keep the directory under a maximum size.

The first two bugs were not a big deal on their own. The first one was that the desktop client sent logs up to the server starting with the oldest one instead of starting with the newest. This isn't really what you want - for example, the server would tell the client to send logs if the client reported an exception, so probably you care about the logs that just happened and not the oldest logs that happen to be on disk. 

The second bug was similar to the first: if the log directory hit its maximum size, the client would delete the logs starting with the newest instead of starting with the oldest. Again, you lose log files either way, but you probably care less about the older ones.

The third bug had to do with the encryption. Sometimes, the server would be unable to decrypt a log file. (We generally didn't figure out why - maybe it was a bitflip.) We weren't handling this error correctly on the backend, so the server would reply with a 500. The client would behave reasonably in the face of a 500: it would assume that the server was down. So it would stop sending log files and not try to send up any of the others.

Returning a 500 on a corrupted log file is clearly not the right behavior. You could consider returning a 400, since it's a problem with the client request. But the client also can't fix the problem - if the log file can't be decrypted now, we'll never be able to decrypt it in the future. What you really want the client to do is just delete the log and move on. In fact, that's the default behavior when the client gets a 200 back from the server for a log file that was successfully stored. So we said, ok - if the log file can't be decrypted, just return a 200.

All of these bugs were straightforward to fix. The first two bugs were on the client, so we'd fixed them on the alpha build but they hadn't gone out to the majority of clients. The third bug we fixed on the server and deployed.

### 📈

Suddenly traffic to the log cluster spikes. The serving team reaches out to us to ask if we know what's going on. It takes me a minute to put all the pieces together.

Before these fixes, there were four things going on:

1. Log files were sent up starting with the oldest
2. Log files were deleted starting with the newest
3. If the server couldn't decrypt a log file it would 500
4. If the client got a 500 it would stop sending logs

A client with a corrupted log file would try to send it, the server would 500, the client would give up sending logs. On its next run, it would try to send the same file again, fail again, and give up again. Eventually the log directory would get full, at which point the client would start deleting its newest files, leaving the corrupted one on disk.

The upshot of these three bugs: if a client ever had a corrupted log file, we would never see logs from that client again. 

The problem is that there were a lot more clients in this state than we thought. Any client with a single corrupted file had been dammed up from sending logs to the server. Now that dam was cleared, and all of them were sending up the rest of the contents of their log directories.

### Our options

Ok, there's a huge flood of traffic coming from machines around the world. What can we do? (This is a fun thing about working at a company with Dropbox's scale, and particularly Dropbox's scale of desktop clients: you can trigger a self-DDOS very easily.)

The first option when you do a deploy and things start going sideways is to rollback. Totally reasonable choice, but in this case, it wouldn't have helped us. The state that we'd transformed wasn't the state on the server but the state on the client - we'd deleted those files. Rolling back the server would prevent additional clients from entering this state but it wouldn't solve the problem.

What about increasing the size of the logging cluster? We did that - and started getting even more requests, now that we'd increased our capacity. We increased it again, but you can't do that forever. Why not? This cluster isn't isolated. It's making requests into another cluster, in this case to handle exceptions. If you have a DDOS pointed at one cluster, and you keep scaling that cluster, you're going to knock over its depedencies too, and now you have two problems.

Another option we considered was shedding load - you don't need every single log file, so can we just drop requests. One of the challenges here was that we didn't have an easy way to tell good traffic from bad. We couldn't quickly differentiate which log files were old and which were new.

The solution we hit on is one that's been used at Dropbox on a number of different occassions: we have a custom header, `chillout`, which every client in the world respects. If the client gets a response with this header, then it doesn't make any requests for the provided number of seconds. Someone very wise added this to the Dropbox client very early on, and it's come in handy more than once over the years.  The logging server didn't have the ability to set that header, but that's an easy problem to solve. So two of my colleagues, Isaac Goldberg and John Lai, implemented support for it. We set the logging cluster chillout to two minutes initially and then managed it down as the deluge subsided over the next couple of days.

### Know your system

The first lesson from this bug is to know your system. I had a good mental model of the interaction between the client and the server, but I wasn't thinking about what would happen when the server was interacting with all the clients at once. There was a level of complexity that I hadn't thought all the way through. 

### Know your tools

The second lesson is to know your tools. If things go sideways, what options do you have? Can you reverse your migration? How will you know if things are going sideways and how can you discover more? All of those things are great to know before a crisis - but if you don't, you'll learn them during a crisis and then never forget.

### Feature flags & server-side gating

The third lesson is for you if you're writing a mobile or a desktop application: _You need server-side feature gating and server-side flags._ When you discover a problem and you don't have server-side controls, the resolution might take days or weeks as you push out a new release or submit a new version to the app store. That's a bad situation to be in. The Dropbox desktop client isn't going through an app store review process, but just pushing out a build to tens of millions of clients takes time. Compare that to hitting a problem in your feature and flipping a switch on the server: ten minutes later your problem is resolved.

This strategy is not without its costs. Having a bunch of feature flags in your code adds to the complexity dramatically. You get a combinatoric problem with your testing: what if feature A is enabled and feature B, or just one, or neither - multiplied across N features. It's extremely difficult to get engineers to clean up their feature flags after the fact (and I was also guilty of this). Then for the desktop client there's multiple versions in the wild at the same time, so it gets pretty hard to reason about. 

But the benefit - man, when you need it, you really need it.

# How to love bugs

I've talked about some bugs that I love and I've talked about why to love bugs. Now I want to tell you how to love bugs. If you don't love bugs yet, I know of exactly one way to learn, and that's to have a growth mindset.

The sociologist Carol Dweck has done a ton of interesting research about how people think about intelligence. She's found that there are two different frameworks for thinking about intelligence. The first, which she calls the fixed mindset, holds that intelligence is a fixed trait, and people can't change how much of it they have. The other mindset is a growth mindset. Under a growth mindset, people believe that intelligence is malleable and can increase with effort.

Dweck found that a person's theory of intelligence - whether they hold a fixed or growth mindset - can significantly influence the way they select tasks to work on, the way they respond to challenges, their cognitive performance, and even their honesty.

[I also talked about a growth mindset in my Kiwi PyCon keynote, so here are just a few excerpts. You can read the full transcript [here](/blog/2015/10/10/effective-learning-strategies-for-programmers/).]

Findings about honesty:

> After this, they had the students write letters to pen pals about the study, saying "We did this study at school, and here's the score that I got." They found that _almost half of the students praised for intelligence lied about their scores_, and almost no one who was praised for working hard was dishonest.

On effort:

> Several studies found that people with a fixed mindset can be reluctant to really exert effort, because they believe it means they're not good at the thing they're working hard on. Dweck notes, "It would be hard to maintain confidence in your ability if every time a task requires effort, your intelligence is called into question."

On responding to confusion:

> They found that students with a growth mindset mastered the material about 70% of the time, regardless of whether there was a confusing passage in it. Among students with a fixed mindset, if they read the booklet without the confusing passage, again about 70% of them mastered the material. But the fixed-mindset students who encountered the confusing passage saw their mastery drop to 30%. Students with a fixed mindset were pretty bad at recovering from being confused.

These findings show that a growth mindset is critical while debugging. We have to recover from confusion, be candid about the limitations of our understanding, and at times really struggle on the way to finding solutions - all of which is easier and less painful with a growth mindset.

### Love your bugs

I learned to love bugs by explicitly celebrating challenges while working at the Recurse Center. A participant would sit down next to me and say, "[sigh] I think I've got a weird Python bug," and I'd say, "Awesome, I _love_ weird Python bugs!" First of all, this is definitely true, but more importantly, it emphasized to the participant that finding something where they struggled an accomplishment, and it was a good thing for them to have done that day.

As I mentioned, at the Recurse Center there are no deadlines and no assignments, so this attitude is pretty much free. I'd say, "You get to spend a day chasing down this weird bug in Flask, how exciting!" At Dropbox and later at Pilot, where we have a product to ship, deadlines, and users, I'm not always uniformly delighted about spending a day on a weird bug. So I'm sympathetic to the reality of the world where there are deadlines. However, if I have a bug to fix, I have to fix it, and being grumbly about the existence of the bug isn't going to help me fix it faster. I think that even in a world where deadlines loom, you can still apply this attitude.

If you love your bugs, you can have more fun while you're working on a tough problem. You can be less worried and more focused, and end up learning more from them. Finally, you can share a bug with your friends and colleagues, which helps you and your teammates.

### Obrigada!

My thanks to folks who gave me feedback on this talk and otherwise contributed to my being there:

- Sasha Laundy
- Amy Hanlon
- Julia Evans
- Julian Cooper
- Raphael Passini Diniz and the rest of the Python Brasil organizing team
