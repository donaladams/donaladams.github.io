---
layout: post
title:  "Parsing User Input in SBT"
date:   2016-04-19 12:00:00 +0100
categories: blog
---

SBT is an incredibly powerful tool but is easily misunderstood. There's quite good support for low-level handling of user input in custom InputTasks but it can be difficult to get started and reading the source was definitely necessary to get stuff done.

The goal of this post is to make it easier for others coming across this problem.

## git checkout -b \*\*\*\*-sbt

I've been working with Scala for the past two years and like many (most?) Scala developers, I've had a bit of a love-hate relationship with SBT. There have been some rude branch names at times when SBT just wouldn't do what I wanted (I'm sure I'm not the only one to do this). To be fair, the problem is not just SBT - my lack of understanding of SBT has definitely gotten in the way too.

My most recent frustration came about during a recent hackweek at [Nitro](www.gonitro.com). We were working on an SBT plugin to help us easily create and manage our build jobs in Jenkins. This was the first time I had worked on an SBT plugin, stepping from the cosy (?) realm of casual SBT user to SBT plugin author. As we were writing a command line tool, the time came that we needed to handle command line options. Enter `sbt.complete.Parsers` and `sbt.InputTask`

## Parsers and InputTasks

I'm not going to describe SBT from the ground up but a quick refresher of the relevant parts is in order. SBT builds are big maps of keys to values. There are three types of values that can be in this map:

1. Settings - these are static values that are computed once and never change (think `val`)
2. Tasks - these are computed every time they're required (think `Unit => T`)
3. InputTasks - these are like tasks with user input (think `String => T`)

We're going to concern ourselves with InputTasks as this is where the user input comes in. Here's the definition of InputTask in sbt:

{% highlight scala %}
class InputTask[T](val parser: State => Parser[Task[T]])
{% endhighlight %}


To make things more concrete, let's use an example that should be familiar to any developer.

## mkdir

`mkdir` is among the first unix commands that you come across when you start using the command line for the first time.

As a reminder:

`mkdir [-pv] [-m mode] directory_name ...`

We have two flags, `-p` for creating intermediate directories and `-v` for verbose output, one option `-m mode`
which specifies the permissions on the created directories, and finally one or more directory names to create.

Now let's go about writing `sbt mkdir`!



