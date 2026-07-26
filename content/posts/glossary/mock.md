+++
title = 'Mock'
date = 2026-06-26
slug = 'mock'
aliases = ['/glossary/mock']
resources = ['glossary']
tags = ['mock', 'testing']
toc = false
+++

A **mock** is a {{< glossary term="test-double" text="test double" >}} that is
preconfigured with expectations about how it should be called, and which verifies
those expectations during a test. Where a {{< glossary term="stub" text="stub" >}}
simply returns canned values, a mock asserts on the _interactions_ -- which
methods were invoked, with what arguments, and how often.

<!--more-->

## How a mock works

A mock is supplied to the unit under test in place of a real
{{< glossary term="collaborator" text="collaborator" >}}, usually via
{{< glossary term="dependency-injection" text="dependency injection" >}}.
It is configured up front with the calls it expects, checks each incoming call
against those expectations, and reports a failure itself when they are not met.
This is _interaction verification_: the test passes or fails based on the
conversation between objects, not on any resulting state.

Most languages have a library that generates mocks so you do not hand-write the
expectation logic, but the shape is always the same -- set the expectations,
exercise the unit, then let the mock verify itself. A double that instead
records calls passively and leaves the assertions to the test is a
{{< glossary term="spy" text="spy" >}}; a mock goes further by baking the
expectations in and failing when they are not satisfied.

## The tradeoff

Because a mock asserts on _how_ a unit calls its collaborators, it couples the
test to the implementation. A test that demands `Save` be called exactly once
breaks the moment you batch two writes, rename the method, or reorder the steps
-- even when the externally observable behavior is identical. Over-mocked suites
tend to be brittle and to restate the implementation back to itself, which is
precisely what a test should _not_ do.

For that reason, prefer a more realistic double and assert on the outcome
wherever the outcome is observable. Use a
{{< glossary term="fake" text="fake" >}} plus state verification for
collaborators like repositories, and reserve mocks for the narrow case where the
_call itself_ is the behavior under test -- there is no state to inspect, so the
interaction is the observable result.

## Example

Sending a notification is a good fit for a mock: when an order is placed, the
service is _supposed_ to notify the customer, and that outgoing call is the whole
point. There is no local state to assert on, so the mock is given the call it
expects and verifies that interaction itself.

{{< tabs >}}
{{< tab icon="cplusplus" label="C++" >}}

```cpp
// orders.hpp -- the abstraction and the unit under test.

// Notifier sends a message to a recipient.
class Notifier {
public:
  virtual ~Notifier() = default;
  virtual auto send(const std::string& to, const std::string& message) -> void = 0;
};

// OrderService places orders and confirms them to the customer.
class OrderService {
public:
  explicit OrderService(Notifier& notifier) : m_notifier(&notifier) {}

  auto place(const std::string& customer) -> void;

private:
  Notifier* m_notifier;
};
```

```cpp
// A mock: expects to notify one recipient, and verifies it itself.
class MockNotifier final : public Notifier {
public:
  explicit MockNotifier(std::string expected) : m_expected(std::move(expected)) {}

  auto send(const std::string& to, const std::string& /*message*/) -> void override {
    REQUIRE(to == m_expected);
    ++m_calls;
  }

  auto verify() const -> void { REQUIRE(m_calls == 1); }

private:
  std::string m_expected;
  int m_calls = 0;
};

TEST_CASE("place notifies the customer") {
  // Arrange
  MockNotifier notifier{"ada@example.com"};
  OrderService service{notifier};

  // Act
  service.place("ada@example.com");

  // Assert
  notifier.verify();
}
```

{{< /tab >}}
{{< tab icon="go" label="Go" >}}

```go
package orders

// Notifier sends a message to a recipient.
type Notifier interface {
  Send(to, message string) error
}

// Service places orders and confirms them to the customer.
type Service struct {
  notifier Notifier
}

func NewService(notifier Notifier) *Service {
  return &Service{notifier: notifier}
}

// Place confirms the order to the customer.
func (s *Service) Place(customer string) error {
  // ... persist the order ...
  return s.notifier.Send(customer, "Your order is confirmed")
}
```

