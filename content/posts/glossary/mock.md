+++
title = 'Mock'
date = 2026-06-26
slug = 'mock'
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

A mock is supplied to the unit under test in place of a real collaborator,
usually via {{< glossary term="dependency-injection" text="dependency injection" >}}.
It records every call it receives, and the test then checks those calls against
its expectations. This is _interaction verification_: the test passes or fails
based on the conversation between objects, not on any resulting state.

Most languages have a library that generates mocks so you do not hand-write the
recording logic, but the shape is always the same -- capture the calls, then
assert on them. A double that records passively and leaves those assertions to
the test is a {{< glossary term="spy" text="spy" >}}; a mock goes further by
baking the expectations in and failing itself when they are not met.

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
point. There is no local state to assert on, so the test verifies the
interaction directly.

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
// A mock: records the recipients it was asked to notify.
class MockNotifier final : public Notifier {
public:
  auto send(const std::string& to, const std::string& /*message*/) -> void override {
    m_recipients.push_back(to);
  }

  auto recipients() const -> const std::vector<std::string>& { return m_recipients; }

private:
  std::vector<std::string> m_recipients;
};

TEST_CASE("place notifies the customer") {
  // Arrange
  MockNotifier notifier;
  OrderService service{notifier};

  // Act
  service.place("ada@example.com");

  // Assert
  REQUIRE(notifier.recipients() == std::vector<std::string>{"ada@example.com"});
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

// mockNotifier records the recipients it was asked to notify.
type mockNotifier struct {
  recipients []string
}

func (m *mockNotifier) Send(to, message string) error {
  m.recipients = append(m.recipients, to)
  return nil
}

func TestPlaceNotifiesCustomer(t *testing.T) {
  t.Parallel()

  // Arrange
  notifier := &mockNotifier{}
  service := orders.NewService(notifier)

  // Act
  err := service.Place("ada@example.com")

  // Assert
  if err != nil {
    t.Fatalf("Place() returned an unexpected error: %v", err)
  }
  if len(notifier.recipients) != 1 || notifier.recipients[0] != "ada@example.com" {
    t.Errorf("notified %v, want one message to %q", notifier.recipients, "ada@example.com")
  }
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
from unittest.mock import ANY, Mock

from orders import Notifier, Service

def test_place_notifies_the_customer():
  # Arrange
  notifier = Mock(spec=Notifier)
  service = Service(notifier)

  # Act
  service.place("ada@example.com")

  # Assert
  notifier.send.assert_called_once_with("ada@example.com", ANY)
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
use std::cell::RefCell;

use orders::{Notifier, OrderService};

// A mock: records the recipients it was asked to notify.
#[derive(Default)]
struct MockNotifier {
  recipients: RefCell<Vec<String>>,
}

impl Notifier for MockNotifier {
  fn send(&self, to: &str, _message: &str) {
    self.recipients.borrow_mut().push(to.to_string());
  }
}

#[test]
fn place_notifies_the_customer() {
  // Arrange
  let notifier = MockNotifier::default();
  let service = OrderService::new(&notifier);

  // Act
  service.place("ada@example.com");

  // Assert
  assert_eq!(*notifier.recipients.borrow(), ["ada@example.com"]);
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
