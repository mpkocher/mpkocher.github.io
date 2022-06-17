---
title: Creating CLI tools leveraging ZIO and Decline using scala-cli
date: 2022-06-02 20:51:36
tags:
    - scala
    - "scala-cli"
    - zio
---
Here's a quick overview and demonstration of kicking the tires on ZIO 1.x + [decline](https://github.com/bkirwi/decline) (commandline argument parser) using `scala-cli` (version 0.1.6) to build commandline tools.

- https://scala-cli.virtuslab.org
- https://zio.dev
- https://github.com/bkirwi/decline
- [Complete Gist](https://gist.github.com/mpkocher/e7b23d082a9ce9cba6e5ec3337658e76)

## High Level Points

- `scala-cli` is a new tool in active development that aims to replace the current `scala` tool. 
- `scala-cli` enables running scripts, loading REPL, compiling, testing, packaging (amongst other features) for "simple" (flat) projects.
- `scala-cli` enables a very Python-ish workflow and integrates with your text editor well. A REPL driven approach can be used using `scala-cli repl File.scala`.
- `scala-cli package` enables [creating an executable](https://scala-cli.virtuslab.org/docs/cookbooks/scala-package) from your main class. This is very useful. No need to deal with Python's conda, pipenv, poetry, etc... for setting up an env. Rsync/scp your packaged tool to another server and your good to go (provided the java versions are compatible).
- Building a commandline tool using ZIO was a useful exercise to kick the tires on ZIO and understand the ZIO 1.x effect system and learn how to compose computation in ZIO.
- `decline` is a CLI parser library using `cats`. It has a very elegant mechanism of composing options/commands. 
- The default `decline` interface had some unexpected behaviors with how `--help` was handled and how errors were handled/mapped to exit codes.


## Specifics

Using `scala-cli setup-ide .` will generate the necessary files for the LSP server for your text editor.

- https://scala-cli.virtuslab.org/docs/commands/setup-ide

Using `directives` `//>` in the top of your scala file, you can define the scala version, library versions, etc...

- https://scala-cli.virtuslab.org/docs/reference/directives

For example, `Declined.scala`

```
//> using platform "jvm"
//> using scala "2.13.8"
//> using lib "dev.zio::zio:1.0.14"
//> using lib "com.monovore::decline:2.2.0"
//> using mainClass "DeclinedApp"
```

Using `zio.App` we can define our `main` by overriding `run`.

```scala
object DeclinedApp extends zio.App {
   override def run(args: List[String]): ZIO[ZEnv, Nothing, ExitCode] = ???
} 
```

Using `decline`, CLI options and arguments are defined using `Opts`.

For example:

```scala
import com.monovore.decline._
import cats.implicits._

val nameOpt = Opts.option[String]("user", help = "User name", "u")

val alphaOpt = Opts
    .option[Double]("alpha", help = "Alpha Filtering", "a")
    .withDefault(1.23)
```

These `Opts` compose in interesting ways, specifically with `ZIO` effects.

```scala
val versionOpt: Opts[RIO[Console, Unit]] = Opts
    .flag(
      "version",
      "Show version and Exit",
      "v",
      visibility = Visibility.Partial
    )
    .orFalse
    .map(_ => putStrLn(VERSION))
```

You can compose options together using `mapN` to define an "action" or "command" that would need multiple commandline options/args.

For example to define a `mainOpt` that uses `name, alpha, force`:

```scala
val nameOpt = Opts.option[String]("user", help = "User name", "u")

val alphaOpt = Opts
    .option[Double]("alpha", help = "Alpha Filtering", "a")
    .withDefault(1.23)

// Add for testing failure modes
val forceOpt = Opts.flag("fail", help = "Manually trigger a failure").orFalse

// Now define our core
val mainOpt: Opts[RIO[Console, Unit]] =
    (nameOpt, alphaOpt, forceOpt).mapN[RIO[Console, Unit]] {
      (name, alpha, force) =>
        if (force)
          ZIO.fail(
            new Exception(s"Manually FAIL triggered by $name! alpha=$alpha")
          )
        else putStrLn(s"Hello $name. Running with alpha=$alpha")
    }

```
In addition to composing using `mapN`, there's `orElse` which enables composing "actions". For example, enabling `--version` to run (if provided) *or* run the "main" application.

```scala
val runOpt = versionOpt orElse mainOpt
```
These composed `Opts` can be used in a `Command` that will handled `--help` and be central point where `Command.parse` can be called. 

```scala
val command: Command[RIO[Console, Unit]] = Command[RIO[Console, Unit]](
    name = "declined",
    header = "Testing decline+zio"
  )(versionOpt orElse mainOpt)
```

Bridging the ouptut of `Command.parse` with ZIO requires a little glue and some special attention to deal with the error cases. `Command.parse` will return an `Either[Help, T]`. The left of `Either` being used as "help+errors" container is a bit of friction point because `--help` triggers the left of the `Either`. `Help.errors` will return a non-empty list of errors (if there are parse errors). 

```scala
def effect(arg: List[String]): ZIO[Console, Throwable, Unit] = {
    command.parse(arg) match {
      case Left(help) => // a bit odd that --help returns here
        if (help.errors.isEmpty) putStrLn(help.show)
        else IO.fail(new Exception(s"${help.errors}"))
      case Right(value) => value.map(_ => ())
    }
}
```

And finally, wire a call to `run` and make sure errors are written to stderr and a non-zero exit code is returned during failure cases. 

```scala
override def run(args: List[String]): ZIO[ZEnv, Nothing, ExitCode] =
    effect(args).map(_ => ExitCode.success).catchAll { ex =>
      for {
        _ <- putStrLnErr(s"Error ${ex}").orDie
      } yield ExitCode.failure
}
```

Running can be done using `scala-cli run Declined.scala`

```bash
$> scala-cli run Declined.scala  -- --version
0.1.0
```

Or by packaging the app and running the generated exe.

```bash
scala-cli package --jvm 14 Declined.scala
Wrote /Users/mkocher/path/to/DeclinedApp, run it with
  ./DeclinedApp
```

Running

```bash
$> ./DeclinedApp --help
Usage: declined [--user <string> [--alpha <floating-point>] [--fail]]

Testing decline+zio

Options and flags:
    --help
        Display this help text.
    --version, -v
        Show version and Exit
    --user <string>, -u <string>
        User name
    --alpha <floating-point>, -a <floating-point>
        Alpha Filtering
    --fail
        Manually trigger a failure
```

And a few smoke tests.


```bash
$> ./DeclinedApp --user Dave --alpha 3.14
Hello Dave. Running with alpha=3.14
$> echo $?
0
$ ./DeclinedApp --user Dave --alpha dragon
Error java.lang.Exception: Invalid floating-point: dragon
$> ./DeclinedApp --user Dave --alpha 3.14 --dragon
Error java.lang.Exception: Unexpected option: --dragon
$>echo $?
1
```

## Summary and Final Comments

- `scala-cli` is a __very__ promising addition for the Scala community.
- `scala-cli` changed my workflow. This new workflow was closer to how I would work in Python.
- I really like ZIO's core composablity ethos, however, it does have a learning curve.
- `Decline`'s `Command[T]` design allows for intergrate with ZIO pretty seemlessly. 
- Misc "scrappy" CLI tools that I would typically write in Python, I could easily write in Scala. 
