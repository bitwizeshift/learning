+++
title = 'Dummy'
date = 2026-07-24
slug = 'dummy'
resources = ['glossary']
tags = ['dummy', 'test-double', 'testing']
toc = false
+++

A **dummy** is a {{< glossary term="test-double" text="test double" >}} that
exists only to fill a required parameter on a code path that never uses it. It
is the simplest member of the family: it carries no behavior, returns nothing
meaningful, and is never configured or asserted on -- a placeholder passed purely
to satisfy a signature.

<!--more-->

## How a dummy works

A dummy is {{< glossary term="dependency-injection" text="injected" >}} in place
of a real collaborator, just like any other double, but the resemblance ends
there. Where a {{< glossary term="stub" text="stub" >}} hands back answers the
unit goes on to use, a dummy is handed to a unit that never touches it on the
path under test. A constructor demands a `Notifier`; the method you are testing
rejects its input before any notification would be sent -- so the parameter still
has to be filled, but with something that does nothing.

Because the unit never calls it, a dummy needs no canned values and records
nothing. Often it is an empty implementation; some teams make it throw if any
method is called, turning "this path does not use the collaborator" from an
assumption into an enforced invariant. Either way the dummy stays out of the
assertions entirely -- the test checks the unit's own result.

## The tradeoff

A dummy keeps a test honest about its scope. Passing a do-nothing (or
deliberately exploding) collaborator documents that the behavior under test does
not depend on it, and there is nothing to configure or maintain -- it is the
cheapest double there is.

The catch is what a proliferation of dummies tends to signal. If a unit forces
every test to supply collaborators it does not use, the class may be taking on
dependencies that belong elsewhere, or doing enough separate jobs that its
constructor has outgrown any single path. Frequent dummies are a nudge to split
responsibilities so each unit receives only what it actually needs. And a dummy
is only ever correct while the path leaves the collaborator untouched: the moment
the unit reads from it, you need a {{< glossary term="stub" text="stub" >}} or
{{< glossary term="fake" text="fake" >}}; the moment the call itself is the
behavior, you need a {{< glossary term="spy" text="spy" >}} or
{{< glossary term="mock" text="mock" >}}.

## Example

Enrolling a user sends a welcome message, but a blank username is rejected before
anything is sent. The rejection path never touches the notifier, so the test
fills that parameter with a dummy and asserts only on the outcome.

{{< tabs >}}
{{< tab icon="cplusplus" label="C++" >}}

```cpp
// enrollment.hpp -- the abstraction and the unit under test.

// Notifier sends a message to a recipient.
class Notifier {
public:
  virtual ~Notifier() = default;
  virtual auto send(std::string_view to, std::string_view message) -> void = 0;
};

// Enrollment signs up users and welcomes the valid ones.
class Enrollment {
public:
  explicit Enrollment(Notifier& notifier) : m_notifier(&notifier) {}

  // Returns true when the user was enrolled, false when rejected.
  auto enroll(std::string_view username) -> bool;

private:
  Notifier* m_notifier;
};
```

```cpp
// A dummy: satisfies the Notifier parameter but is never called.
class DummyNotifier final : public Notifier {
public:
  auto send(std::string_view, std::string_view) -> void override {}
};

TEST_CASE("enroll rejects a blank username before notifying") {
  // Arrange
  auto notifier = DummyNotifier{};
  auto enrollment = Enrollment{notifier};

  // Act
  const bool enrolled = enrollment.enroll("");

  // Assert
  REQUIRE_FALSE(enrolled);
}
```

{{< /tab >}}
{{< tab icon="go" label="Go" >}}

```go
package enrollment

// Notifier sends a message to a recipient.
type Notifier interface {
  Send(to, message string) error
}

// Enrollment signs up users and welcomes the valid ones.
type Enrollment struct {
  notifier Notifier
}

func New(notifier Notifier) *Enrollment {
  return &Enrollment{notifier: notifier}
}

// Enroll signs up the user, reporting whether the name was accepted.
func (e *Enrollment) Enroll(username string) (bool, error) {
  if username == "" {
    return false, nil
  }
  if err := e.notifier.Send(username, "Welcome"); err != nil {
    return false, err
  }
  return true, nil
}
```

