---
marp: true
---

# Implicits in Scala 2

*With great power comes great responsibility*

---

## What are implicits?


Implicits powerful language feature that enable

* cleaner syntax
* design of intuitive APIs
* dependency injection
* type classes 

Implicits can also make a code base much harder to read and introduce unintended behaviour. Use with care.

---

## Implicit `vals`

Defining an implicit `val` introduces the `val` into implicit scope.

```scala
@ implicit val x: Int = 5
x: Int = 5
```

*WARNING: Don't do this for common types such as `Int`, `String`, `Option` etc.*

---

## Implicit parameters

Functions may also have implicit parameters.

*"Explicit" parameter*

```scala
@ def showInt(n: Int): String = s"The Int is $n"
defined function showInt

@ showInt(10)
res2: String = "The Int is 10"
```

*Implicit parameter*
```scala
@ def showInt(implicit n: Int): String = s"The implicit Int is $n"
defined function showInt

@ showInt // uses implicit val x: Int = 5 previously defined!
res4: String = "The implicit Int is 5"
```

---

## Ambiguous implicit values

Only one implicit `val` of a given type can be in scope.

```scala
@ implicit val x2: Int = 15
x2: Int = 15

@ showInt
cmd6.sc:1: ambiguous implicit values:
 both value x in object cmd0 of type => Int
 and value x2 in object cmd5 of type => Int
 match expected type Int
val res6 = showInt
           ^
Compilation Failed
```

---

## Implicit conversions

It is also possible to define implicit functions. These can be used for implicit type conversions. For example, the implicit function `optionToString` allows us to use option strings as if they were strings.

```scala
@ implicit def optionToString(option: Option[String]): String = 
    option.getOrElse("")

@ Some("cat").toUpperCase
res9: String = "CAT"
```

This works by the compiler inserting the implicit function when required to make the types line up.

```scala
@ optionToString(Some("cat")).toUpperCase
res10: String = "CAT"
```

*WARNING: Don't do this for common types such as `Int`, `String`, `Option` etc.*

---

## Implicit classes (Extension methods)

Implicit classes are used to defined extension methods to provide cleaner syntax. Extension methods are also discoverable when using an IDE which improves user experience.

```scala
@ implicit class OptionStringOps(private val option: Option[String]) extends AnyVal {
    def containsIgnoreCase(elem: String): Boolean =
      option.exists(_.equalsIgnoreCase(elem))
  }
defined class OptionStringOps

@ Some("Cat").containsIgnoreCase("cat")
res12: Boolean = true
  // if defined as a simple function -
  // containsIgnoreCase(Some("Cat"), "cat")
```

Similar to implicit functions, implicit classes are inserted by the compiler when required to provide extension methods.
```scala
@ OptionStringOps(Some("Cat")).containsIgnoreCase("cat")
res13: Boolean = true
```

---

## Type equality

`implicit ev: A =:= String` means *implicit evidence that `A` is of type `String`*

```scala
@ implicit class OptionOps[A](private val option: Option[A]) extends AnyVal {
    def containsOneOf(elem: A, elems: A*): Boolean =
      option.exists(e => (elem +: elems).contains(e))

    def containsIgnoreCase(elem: String)(
      implicit ev: A =:= String): Boolean =
        option.exists(_.equalsIgnoreCase(elem))
  }
defined class OptionStringOps

@ Some(1).containsOneOf(1, 2, 3)
res44: Boolean = true

@ Some("durian").containsIgnoreCase("DURIAN")
res15: Boolean = true
```

---

## Type classes

* Used for ad hoc polymorphism
* Alternative to inheritance
* Basis for functional programming in Scala

---

## Semigroup type class

* A semigroup is a set with an associative binary operator

```scala
@ "a" + ("b" + "c") == ("a" + "b") + "c"
res18: Boolean = true
@ 1 + (2 + 2) == (1 + 2) + 3
res19: Boolean = false
```

* In less technical language, semigroups are used for smashing things of the same type together.

---

## Semigroup type class

We can define `Semigroup` using traits,

```scala
@ trait Semigroup[A] {
    def combine(x: A, y: A): A
  }
defined trait Semigroup
```

... and provide `Semigroup` implementations for `Int` and `String` -

```scala
@ val intSemigroup = new Semigroup[Int] {
    def combine(x: Int, y: Int): Int = x + y
  }

@ val strSemigroup = new Semigroup[String] {
    def combine(x: String, y: String): String = x + y
  }
```

---

## Semigroup type class

Putting the `Semigroup` implementations to use.

```scala
@ strSemigroup.combine("my ", "cat")
res24: String = "my cat"

@ def describe[A](x: A, y: A)(semigroup: Semigroup[A]): String =
    s"The result of combining $x and $y is ${semigroup.combine(x, y)}"
defined function describe

@ describe(5,6)(intSemigroup)
res26: String = "The result of combining 5 and 6 is 11"
```

*Note that implicit have not been used.*


---

## Semigroup type class

Full implementation of the `Semigroup` type class using implicits.

```scala
@ trait Semigroup[A] {
    def combine(x: A, y: A): A
  }

@ object Semigroup {
    def apply[A](implicit ev: Semigroup[A]) = ev // the summoner

    // Semigroup type class instance for Int
    implicit val intSemigroup = new Semigroup[Int] {
      def combine(x: Int, y: Int): Int = x + y
    }

    // Semigroup type class instance for String
    implicit val strSemigroup = new Semigroup[String] {
      def combine(x: String, y: String): String = x + y
    }

    // Semigroup type class instance for Option[A]
    implicit def optionSemigroup[A](implicit ev: Semigroup[A]) = new Semigroup[Option[A]] {
      def combine(x: Option[A], y: Option[A]): Option[A] =
        (x, y) match {
          case (Some(x_), Some(y_)) => Some(ev.combine(x_, y_))
          case (Some(x_), _) => Some(x_)
          case (_, Some(y_)) => Some(y_)
          case _ => None
        }
    }

    // defines |+| operator using an implicit class
    implicit class SemigroupOps[A](private val x: A) extends AnyVal {
      def |+|(y: A)(implicit ev: Semigroup[A]): A = ev.combine(x, y)
    }
  }
```

---

## Semigroup type class usage

`Semigroup` instance is provided implicitly.

```scala
@ def describe[A](x: A, y: A)(implicit semigroup: Semigroup[A]): String =
    s"The result of combining $x and $y is ${semigroup.combine(x, y)}"
```

Since we implemented `Semigroup` instances for both `Int` and `String`, we can `describe` the result of combining `Int`s and `String`s.

```scala
@ import Semigroup._ // make sure type class instances are in scope
import Semigroup._
@ describe(9,0)
res32: String = "The result of combining 9 and 0 is 9"
@ describe("my", "cat")
res33: String = "The result of combining my and cat is mycat"
```

---

## Semigroup type class usage

We can also `describe` the result of combining `Option`s assuming we have a `Semigroup` instance for the inner type.

```scala
@ describe(Option(9), Option(0))
res34: String = "The result of combining Some(9) and Some(0) is Some(9)"
@ describe(Option(9), None)
res35: String = "The result of combining Some(9) and None is Some(9)"
@ describe(None, Option(0))
res36: String = "The result of combining None and Some(0) is Some(0)"
```

---

## Syntactic sugar for type constraints

 It can often be clearer to use a type constraint. This is simply syntactic sugar for implicit parameters. In the example below, the type constraint, `A: Semigroup`, is the same as providing `implicit ev: Semigroup[A]`.

```scala
@ def describe[A: Semigroup](x: A, y: A): String =
    s"The result of combining $x and $y is ${x |+| y}"
```