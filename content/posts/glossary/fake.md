+++
title = 'Fake'
date = 2026-07-23
slug = 'fake'
resources = ['glossary']
tags = ['fake', 'test-double', 'testing']
toc = false
+++

A **fake** is a {{< glossary term="test-double" text="test double" >}} with a
real, working implementation -- just one that takes a shortcut unsuitable for
production, such as an incrementing counter standing in for a database sequence.
It carries genuine, evolving state, so a unit gets realistic behavior from it and
the test asserts on the outcome that behavior produces.

<!--more-->

## How a fake works

A fake honors the same contract as the real collaborator, so the unit under test
interacts with it exactly as it would in production. That lets the test verify
behavior through the unit's own public interface: the assertion never reaches
into the fake, it only checks what the unit observably does. This is the most
refactor-resilient form of verification, because it depends on the unit's
contract rather than on any particular sequence of calls.

## The tradeoff

A fake is the right default once a {{< glossary term="stub" text="stub" >}}'s
canned answers stop being convincing -- when the collaborator carries state, when
one call's result depends on an earlier one, or when several tests need the same
realistic behavior. Written once and reused, a fake keeps individual tests short
and keeps their assertions about outcomes rather than calls.

The cost is that a fake is real code: it can harbor bugs, and if its behavior
diverges from the real implementation the suite goes green against a lie. Keep a
fake small, and hold it to the same contract as production -- ideally by running
one shared set of contract tests against both -- so the two cannot drift. When a
collaborator only needs to answer a single canned question, a fake is overkill
and a stub is simpler; when the behavior under test is the outgoing call itself,
a {{< glossary term="spy" text="spy" >}} fits better.

## Example

A tracker stamps each ticket it opens with a reference drawn from an `IdSource`.
The contract is that every ticket gets a _distinct_ reference -- so the double
has to hand out fresh values, not a canned one. A stub returning a fixed id would
make two tickets collide; a working fake counter gives the real behavior, and the
test asserts the outcome through the tracker's public interface.

{{< tabs >}}
{{< tab icon="cplusplus" label="C++" >}}

```cpp
// tracker.hpp -- the abstraction and the unit under test.

class IdSource {
public:
  virtual ~IdSource() = default;
  virtual auto next() -> std::string = 0;
};

struct Ticket {
  std::string reference;
  std::string title;
};

class Tracker {
public:
  explicit Tracker(IdSource& ids) : m_ids(&ids) {}

  auto open(const std::string& title) -> Ticket;

private:
  IdSource* m_ids;
};
```

```cpp
// A fake: a working IdSource that hands out sequential identifiers.
class FakeIdSource final : public IdSource {
public:
  auto next() -> std::string override { return "id-" + std::to_string(++m_count); }

private:
  int m_count = 0;
};

TEST_CASE("open stamps each ticket with a distinct reference") {
  // Arrange
  FakeIdSource ids;
  Tracker tracker{ids};

  // Act
  const Ticket first = tracker.open("disk full");
  const Ticket second = tracker.open("disk full");

  // Assert
  REQUIRE(first.reference != second.reference);
}
```

{{< /tab >}}
{{< tab icon="go" label="Go" >}}

```go
package tracker

// IDSource hands out identifiers for new tickets.
type IDSource interface {
  Next() string
}

// Ticket is an opened unit of work.
type Ticket struct {
  Reference string
  Title     string
}

// Tracker opens tickets, stamping each with a reference from the source.
type Tracker struct {
  ids IDSource
}

func New(ids IDSource) *Tracker {
  return &Tracker{ids: ids}
}

// Open opens a ticket with the given title.
func (t *Tracker) Open(title string) Ticket {
  return Ticket{Reference: t.ids.Next(), Title: title}
}
```