```go
package enrollment_test // external package: only the exported API is visible

// dummyNotifier satisfies the Notifier parameter but is never called.
type dummyNotifier struct{}

func (dummyNotifier) Send(to, message string) error { return nil }

func TestEnrollment_Enroll_BlankUsername_DoesNotEnroll(t *testing.T) {
  t.Parallel()

  // Arrange
  service := enrollment.New(dummyNotifier{})

  // Act
  enrolled, err := service.Enroll("")

  // Assert
  if got, want := err, (error)(nil); !cmp.Equal(got, want, cmpopts.EquateErrors()) {
    t.Fatalf("Enrollment.Enroll() err = %v, want %v", got, want)
  }
  if got, want := enrolled, false; got != want {
    t.Errorf("Enrollment.Enroll(%q) = %v, want %v", "", got, want)
  }
}
```

{{< /tab >}}
{{< tab icon="python" label="Python" >}}

```python
# enrollment.py -- the abstraction and the unit under test.
from typing import Protocol


# Notifier sends a message to a recipient.
class Notifier(Protocol):
  def send(self, to: str, message: str) -> None: ...


# Enrollment signs up users and welcomes the valid ones.
class Enrollment:
  def __init__(self, notifier: Notifier) -> None:
    self._notifier = notifier

  def enroll(self, username: str) -> bool:
    if not username:
      return False
    self._notifier.send(username, "Welcome")
    return True
```

```python
from enrollment import Enrollment


# A dummy: satisfies the Notifier parameter but is never called.
class DummyNotifier:
  def send(self, to: str, message: str) -> None: ...


def test_enroll_rejects_a_blank_username_before_notifying():
  # Arrange
  enrollment = Enrollment(DummyNotifier())

  # Act
  enrolled = enrollment.enroll("")

  # Assert
  assert enrolled is False
```

{{< /tab >}}
{{< tab icon="rust" label="Rust" >}}

```rust
// src/lib.rs -- the abstraction and the unit under test.

/// Notifier sends a message to a recipient.
pub trait Notifier {
  fn send(&self, to: &str, message: &str);
}

/// Enrollment signs up users and welcomes the valid ones.
pub struct Enrollment<'a> {
  notifier: &'a dyn Notifier,
}

impl<'a> Enrollment<'a> {
  pub fn new(notifier: &'a dyn Notifier) -> Self {
    Self { notifier }
  }

  /// Returns true when the user was enrolled, false when rejected.
  pub fn enroll(&self, username: &str) -> bool {
    if username.is_empty() {
      return false;
    }
    self.notifier.send(username, "Welcome");
    true
  }
}
```

```rust
// tests/enrollment.rs -- a separate crate that sees only the public API.
use enrollment::{Enrollment, Notifier};

// A dummy: satisfies the Notifier parameter but is never called.
struct DummyNotifier;

impl Notifier for DummyNotifier {
  fn send(&self, _to: &str, _message: &str) {}
}

#[test]
fn enroll_rejects_a_blank_username_before_notifying() {
  // Arrange
  let notifier = DummyNotifier;
  let enrollment = Enrollment::new(&notifier);

  // Act
  let enrolled = enrollment.enroll("");

  // Assert
  assert!(!enrolled);
}
```

{{< /tab >}}
{{< /tabs >}}

The dummy never runs, so the test shows the rejection is decided without the
notifier at all. Make the dummy throw on any call and that becomes an enforced
guarantee rather than a quiet assumption -- but a dummy is still never configured
and never appears in an assertion.

## References

* {{< link
    url="https://martinfowler.com/articles/mocksArentStubs.html"
    text="Mocks Aren't Stubs"
    icon="link"
    hover="Martin Fowler" >}} -- places the dummy at the simple end of the double
  taxonomy, alongside stubs, spies, fakes, and mocks.
* {{< link
    url="http://xunitpatterns.com/Dummy%20Object.html"
    text="Dummy Object"
    icon="link"
    hover="xUnit Patterns" >}} -- Gerard Meszaros' catalog entry for the pattern.
* {{< link
    url="https://en.wikipedia.org/wiki/Test_double"
    text="Test double"
    icon="link"
    hover="Wikipedia" >}} -- overview of the double family the dummy belongs to.