```go
package orders_test

// mockNotifier expects to notify one recipient, and verifies it itself.
type mockNotifier struct {
  t        *testing.T
  expected string
  calls    int
}

func (m *mockNotifier) Send(to, message string) error {
  m.t.Helper()
  if to != m.expected {
    m.t.Errorf("Send() notified %q, want %q", to, m.expected)
  }
  m.calls++
  return nil
}

func (m *mockNotifier) verify() {
  m.t.Helper()
  if m.calls != 1 {
    m.t.Errorf("Send() called %d times, want 1", m.calls)
  }
}

func TestPlaceNotifiesCustomer(t *testing.T) {
  t.Parallel()

  // Arrange
  notifier := &mockNotifier{t: t, expected: "ada@example.com"}
  service := orders.NewService(notifier)

  // Act
  if err := service.Place("ada@example.com"); err != nil {
    t.Fatalf("Place() returned an unexpected error: %v", err)
  }

  // Assert
  notifier.verify()
}
```

{{< /tab >}}
{{< tab icon="python" label="Python" >}}

```python
# orders.py -- the abstraction and the unit under test.
from typing import Protocol

# Notifier sends a message to a recipient.
class Notifier(Protocol):
  def send(self, to: str, message: str) -> None: ...

# Service places orders and confirms them to the customer.
class Service:
  def __init__(self, notifier: Notifier) -> None:
    self._notifier = notifier

  def place(self, customer: str) -> None:
    # ... persist the order ...
    self._notifier.send(customer, "Your order is confirmed")
```

```python
from orders import Service

# A mock: expects to notify one recipient, and verifies it itself.
class MockNotifier:
  def __init__(self, expected: str) -> None:
    self._expected = expected
    self._calls = 0

  def send(self, to: str, message: str) -> None:
    assert to == self._expected, f"notified {to!r}, want {self._expected!r}"
    self._calls += 1

  def verify(self) -> None:
    assert self._calls == 1, f"send called {self._calls} times, want 1"

def test_place_notifies_the_customer():
  # Arrange
  notifier = MockNotifier("ada@example.com")
  service = Service(notifier)

  # Act
  service.place("ada@example.com")

  # Assert
  notifier.verify()
```

{{< /tab >}}
{{< tab icon="rust" label="Rust" >}}

```rust
// src/lib.rs -- the abstraction and the unit under test.

/// Notifier sends a message to a recipient.
pub trait Notifier {
  fn send(&self, to: &str, message: &str);
}

/// OrderService places orders and confirms them to the customer.
pub struct OrderService<'a> {
  notifier: &'a dyn Notifier,
}

impl<'a> OrderService<'a> {
  pub fn new(notifier: &'a dyn Notifier) -> Self {
    Self { notifier }
  }

  pub fn place(&self, customer: &str) {
    // ... persist the order ...
    self.notifier.send(customer, "Your order is confirmed");
  }
}
```

```rust
// tests/orders.rs -- a separate crate that sees only the public API.
use std::cell::Cell;

use orders::{Notifier, OrderService};

// A mock: expects to notify one recipient, and verifies it itself.
struct MockNotifier {
  expected: String,
  calls: Cell<u32>,
}

impl MockNotifier {
  fn new(expected: &str) -> Self {
    Self { expected: expected.to_string(), calls: Cell::new(0) }
  }

  fn verify(&self) {
    assert_eq!(self.calls.get(), 1);
  }
}

impl Notifier for MockNotifier {
  fn send(&self, to: &str, _message: &str) {
    assert_eq!(to, self.expected);
    self.calls.set(self.calls.get() + 1);
  }
}

#[test]
fn place_notifies_the_customer() {
  // Arrange
  let notifier = MockNotifier::new("ada@example.com");
  let service = OrderService::new(&notifier);

  // Act
  service.place("ada@example.com");

  // Assert
  notifier.verify();
}
```

{{< /tab >}}
{{< /tabs >}}

{{< note >}}
Notice the mock only asserts _that_ the customer was notified, not what the rest
of the system now looks like. If the test also cared about, say, the order being
persisted, that part should be checked with a fake repository and a state
assertion -- not another mock.
{{< /note >}}

## References

* {{< link
    url="https://martinfowler.com/articles/mocksArentStubs.html"
    text="Mocks Aren't Stubs"
    icon="link"
    hover="Martin Fowler" >}} -- the reference on when interaction verification
  earns its place, and when it does not.
* {{< link
    url="http://xunitpatterns.com/Mock%20Object.html"
    text="Mock Object"
    icon="link"
    hover="xUnit Patterns" >}} -- Gerard Meszaros' catalog entry for the pattern.
* {{< link
    url="https://en.wikipedia.org/wiki/Mock_object"
    text="Mock object"
    icon="link"
    hover="Wikipedia" >}} -- short overview of mocks and interaction testing.