```go
package tracker_test // external package: only the exported API is visible

// fakeIdSource is a working IdSource that hands out sequential identifiers.
type fakeIdSource struct {
  count int
}

func (s *fakeIdSource) Next() string {
  s.count++
  return fmt.Sprintf("id-%d", s.count)
}

func TestOpenStampsDistinctReferences(t *testing.T) {
  t.Parallel()

  // Arrange
  tr := tracker.New(&fakeIdSource{})

  // Act
  first := tr.Open("disk full")
  second := tr.Open("disk full")

  // Assert
  if got, want := (first.Reference == second.Reference), false; got != want {
    t.Errorf("tracker.Open(...) repeat id = %v, want = %v", got, want)
  }
}
```

{{< /tab >}}
{{< tab icon="python" label="Python" >}}

```python
# tracker.py -- the abstraction and the unit under test.
from dataclasses import dataclass
from typing import Protocol

class IdSource(Protocol):
  def next(self) -> str: ...

@dataclass(frozen=True)
class Ticket:
  reference: str
  title: str


class Tracker:
  def __init__(self, ids: IdSource) -> None:
    self._ids = ids

  def open(self, title: str) -> Ticket:
    return Ticket(reference=self._ids.next(), title=title)
```

```python
from tracker import Tracker


# A fake: a working IdSource that hands out sequential identifiers.
class FakeIdSource:
  def __init__(self) -> None:
    self._count = 0

  def next(self) -> str:
    self._count += 1
    return f"id-{self._count}"

def test_open_stamps_distinct_references():
  # Arrange
  tracker = Tracker(FakeIdSource())

  # Act
  first = tracker.open("disk full")
  second = tracker.open("disk full")

  # Assert
  assert first.reference != second.reference
```

{{< /tab >}}
{{< tab icon="rust" label="Rust" >}}

```rust
// src/lib.rs -- the abstraction and the unit under test.

pub trait IdSource {
  fn next(&self) -> String;
}

pub struct Ticket {
  pub reference: String,
  pub title: String,
}

pub struct Tracker<'a> {
  ids: &'a dyn IdSource,
}

impl<'a> Tracker<'a> {
  pub fn new(ids: &'a dyn IdSource) -> Self {
    Self { ids }
  }

  pub fn open(&self, title: &str) -> Ticket {
    Ticket { reference: self.ids.next(), title: title.to_string() }
  }
}
```

```rust
// tests/tracker.rs -- a separate crate that sees only the public API.
use std::cell::Cell;

use tracker::{IdSource, Tracker};

// A fake: a working IdSource that hands out sequential identifiers.
#[derive(Default)]
struct FakeIdSource {
  count: Cell<u32>,
}

impl IdSource for FakeIdSource {
  fn next(&self) -> String {
    self.count.set(self.count.get() + 1);
    format!("id-{}", self.count.get())
  }
}

#[test]
fn open_stamps_distinct_references() {
  // Arrange
  let ids = FakeIdSource::default();
  let tracker = Tracker::new(&ids);

  // Act
  let first = tracker.open("disk full");
  let second = tracker.open("disk full");

  // Assert
  assert_ne!(first.reference, second.reference);
}
```

{{< /tab >}}
{{< /tabs >}}

{{< note >}}
The two references differ only because the fake carries state and hands out a new
value each call. That is the line between a fake and a
{{< glossary term="stub" text="stub" >}}: a stub answers, a fake behaves.
{{< /note >}}

## References

* {{< link
    url="https://martinfowler.com/articles/mocksArentStubs.html"
    text="Mocks Aren't Stubs"
    icon="link"
    hover="Martin Fowler" >}} -- places fakes among the other doubles and
  contrasts state with interaction verification.
* {{< link
    url="http://xunitpatterns.com/Fake%20Object.html"
    text="Fake Object"
    icon="link"
    hover="xUnit Patterns" >}} -- Gerard Meszaros' catalog entry for the pattern.
* {{< link
    url="https://en.wikipedia.org/wiki/Test_double"
    text="Test double"
    icon="link"
    hover="Wikipedia" >}} -- overview of the double family the fake belongs to.
