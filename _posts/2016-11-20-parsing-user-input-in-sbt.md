---
layout: post
title: Parsing user input in SBT
date: 2016-04-19 12:00:00 +0100
categories: blog
---

I've been working with Scala for the past two years and like many (most?) Scala developers, I've had a love-hate relationship with SBT. SBT is a powerful tool but it's easy to lose your footing. There have been some rude branch names at times when SBT just wouldn't do what I wanted [^1].

The most recent such branch came about during a recent hackweek at [Nitro](www.gonitro.com). We were working on an SBT plugin to help us easily create and manage our build jobs in Jenkins. This was the first time I needed to handle command line input in SBT (other than handling simple environment variables). This post is based on the things I learned in the process. I hope it will be useful to others going down the same path.

First, we'll look at what SBT provides for writing commands line tools in general. After this, we'll look at a specific example of how to do this.

# Listen to me, SBT

When you want to add a command-line utility to an SBT project, you essentially have two options:

1. Add an `InputTask`
2. Add a `Command`

InputTasks and Commands are quite similar and can both be used to add command line utilities to your project. The similarity can be seen in the signatures of their constructors.

Here's a simplified definition of `InputTask` [^2]:

{% highlight scala %}
final class InputTask[T] private (val parser: State => Parser[Task[T]]) {
    ...
}
{% endhighlight %}

As you can see its single argument is a function that produces a parser of some sort. More on this in a moment.

Here's one of `Command`'s factory functions [^3]:

{% highlight scala %}
object Command {
    def apply[T](name: String, help: Help = Help.empty)(parser: State => Parser[T])(effect: (State, T) => State): Command
}
{% endhighlight %}

The point of similarity with `InputTask` here is the parser argument which is another function of `State => Parser`.

So what are `State` and `Parser` and how do they help us write useful tools in sbt?

# State

Well actually, `State` doesn't really help us much if we want to add simple command-line utilities so let's leave that for another day!

# Parser[T]

`Parser[T]` is the tool that SBT provides to, as you might have guessed, parse command line input. Parsers operate on `String` or `Char` input and can produce an arbitrary value `T` if successful. SBT provides a number of parsers out of the box that you can use immediately. Small parsers can be combined in many was to create more complex parsers.\

Here's an example parser that

```scala

import sbt.complete.DefaultParsers._ // predefined parsers live here


// Parser that matches "hello <name>!" and returns <name>
val helloWorld: Parser[String] = "hello" ~> Space ~> StringBasic <~ '!' <~ EOF

```

## mkdir

To make things more concrete, let's use an example that should be familiar. `mkdir` is among the first unix commands that you come across when you start using the command line. As a reminder, here's what it looks like:

`mkdir [-pv] [-m mode] directory_name ...`

We have two flags, `-p` for creating intermediate directories and `-v` for verbose output, one option `-m mode`
which specifies the permissions on the created directories, and finally one or more directory names to create.

Now let's go about writing `sbt mkdir`!





{% comment %} PARSER VS SCOPT {% endcomment%}


{% comment %} CONCLUSION {% endcomment%}


[^1]: `git checkout -b ****-sbt`
[^2]: [InputTask.scala](https://github.com/sbt/sbt/blob/0.13/main/settings/src/main/scala/sbt/InputTask.scala)
[^3]: [Command.scala](https://github.com/sbt/sbt/blob/0.13/main/command/src/main/scala/sbt/Command.scala)




