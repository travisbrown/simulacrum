simulacrum
==========

[![Gitter](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/mpilquist/simulacrum?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

Type classes rock. Alas, their encoding in Scala requires a lot of boilerplate, which doesn't rock. There is inconsistency between projects, where type classes are encoded differently. There is inconsistency within projects, where object-oriented forwarders (aka. ops, syntax) accidentally differ in exact parameter lists or forwarders are missing where they are expected to be. Even in disciplined teams, the bike-shedding opportunities alone are a source of lost productivity.

This project addresses these concerns by introducing first class support for type classes in Scala 2.11. For example:

```scala
import simulacrum._

@typeclass trait Semigroup[A] {
  @op("|+|") def append(x: A, y: A): A
}
```

Given this definition, something similar to the following is generated at compile time:

```scala
trait Semigroup[A] {
  def append(x: A, y: A): A
}

object Semigroup {
  def apply[A](implicit tc: Semigroup[A]): Semigroup[A] = tc

  trait Ops[A] {
    def self: A
    def typeClassInstance: Semigroup[A]
    def |+|(y: A): A = semigroup.append(self, y)
  }

  trait ToSemigroupOps {
    implicit def toSemigroupOps[A](a: A)(implicit sg: Semigroup[A]): Ops[A] = new Ops[A] {
      def self = a
      def semigroup = sg
    }
  }

  object ops extends ToSemigroupOps
}
```

Subtyping of type classes is supported (e.g., you can define a `Monoid` type class that extends `Semigroup` and the generated code adapts accordingly). Higher kinds are also supported -- specifically, type classes that are polymorphic over type constructors, like `Functor`. The current implementation only supports unary type constructors, but support for binary type constructors is planned.

This allows usage like:

```scala
implicit val semigroupInt: Semigroup[Int] = new Semigroup[Int] {
  def append(x: Int, y: Int) = x + y
}

import Semigroup.ops._
1 |+| 2 // 3


```

See [the examples](src/test/scala/simulacrum/examples.scala) for more.

## Usage

This project currently only supports Scala 2.11. The project is based on macro paradise. To use the project, add the following to your build.sbt:

```scala
addCompilerPlugin("org.scalamacros" % "paradise" % "2.1.0-M5" cross CrossVersion.full)

libraryDependencies += "com.github.mpilquist" %% "simulacrum" % "0.1.0"
```

To use the latest SNAPSHOT version, add the following:

```scala
resolvers += "Sonatype Public" at "https://oss.sonatype.org/content/groups/public/"

addCompilerPlugin("org.scalamacros" % "paradise" % "2.1.0-M5" cross CrossVersion.full)

libraryDependencies += "com.github.mpilquist" %% "simulacrum" % "0.2.0-SNAPSHOT"
```

Macro paradise must exist in projects which use `@typeclass`, but code that dependencies on the generated type classes do not need macro paradise.

Feedback is much appreciated. The generated code is a result of working with project leads of a variety of open source projects that use type classes. However, there's certainly room for improvement, so please open issues or PRs containing feedback.

## Known Limitations

 - Only type classes that abstract over a proper type or a unary type constructor are currently supported. This will be extended to binary type constructors in the future, and perhaps n-ary type constructors.
 - When defining a type class as a subtype of another type class, and defining an abstract member of the super type concretely in the sub type, the `override` keyword must be used. For example, defining `map` in terms of `flatMap` requires `override def map[A, B](...)`.
 - Type parameters to type classes may not be specialized or otherwise annotated. We hope to address this soon.
 - Simulacrum annotations may not be name aliased via imports. That is, you cannot do:

```scala
  import simulacrum.{ op => foo }
  @typeclass Bar[A] {
    @foo def bar(a: A, a: A): A
  }
```

 - See the GitHub issues list for other known limitations and please open issues for any other limitations you encounter. If you suspect a problem, it may be helpful to run with the `simulacrum.trace` system property (e.g., `sbt -Dsimulacrum.trace compile`), which adds a significant amount of logging to the compiler output.
