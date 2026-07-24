+++
title = 'Dependency Injection'
date = 2026-07-22
slug = 'dependency-injection'
resources = ['glossary']
tags = ['dependency-injection', 'testing']
toc = false
+++

**Dependency injection** (DI) is a technique where an object receives the
collaborators it needs from the outside -- typically through its constructor --
instead of constructing or looking them up itself. The object depends on an
_abstraction_ (an `interface`, `protocol`, `trait`, etc) and the concrete
implementation is supplied by the caller that constructs it.

<!--more-->

The point is _inversion of control_: the consumer no longer decides which
concrete dependency it uses. That decision moves up to whoever wires the object
together. This enables two critical things:

1. Better composability, provided the abstraction hierarchy is well-defined, and
2. Better testability, since production code can pass the _real_ implementation,
   whereas unit tests now pass a {{< glossary term="test-double" text="test double" >}}.

Nothing inside the consumer changes between the two.

## Pattern

Dependency-injection requires accepting and working with _abstractions_ instead
of concretions. This generally means that function calls, object members/fields,
etc. are done working with the abstractions rather than the concretions.

Within this, there are two common approaches:

* **Runtime dependency injection**: Done at runtime by passing in concretions
  that implements an abstraction. For functions, this is done through accepting
  the abstraction in the signature instead of the concretion. For _types_, this
  is done through holding onto member fields that reference the abstractions
  instead of the concrete types; generally this form is initialized by
  having the constructor accept abstractions.

  This is the most common form of dependency-injection, and the easiest to
  leverage.

* **Static dependency injection**: Done at _compile-time_ for languages that
  support it. This generally leverages generics or `template`-like syntax.

  Instead of accepting an abstraction as an input/field, the abstraction is a
  _type constraint_ on a statically known type. This form of DI is frequently
  used in machine languages to support more optimal code generation -- since
  constraining instantiations allows the compiler to know exactly which code
  paths will exist.

