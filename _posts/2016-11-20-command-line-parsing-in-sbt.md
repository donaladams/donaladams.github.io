---
layout: post
title: Command Line Parsing in SBT
date: 2016-11-23 12:00:00 +0100
categories: blog
---

I've been working with Scala for the last couple of years and like many (if not most) Scala developers, I've had my ups and downs with SBT. SBT is a powerful tool but it's easy to lose your footing.

During our most recent hackweek at [Nitro](http://www.gonitro.com) I worked on an SBT plugin with some colleagues to help us create and manage our build pipelines in Jenkins. This was the first time I needed to handle proper command line input in SBT.

In this post, we'll look at implementing a command line tool in sbt, making use of its built-in parsing infrastructure to handle command line arguments. If you make it to the end, we'll also suggest an alternative, potentially more pragmatic approach than that suggested in the SBT documentation.

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

`InputTask` and `Command` are quite similar and could both be used for `sbt mkdir`. The prevailing advice is to use `InputTask` as a first choice as `Command` is lower level, allowing you to actually modify the state of the build [^1]. Whichever you choose to use, from the point of view of parsing command line arguments, the approach will be the same. Both `Command` and `InputTask` require a `Parser[T]` to transform the command line input into something useful.

Here's a simplified definition of `InputTask`[^2] and the signature of one of the constructors for `Command`[^3]:

```scala
final class InputTask[T] private (val parser: State => Parser[Task[T]]) {
    ...
}

object Command {
    def apply[T](name: String, help: Help = Help.empty)(parser: State => Parser[T])(effect: (State, T) => State): Command
}

```

The point of similarity between these two is the argument `parser: State => Parser[_]`.

`Parser[T]` is the tool that SBT provides to, as you might have guessed, parse command line input[^4]. `Parser[T]` defines the low-level methods that you would need to use directly if you were defining a new `Parser[T]` implementation. Generally, you won't need to do this. Instead, SBT provides a number of low-level parsers for common scenarios in `sbt.complete.Parsers`[^5]. You will generally combine these together to make more useful `Parser[T]` instances using the combinators in `RichParser` which implicitly enriches instances of `Parser[T]`.

If you decide that you need to use these parsers in your SBT project, you should definitely try to familiarise yourself with both `sbt.complete.Parser` and `sbt.complete.Parsers`.

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

You're free to choose which suits your particular situation better. Whichever way you go, you will need a `Parser[MkdirCommand]` to handle the input.

## Grammar Time

A good place to start when working with parsers is with a grammar. This describes the input that the parser will accept. This might sound
daunting but it's really not so bad - it's helpful to guide our implementation, even if it's not completely formal.

Here's a possible BNF-like grammar [^6] for `mkdir`. This borrows from `man chmod` which helpfully provides a grammar for the "symbolic mode" used in `mkdir`.

{% highlight c %}

// top-level rule defining mkdir cmd
cmd 		::= mkdir [options] directories
directories     ::=  [directory_name ...] directory_name
// command line options
options 	::= mode_option | [flag_options ...] flag_options

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
flag_options 	::= - [flag ...] flag
flag 		::= v | p

{% endhighlight %}

## Parser[MkdirCommand]

`MkdirCommand` is the data structure used to collect the information we want from `sbt mkdir`'s command line arguments.

```scala
case class MkdirCommand(verbose: Boolean = false,
                        createIntermediate: Boolean = false,
                        mode: Option[Mode] = None,
                        directories: Seq[File] = Seq())
```

Now that we have our grammar in place, writing the parsers is (theoretically) straightforward.

The approach will be to create low-level parsers for the "terminals" in the grammar (those values that are only found on the right-hand side of the rules) using SBT's built in parsers.

Then we will build higher-level parsers by combining these until we get a `Parser[MkdirCommand]` that handles the entire `sbt mkdir` input.

## Parsing directories

Without a parser for handling directory names, `mkdir` wouldn't be much use so let's start here.

Here's a solution that introduces us to a number of features of SBT parsers.

```scala
val directory: Parser[File] = StringBasic.map(p => new File(p))
val directories: Parser[Seq[File]] = (Space ~> directory)+
```

The `directory` parser uses a built-in parser `StringBasic` to match a String in the user input. We then map over this `Parser[String]` to transform it into
a `Parser[File]` which is the type we want to use to represent the directories in `MkdirCommand`. This parser is equivalent to the `directory_name` terminal in our grammar.

`mkdir` accepts a sequence of directories so we need to define `val directories: Parser[Seq[File]]` that repeats our `directory` parser one or more times. This corresponds to this rule in our grammar: `directories ::=  [directory_name ...] directory_name`.

This example introduces serveral other new symbols:

* `Space` - this a built-in parser that matches one or more whitespace characters
* `~>` - this is used to combine parsers in sequence, returning the value to the right of `~>`, in this case the File representing the directory.
* `+` - match the parser one or more times

## Parser[Mode]

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

First, here's a potential solution for parsing "absolute modes". The approach is the same as before - we start with a terminal parser `val octalDigit: Parser[Char]` and build larger parsers from it using parser combinators. One thing to point out in this example is the use of `def chars(legal: String): Parser[Char]`. This comes from `sbt.complete.Parser` and parses a single Char if it is found in the provided string of legal characters.

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

Next up is the "symbolic" mode parser. The structure of this is more complicated but we have our grammar to guide us. Again, the process is the same - start at the terminals and work your way up.

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

## Wrapping it up

To complete our `Parser[MkdirCommand]` we still need to define a parser for the remaining boolean flags (`-p` and `-v`) and then combine this, our mode parser (`Parser[Mode]`) and our directories parser (`Parser[Seq[File]]`) into our mkdir parser (`Parser[MkdirCommand]`).

The approach is basically the same as what we have seen before so it won't add much to the discussion to show this here. If you are interested in seeing this explicitly or trying out the full `Parser[MkdirCommand]`, the full source code for these examples is [here](https://github.com/epileptic-fish/sbt-parsers).

## Conclusion

SBT's parser infrastructure is powerful and working with grammars and parsers can be fun. I hope that this post is helpful to anyone starting down this path. However, I would also like to make clear that it is not without its problems. Here are some caveats that I would like add.

* Constructing parsers like this quite work-intensive. You may need to write a lot of `Parser[T]`s to get what you want done.
* It can be difficult to express exactly what you want in terms of `Parser[T]`s. Writing parsers like this is quite low-level.
* You cannot just rely on the documentation - you will need to read the source to get things done.
* There are some rough edges to SBT's parsers. For example, you can use multiple `Parser[T]` instances in an InputTask but they are evaluated in reverse order. This is [considered a bug](https://github.com/sbt/sbt/issues/2033).
* It is easy to break your `Parser[T]` with a small change in any of the component `Parser[T]`. This fragility raises questions about the maintainability of your SBT project.

## Epilogue: An alternative approach

Finally, I'd like to point out an alternative and more pragmatic approach to parsing command line arguments in SBT. This approach is much simpler, more accessible and likely more familiar to developers.

The basic idea is to use the `spaceDelimited: Parser[Seq[String]]` parser to split the entire input string into space-separated strings. Then you can use the simple and widely-used command line argument parsing library [scopt](https://github.com/scopt/scopt) to handle the parsing for you.

This removes the need to maintain a complex hierarchy of SBT parsers, presumably improving maintainability. I would suggest starting with this approach and only taking on the complexity of building your own `Parser`s if you find it necessary.

Here is what `sbt mkdir` might look like with this approach. Firstly, we define our `scopt.OptionParser` and data structure for holding the results:

```scala
case class MkdirConfig(createIntermediate: Boolean = false,
                       verbose: Boolean = false,
                       mode: String = "0777",
                       directories: Seq[File] = Seq())


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

  help("help").text("prints this usage text")

  arg[File]("<dir>...").unbounded().required().action((x, c) =>
    c.copy(directories = c.directories :+ x)
  ).text("directories to create")

  override def terminate(exitState: Either[String, Unit]): Unit = ()
}
```

Then we hook this up in our `InputTask`:

```scala
lazy val mkdir = inputKey[Unit]("make directories with scopt")

mkdir := {
  parser.parse(args.evaluated, MkdirConfig()) match {
    case Some(config) => Mkdir.run(config)
    case None => ()
  }
}
```

or `Command`:

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

### Footnotes

[^1]: [Discussion on the SBT mailing list](https://groups.google.com/forum/#!topic/simple-build-tool/vgxkDSgOnlc)
[^2]: [InputTask.scala](https://github.com/sbt/sbt/blob/0.13/main/settings/src/main/scala/sbt/InputTask.scala)
[^3]: [Command.scala](https://github.com/sbt/sbt/blob/0.13/main/command/src/main/scala/sbt/Command.scala)
[^4]: [SBT docs on Parsing Input](http://www.scala-sbt.org/0.13/docs/Parsing-Input.html)
[^5]: [Parsers.scala](https://github.com/sbt/sbt/blob/0.13/util/complete/src/main/scala/sbt/complete/Parsers.scala)
[^6]: [Backhaus-Naur Form grammars](https://en.wikipedia.org/wiki/Backus%E2%80%93Naur_form)

