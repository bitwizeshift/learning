+++
title = 'Sealed Interface'
date = 2026-07-26
slug = 'sealed-interface'
aliases = ['/glossary/sealed-interface']
resources = ['glossary']
tags = ['sealed-interface', 'design']
toc = false
+++

A **sealed interface** is an interface whose set of permitted implementations is
fixed. Where an ordinary interface is open -- any type, in any package, can
implement it -- a sealed one closes the set so that the implementations are
always fixed and known _by_ the `interface`. Some languages handle this with
special keywords/syntax, others enable it through privacy semantics, or even
exhaustive `switch`/`match`es.

<!--more-->

## Why it matters

Sealing answers one question: _who is allowed to implement this interface?_
Normal interfaces can be implemented by anyone, either the library author, or
a consumer of the library -- provided the interface is namable. A sealed one
changes these semantics so that only a fixed, well-known set of types may
implement the interface. This leads to two critical improvements:

1. **Control over the invariants**:  When the author of the interface names the
   complet set of implementers, any property that is supposed to hold across
   that set can be _guaranteed_ by the library author. This enforces that no
   foreign package definitions can slip in a type that quietly or
   unintentionally violates it. This makes it much easier to guarantee that
   every type is
   {{< glossary term="liskov-substitution-principle" text="substitutable" >}},
   even if an interface has implied contract behaviors.

2. **Exhaustive semantics**: Code htat consumes the interface can more easily
   be implemented against the _entire set_ of implementations and have better
   assurance that the family will not grow unexpectedly.

## Example

Consider a `Shape` interface with a fixed set of implementers -- `Circle` and
`Rectangle` -- that no code outside the defining unit is allowed to extend. Each
language seals the set differently, but the guarantee is the same: these are the
only shapes there will ever be.

{{< tabs >}}
{{< tab icon="go" label="Go" >}}

```go
package geometry

import "math"

// Shape is sealed by the unexported marker method: only types declared in this
// package can satisfy it, so the set of shapes is fixed and known.
type Shape interface {
  Area() float64
  isShape()
}

type Circle struct{ Radius float64 }

type Rectangle struct {
  Width  float64
  Height float64
}

func (Circle) isShape()    {}
func (Rectangle) isShape() {}

func (c Circle) Area() float64    { return math.Pi * c.Radius * c.Radius }
func (r Rectangle) Area() float64 { return r.Width * r.Height }
```

{{< /tab >}}
{{< tab icon="rust" label="Rust" >}}

```rust
mod sealed {
  pub trait Sealed {}
}

// Shape can only be implemented by types that implement the private Sealed
// supertrait, which no outside crate can name.
pub trait Shape: sealed::Sealed {
  fn area(&self) -> f64;
}

pub struct Circle {
  pub radius: f64,
}

pub struct Rectangle {
  pub width: f64,
  pub height: f64,
}

impl sealed::Sealed for Circle {}
impl sealed::Sealed for Rectangle {}

impl Shape for Circle {
  fn area(&self) -> f64 {
    std::f64::consts::PI * self.radius * self.radius
  }
}

impl Shape for Rectangle {
  fn area(&self) -> f64 {
    self.width * self.height
  }
}
```

{{< /tab >}}
{{< tab icon="java" label="Java" >}}

```java
sealed interface Shape permits Circle, Rectangle {
  double area();
}

record Circle(double radius) implements Shape {
  public double area() {
    return Math.PI * radius * radius;
  }
}

record Rectangle(double width, double height) implements Shape {
  public double area() {
    return width * height;
  }
}
```

{{< /tab >}}
{{< tab icon="kotlin" label="Kotlin" >}}

```kotlin
import kotlin.math.PI

sealed interface Shape {
  fun area(): Double
}

data class Circle(val radius: Double) : Shape {
  override fun area() = PI * radius * radius
}

data class Rectangle(val width: Double, val height: Double) : Shape {
  override fun area() = width * height
}
```

{{< /tab >}}
{{< /tabs >}}

Java and Kotlin state the seal in the language: `permits` -- explicit in Java,
inferred from the same file in Kotlin -- fixes the list, and a class outside it
will not compile as a `Shape`. Go seals through privacy: the unexported `isShape`
marker can only be satisfied inside the package, so no importer can add a `Shape`.
Rust's _sealed trait_ pattern does the same with a public trait bounded by a
private one -- an outside crate cannot name the supertrait, so it cannot implement
`Shape`.

Whichever mechanism is used, the permitted implementations form a closed set that
can be drawn directly.

## When not to seal

Sealing is the wrong default whenever the set of implementations is meant to be
_open_. Extension points like plugins, strategies, drivers, storage backends,
codecs, etc. exist precisely so that code outside the defining package can
supply new implementations without the original being changed. This follows the
{{< glossary term="open-closed-principle" text="open/closed principle" >}} since
it's open for modification. Sealing an interface disables that extension point.

Sealing is only the right choice when a type hierarchy is intentionally bounded
and **exhaustive**, having no logical reason to ever be extensible. Examples
of this are things with a fully-known family, like an AST, a protocol message,
etc. -- anything where an implementation would logically be a defect. In these
cases, sealing the interface is likely the right choice.

## References

* {{< link
    url="https://openjdk.org/jeps/409"
    text="JEP 409: Sealed Classes"
    icon="link"
    hover="OpenJDK" >}} -- the specification that introduced `sealed` and
  `permits` to Java, restricting which classes may implement an interface.
* {{< link
    url="https://kotlinlang.org/docs/sealed-classes.html"
    text="Sealed classes and interfaces"
    icon="link"
    hover="Kotlin" >}} -- Kotlin's `sealed` modifier and the file/module scope
  that fixes the set of implementers.
* {{< link
    url="https://rust-lang.github.io/api-guidelines/future-proofing.html"
    text="Sealed traits protect against downstream implementations"
    icon="link"
    hover="Rust API Guidelines" >}} -- the sealed-trait pattern and when to close
  a trait against foreign implementers.