DI does not require a framework or container. Passing a collaborator as an
argument _is_ dependency injection; the "container" is often just the code in
`main` (or a test's _Arrange_ step) that constructs the object graph. Frameworks
automate that wiring for large graphs, but they are an optimization, not the
pattern itself.

Injected dependencies make substitution and testing easy and keep coupling
explicit. This does come at the cost of frequently requiring more constructor
parameters and wiring code needed to assemble the dependency graph; but with
well-designed abstractions this tradeoff allows more scalability and better
maintainability in the long-run.

{{< note >}}
When creating abstractions for dependency injection, it's important to not just
throw all complicated/external parts in one large interface so that it can be
testable. It's better to create small abstractions that focus solely on a
single responsibility so that you can divide the logical _collaborators_ that
your system requires.
{{< /note >}}

## Example

For an example of **runtime dependency injection**, imagine we have a `Report`
object that needs to record the time. If we hard-code a dependency to the
system clock, we have no way to reliably test the results without skipping or
ignoring the recorded time value. Whereas, if we define a `Clock` abstraction
and rely abstractly on that, we can pass in a fake clock during testing and
ensure deterministic results.

{{< tabs >}}
{{< tab icon="cplusplus" label="C++" >}}

```cpp
// The dependency, expressed as an interface.
class Clock {
public:
  using Time = std::chrono::system_clock::time_point;

  virtual ~Clock() = default;
  virtual auto now() const -> Time = 0;
};

// The consumer receives the dependency; it never creates one.
class Report {
public:
  explicit Report(const Clock& clock) : m_clock(&clock) {}

  auto generate() const -> std::string {
    auto t = m_clock->now();
    // ... build the report using t
  }

private:
  const Clock* m_clock;
};
```

{{< /tab >}}
{{< tab icon="go" label="Go" >}}

```go
// The dependency, expressed as an interface.
type Clock interface {
  Now() time.Time
}

// The consumer holds the dependency; the caller supplies it.
type Report struct {
  clock Clock
}

func NewReport(clock Clock) *Report {
  return &Report{clock: clock}
}

func (r *Report) Generate() string {
  t := r.clock.Now()
  // ... build the report using t
}
```

{{< /tab >}}
{{< tab icon="python" label="Python" >}}

```python
from typing import Protocol
from datetime import datetime

# The dependency, expressed as a protocol.
class Clock(Protocol):
  def now(self) -> datetime: ...

# The consumer receives the dependency; the caller supplies it.
class Report:
  def __init__(self, clock: Clock) -> None:
    self._clock = clock

  def generate(self) -> str:
    t = self._clock.now()
    ...  # build the report using t
```

{{< /tab >}}
{{< tab icon="rust" label="Rust" >}}

```rust
use std::time::SystemTime;

// The dependency, expressed as a trait.
trait Clock {
  fn now(&self) -> SystemTime;
}

// The consumer is generic over the dependency; the caller supplies it.
struct Report {
  clock: Box<Clock>,
}

impl Report {
  fn new(clock: Box<Clock>) -> Self {
    Report { clock }
  }

  fn generate(&self) -> String {
    let t = self.clock.now();
    // ... build the report using t
    todo!()
  }
}
```

{{< /tab >}}
{{< /tabs >}}

This same example can be done using **static dependency injection** in languages
that support it. The difference here is that the `Report<Clock>` becomes a
_concrete_ type that relies on the _concrete_ clock; but within tests can still
be instantiated with `Report<FakeClock>` as needed.

{{< tabs >}}
{{< tab icon="cplusplus" label="C++" >}}

```cpp
// The dependency, expressed as an interface.
concept Clock {
  { now() -> std::same_as<std::chrono::system_clock::time_point> };
};

// The consumer is generic over the dependency, supplied by the caller.
class Report<Clock C> {
public:
  explicit Report(const C& clock) : m_clock(&clock) {}

  auto generate() const -> std::string {
    auto t = m_clock->now();
    // ... build the report using t
  }

private:
  const C* m_clock;
};
```

{{< /tab >}}
{{< tab icon="go" label="Go" >}}

```go
// The dependency, expressed as an interface.
type Clock interface {
  Now() time.Time
}

// The consumer is generic over the dependency, supplied by the caller.
type Report[C Clock] struct {
  clock C
}

func NewReport[C Clock](clock C) *Report {
  return &Report{clock: clock}
}

func (r *Report[C]) Generate() string {
  t := r.clock.Now()
  // ... build the report using t
}
```

{{< /tab >}}
{{< tab icon="rust" label="Rust" >}}

```rust
use std::time::SystemTime;

// The dependency, expressed as a trait.
trait Clock {
  fn now(&self) -> SystemTime;
}

// The consumer is generic over the dependency; the caller supplies it.
struct Report<C: Clock> {
  clock: C,
}

impl <C: Clock> Report<C> {
  fn new(clock: C) -> Self {
    Report { clock }
  }

  fn generate(&self) -> String {
    let t = self.clock.now();
    // ... build the report using t
    todo!()
  }
}
```

{{< /tab >}}
{{< /tabs >}}

For languages like C++ and Rust, this also allows for the template inputs to
have static functions that don't rely on a concrete instantiation -- which is a
very powerful way to abstract global/C-style APIs, since this can allow you to
do things like abstract OpenGL or OS-system calls.

For example, abstracting part of the OpenGL API can be done like:


{{< tabs >}}
{{< tab icon="cplusplus" label="C++" >}}

```cpp
concept OpenGL {
  { glClear(::GLbitfield) };
  // ... other OpenGL funcs
};

class Renderer<OpenGL OGL> {
public:
  // ...

  auto render() const -> std::expected<void> {
    // No instance; everything is static
    OGL::glClear(...)

    // ... render the rest of the frame
  }

  // No field necessary
};
```

{{< /tab >}}
{{< tab icon="rust" label="Rust" >}}

```rust
trait OpenGL {
  fn clear(gl::bitfield);
  // ... other OpenGL funcs
}

// The consumer is generic over the dependency; the caller supplies it.
struct Renderer<OGL: OpenGL> {}

impl <OGL: OpenGL> Report<OGL> {

  fn render(&self) -> String {
    OGL::clear()
    // ... render the rest of the frame

    todo!()
  }
}
```

{{< /tab >}}
{{< /tabs >}}

What this allows for is the real code to be constructed with `Renderer<OpenGL>`,
which compiles down to exact OpenGL calls -- whereas test code can use a
`Renderer<StubOpenGL>` implementation that enables controllable behavior just
for tests.

## References

* {{< link
  url="https://martinfowler.com/articles/injection.html"
  text="Inversion of Control Containers and the Dependency Injection pattern"
  icon="link"
  hover="Martin Fowler" >}} -- Fowler's original article distinguishing DI
  from service locators and covering the injection variants.
* {{< link
  url="https://en.wikipedia.org/wiki/Dependency_injection"
  text="Dependency injection"
  icon="link"
  hover="Wikipedia" >}} -- overview of the pattern, terminology, and common
  criticisms.
* {{< link
  url="https://en.wikipedia.org/wiki/Dependency_inversion_principle"
  text="Dependency inversion principle"
  icon="link"
  hover="Wikipedia" >}} -- the SOLID principle DI is most often used to
  satisfy: depend on abstractions, not concretions.
