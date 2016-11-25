---
layout: post
title: Command Line Parsing in SBT
date: 2016-11-23 12:00:00 +0100
categories: blog
---

I've been working with Scala for the last couple of years and like many (if not most) Scala developers, I've had my ups and downs with [SBT](http://www.scala-sbt.org/). SBT is a powerful tool but it's easy to lose your footing.

During our most recent hackweek at [Nitro](http://www.gonitro.com) I worked on an SBT plugin with some colleagues to help us create and manage our build pipelines in Jenkins. This was the first time I needed to handle proper command line input in SBT.

In this post, we'll look at two ways of handling command line input in SBT:

1. The _SBT way_, using its [built-in parsers](http://www.scala-sbt.org/0.13/docs/Parsing-Input.html) from the ground up
2. Using [scopt](https://github.com/scopt/scopt) to handle command line parsing for using

I would definitely recommend the second approach if you are looking to do this yourself as I believe it is simpler, more pragmatic and more robust to change.

## mkdir

To make the discussion more concrete, we'll use an example that should be familiar to all developers. `mkdir` is among the first unix commands that you come across when you start using the command line. It has a small but non-trivial set of command line arguments, making it a good example to drive our discussion.

As a reminder, here's what it looks like:

`mkdir [-pv] [-m mode] directory_name ...`

We have three types of argument here:

1. Boolean flags: `-p` for creating intermediate directories and `-v` for verbose output
2. Flags with arguments: `-m mode` for specifying the permissions on the resulting directories
3. Space-separated string arguments for specifying the names of directories to create

Now let's implement `mkdir` in SBT!

## sbt mkdir

When you want to add a command-line utility to an SBT project, you have two options:

1. Add an [InputTask](http://www.scala-sbt.org/0.13/docs/Input-Tasks.html)
2. Add a [Command](http://www.scala-sbt.org/0.13/docs/Commands.html)

`InputTask` and `Command` are quite similar and could both be used for `sbt mkdir`, no matter what approach we take to parsing user input. The prevailing advice is to use `InputTask` as a first choice as `Command` is lower level, allowing you to actually modify the state of the build[^1].

Here's a simplified definition of `InputTask`[^2] and the signature of one of the constructors for `Command`[^3]:

```scala
final class InputTask[T] private (val parser: State => Parser[Task[T]]) {
    ...
}

object Command {
    def apply[T](name: String, help: Help = Help.empty)(parser: State => Parser[T])(effect: (State, T) => State): Command
}

```

The point of similarity between these two is the argument `parser: State => Parser[_]`. The `Parser[_]` returned by this function is where we get access to the user's command line input.

## Approach 1: The SBT way

`Parser[T]` is the tool that SBT provides to, as you might have guessed, parse command line input[^4]. `Parser[T]` defines the low-level methods needed for directly implementing new parsers. Generally, you won't need to do this. Instead, SBT provides a number of parsers for common scenarios in `sbt.complete.Parsers`[^5]. You will generally combine these together to build more useful `Parser[T]` instances using the combinators in `sbt.complete.RichParser` which implicitly enriches `Parser[T]`.

If you decide that you need to use these parsers in your SBT project, you will need to familiarise yourself with both `sbt.complete.Parser` and more importantly `sbt.complete.Parsers`.

### Step 1: Add an InputTask or Command to your build definition

Here is what an `InputTask` and a `Command` would look like for `sbt mkdir` in our `build.sbt` file.

```scala
import sbt.complete.Parser // defines the basic Parser[T] infrastructure
import sbt.complete.DefaultParsers._ // provides built-in parsers for common types
import Mkdir.{run, MkdirCommand}

// Let's assume for now that this exists!
lazy val mkdirParser: Parser[MkdirCommand] = ...

```

Here's `sbt mkdir` as an `InputTask`:

```scala
lazy val mkdir = inputKey[Unit]("make directories with the help of sbt parsers")

mkdir := {
  // The ".parsed" macro is available in InputTasks and
  // it applies the given parser the user's command line input.
  val mkdirCommand: MkdirCommand = mkdirParser.parsed

  // implementation of mkdir
  Mkdir.run(mkdirCommand)
}
```

And here it is as a `Command`:

```scala
def mkdirCmd = Command("mkdir")(_ => mkdirParser) { (state, mkdirCmd: MkdirCommand) =>
    Mkdir.run(mkdirCommand)
    state
  }

// Add the Command to the list of commands in the project settings
commands ++= Seq(mkdirCmd)
```

So far these are both quite straightforward and you're free to choose which suits your preference and situation better. Whichever way you go, you will need a `Parser[MkdirCommand]` to handle the input.

### Step 2: Define MkdirCommand to collect the input

This is the simple data structure used to collect the information we want from `sbt mkdir`'s command line arguments.

```scala
case class MkdirCommand(verbose: Boolean = false,
                        createIntermediate: Boolean = false,
                        mode: Option[Mode] = None,
                        directories: Seq[File] = Seq())
```

### Step 3: Grammar Time

When working with parsers, it's really helpful to define a grammar for the input you want to parse. This might sound
daunting but it's really not so bad. Even if it's not completely formal, it will prove invaluable while we are building up our `Parser[T]` from the ground up.

Here's a possible BNF-like grammar[^6] for `mkdir`. This borrows from `man chmod` which helpfully provides a grammar for the "symbolic mode" used in `mkdir`.

{% highlight c %}

// top-level rule defining mkdir cmd
cmd 		::= mkdir [options] directories
directories     ::=  [directory_name ...] directory_name
// command line options
options 	::= mode_option | flag_options

// modes from chmod
mode_option 	::= -m mode

mode 		::= symbolic_mode | absolute_mode

// absolute modes e.g. 0777
absolute_mode 	::= octal_number
octal_number    ::= octal_digit | octal_number
octal_digit 	::= 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7

// symbolic modes e.g. u=rwx,go=rx
symbolic_mode 	::= clause [, clause ...]
clause        	::= [who ...] [action ...] action
action        	::= op [perm ...]
who           	::= a | u | g | o
op            	::= + | - | =
perm          	::= r | s | t | w | x | X | u | g | o

// boolean flags
flag_options    ::= [flag_option ...]
flag_option     ::= - flag
flag            ::= v | p


{% endhighlight %}

### Step 4: Construct Parser[MkdirCommand]

Now that we have our grammar in place, writing the parsers is (theoretically) straightforward.

The approach will be to create parsers for the "terminals" in the grammar (those values that are only found on the right-hand side of the rules) using SBT's built-in parsers.

Then we will successively build larger and larger parsers by combining smaller ones until we get a `Parser[MkdirCommand]` that handles the entire `sbt mkdir` input defined by our grammmar.

Let's look at some examples.

#### Parsing directories

Without a parser for handling directory names, `mkdir` wouldn't be much use so let's start here. This is the part of the grammar we're interested in:

```c
directories     ::=  [directory_name ...] directory_name
```

We need a `Parser[Seq[File]]` that extracts the list of directories to create from the user input. Let's start from the bottom and work our way up.

First, we define `val directory: Parser[File]` that uses the built-in `StringBasic` parser to match a `String` in the user input. We then map over this `Parser[String]` to transform it into a `Parser[File]` - the type use to represent directories in `MkdirCommand`. This parser is equivalent to the `directory_name` terminal in our grammar.

Next, `mkdir` accepts a sequence of directories so we need to define `val directories: Parser[Seq[File]]` that combines our `directory` parser with the built in `Space` parser and then repeats this combined parser one or more times to transform a space-separated list of strings into a `Seq[File]`.

```scala
val directory: Parser[File] = StringBasic.map(p => new File(p))
val directories: Parser[Seq[File]] = (Space ~> directory)+
```

This example introduces serveral other new symbols:

* `Space` - this a built-in parser that matches one or more whitespace characters
* `~>` - this is used to combine parsers in sequence, returning the value to the right of `~>`, in this case the File representing the directory.
* `+` - match the parser one or more times

#### Parsing modes

Next, let's build a parser for another part of our grammar - the mode option `-m mode`. The value of `mode` can be one of two types, "symbolic mode" or an "absolute mode", both defined in `chmod`'s man pages.

First, we define some simple data structures to help us represent these modes and their components in our code.

```scala
sealed trait Op {
  def symbol: Char
}

object Op {
  def apply(char: Char): Op = char match {
    case Plus.symbol => Plus
    case Minus.symbol => Minus
    case Equals.symbol => Equals
  }
}
case object Plus extends Op { override val symbol = '+' }
case object Minus extends Op { override val symbol = '-'}
case object Equals extends Op { override val symbol = '=' }

case class Who(char: Char)
case class Permission(char: Char)
case class Action(op: Op, permissions: Seq[Permission])
case class Clause(who: Seq[Who], actions: Seq[Action])

sealed trait Mode
case class SymbolicMode(clauses: Seq[Clause]) extends Mode
case class AbsoluteMode(mode: String) extends Mode

```

First, let's tackle "absolute modes". The approach is the same as before - we start with a terminal parser `val octalDigit: Parser[Char]` and build larger parsers from it using parser combinators. One thing to point out in this example is the use of `def chars(legal: String): Parser[Char]`. This comes from `sbt.complete.Parser` and parses a single `Char` if it is found in the provided string of legal characters.

```scala
// The set of valid characters for an octal digit
val octalChars = Set('0', '1', '2', '3', '4', '5', '6', '7')

// octal_digit ::= 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7
val octalDigit: Parser[Char] = chars(octalChars.mkString)

// octal_number ::= octal_digit | octal_number
val octalNumber: Parser[String] = (octalDigit+).map(_.mkString)

// absolute_mode ::= octal_number
val absoluteMode: Parser[AbsoluteMode] = octalNumber.map(AbsoluteMode)

```

Next up is the "symbolic mode" parser. The structure of this is more complicated but thankfully we have our grammar to guide us. Again, the process is the same - start at the terminals and work your way up.

```scala
// Terminal symbols
val permissionChars = Set('r', 's', 't', 'w', 'x', 'X', 'u', 'g', 'o')
val whoChars = Set('a', 'u', 'g', 'o')
val opChars = Set('+', '-', '=')

// Terminal Parsers
val permissions: Parser[Seq[Permission]] =
  chars(permissionChars.mkString).map(Permission)*
val who: Parser[Who] = chars(whoChars.mkString).map(Who)
val op: Parser[Op] = chars(opChars.mkString).map(Op(_))

// action ::= op [perm ...]
val action: Parser[Action] = (op ~ permissions).map(Action.tupled)

// clause ::= [who ...] [action ...] action
val clause: Parser[Clause] = ((who*) ~ (action+)).map(Clause.tupled)

// symbolic_mode ::= clause [, clause ...]
val symbolicMode: Parser[SymbolicMode] = ((clause <~ ','.?)+).map(SymbolicMode)

```

This example introduces some new symbols:

* `*` - this allows us to match a parser zero or more times
* `~` - similar to `~>` this allows us to match parsers in sequence but in this case, it returns the successful result from both sides of `~`. In the example above `(op ~ permissions)` has a type `Parser[(Op, Seq[Permission])]`
* `<~` - matches two parsers in sequence like `~` and `~>` but this time, it returns the value on the left
* `?` - optionally match the parser

Finally, we need to combine `val symbolicMode: Parser[SymbolicMode]` and `val absoluteMode: Parser[AbsoluteMode]` into a `Parser[Mode]` that looks for the `-m` option
and then matches either an absolute mode or a symbolic mode:

```scala
val modeParser: Parser[Mode] = ("-m" ~ Space) ~> (absoluteMode | symbolicModeParser)
```

#### Parsing the entire command

We're not done yet. To complete our `Parser[MkdirCommand]` we still need to define a parser for the remaining boolean flags (`-p` and `-v`) and then combine this with the parsers defined above into our mkdir parser `Parser[MkdirCommand]`.

The approach is the same as we have seen in the last two examples so it won't add much to show this here. Hopefully this has given you a good feeling for parsing user input the "SBT way".

If you are interested in seeing this explicitly or trying out the full `Parser[MkdirCommand]`, the full source code for these examples is [here](https://github.com/epileptic-fish/sbt-parsers).

## Thoughts on the _SBT way_

SBT's parser infrastructure is powerful and working with grammars and parsers can be fun. However, I would also like to make clear that it is not without its problems. Here are some caveats that I would like add.

* Constructing parsers like this quite work-intensive - you may need to write a lot of `Parser[T]`s to get what you want done.
* It can be difficult to express exactly what you want in terms of `Parser[T]`s. Writing parsers like this is quite low-level.
* You cannot just rely on the documentation - you will need to read the source to get things done.
* It has a complex symbolic syntax which is not to everybody's taste.
* There are some rough edges to SBT's parsers. For example, you can use multiple `Parser[T]` instances in an InputTask but they are evaluated in reverse order. This is [considered a bug](https://github.com/sbt/sbt/issues/2033).
* It is easy to break your `Parser[T]` with a small change in any of the component `Parser[T]`s. This fragility raises questions about the maintainability of your SBT project. If you have to handle a large number of command line arguments, you may find this approach frustrating.


### Approach 2: scopt

Now I'd like to point out an alternative and more pragmatic approach to parsing command line arguments in SBT. This approach is simpler, more accessible and likely more familiar to developers. This is the approach that I would take next time this problem comes up.

The basic idea is to use the `spaceDelimited: Parser[Seq[String]]` parser to split the entire input string into space-separated strings. Then you can use the simple and widely-used command line argument parsing library [scopt](https://github.com/scopt/scopt) to handle the parsing for you.

This removes the need to maintain a complex hierarchy of SBT parsers, improving maintainability and simplifying your code. Instead, you have a single flat `scopt.OptionParser` that defines your parameters, how to handle them and looks after the low-level parsing details for you.

Let's see what `sbt mkdir` might look like with this approach.

### Step 1: Define MkdirConfig

Firstly we define a data structure with sensible defaults to collect the information from the input. This is much the same as with the previous approach.

```scala
case class MkdirConfig(createIntermediate: Boolean = false,
                       verbose: Boolean = false,
                       mode: String = "0777",
                       directories: Seq[File] = Seq())
```

### Step 2: Define an scopt.OptionParser

Next, we define our `scopt.OptionParser`. This will do most of the parsing work for us.

```scala

lazy val parser = new scopt.OptionParser[MkdirConfig]("mkdir") {
  head("mkdir", "make directories")

  opt[Unit]('p', "create").action((_, c) =>
    c.copy(createIntermediate = true)
  ).text("-p is used to create intermediate directories")

  opt[Unit]('v', "verbose").action((_, c) =>
    c.copy(verbose = true)
  ).text("verbose output")

  opt[String]('m', "mode").action((m, c) =>
    c.copy(mode = m)
  ).text("the directory creation mode")

  arg[File]("<dir>...").unbounded().required().action((x, c) =>
    c.copy(directories = c.directories :+ x)
  ).text("directories to create")

  override def terminate(exitState: Either[String, Unit]): Unit = ()
}
```

### Step 3: Define an InputTask or Command

Just like in the first approach, we need to add an InputTask or Command to your build definition.

Here's an example `InputTask`:

```scala
lazy val mkdir = inputKey[Unit]("make directories with scopt")

mkdir := {
  val args: Parser[Seq[String]] = spaceDelimited("arg").parsed
  parser.parse(args, MkdirConfig()) match {
    case Some(config) => Mkdir.run(config)
    case None => ()
  }
}
```

And here's an example `Command`:

```scala
def mkdirScoptCmd = Command.args("mkdir", parser.usage) { (state, args) =>
  parser.parse(args, MkdirConfig()) match {
    case Some(c) => println(c)
    case None => ()
  }
  state
}

commands ++= Seq(mkdirScoptCmd)
```

And that's it. We're done!

You'll notice that we're not completely doing away with SBT's built in parsers in this approach. We're just limiting their use to the simple `spaceDelimited` parser that splits the user input on whitespace. Once we've done this, `scopt` looks after all of the details for us.

## Thoughts on the _scopt way_

I much prefer this second approach to handling command line arguments in SBT for a number of reasons:

* It's simple and pragmatic - you don't want your parsing code to take up lots of space.
* No need to make low-level parsing decisions.
* You don't need to "look under the hood".
* The `OptionParser` is more robust to modification as the individual parsing components are independant. This helps with maintainability.

I find it hard to see a downside to this approach. Happy parsing!

### Footnotes

[^1]: [Discussion on the SBT mailing list](https://groups.google.com/forum/#!topic/simple-build-tool/vgxkDSgOnlc)
[^2]: [InputTask.scala](https://github.com/sbt/sbt/blob/0.13/main/settings/src/main/scala/sbt/InputTask.scala)
[^3]: [Command.scala](https://github.com/sbt/sbt/blob/0.13/main/command/src/main/scala/sbt/Command.scala)
[^4]: [SBT docs on Parsing Input](http://www.scala-sbt.org/0.13/docs/Parsing-Input.html)
[^5]: [Parsers.scala](https://github.com/sbt/sbt/blob/0.13/util/complete/src/main/scala/sbt/complete/Parsers.scala)
[^6]: [Backhaus-Naur Form grammars](https://en.wikipedia.org/wiki/Backus%E2%80%93Naur_form)

