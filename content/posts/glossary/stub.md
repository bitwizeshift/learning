+++
title = 'Stub'
date = 2026-07-23
slug = 'stub'
resources = ['glossary']
tags = ['stub', 'test-double', 'testing']
toc = false
+++

A **stub** is a {{< glossary term="test-double" text="test double" >}} that
returns canned, pre-arranged answers to the calls a unit makes during a test. It
supplies the inputs a unit needs from a collaborator -- a configured user, a
fixed exchange rate, an allow-or-deny decision -- without the cost or
nondeterminism of the real thing. Unlike a {{< glossary term="mock" text="mock" >}},
a stub makes no assertions of its own; it only feeds the unit so the test can
check what the unit does with the answer.

<!--more-->

## How a stub works

A stub is {{< glossary term="dependency-injection" text="injected" >}} in place
of a real collaborator and hands back values the test chose in advance. The unit
under test cannot tell the difference, so it runs its real logic against those
values and the test asserts on the unit's observable result -- never on the stub
itself. Because a stub answers a query rather than receiving a command, this
stays a form of state verification: given this input from the collaborator, does
the unit produce the right output?

## The tradeoff

A stub is the cheapest and least brittle double to reach for when a unit's
behavior branches on what a collaborator returns. It keeps a test deterministic,
keeps the assertion on output, and does not couple the test to how the unit calls
the collaborator.

The limitation is the flip side: a stub can feed a unit but never confirm that a
call happened. When the behavior under test is an outgoing effect -- an email
sent, an event published -- a stub cannot see it, and a
{{< glossary term="spy" text="spy" >}} or {{< glossary term="mock" text="mock" >}}
fits instead. A stub is also only as accurate as the answer you hard-code; if it
drifts from what the real collaborator would return, the test passes against a
fiction. Once a collaborator's behavior is rich enough that canned answers stop
being convincing -- state that must be written and read back, one answer that
depends on an earlier call -- prefer a {{< glossary term="fake" text="fake" >}}.

## Example

A request handler grants or denies access based on what an `Authorizer` reports.
Stubbing the authorizer to a fixed decision lets the test drive both branches and
assert on the status the handler returns.

{{< tabs >}}
{{< tab icon="cplusplus" label="C++" >}}

```cpp
// access.hpp -- the abstraction and the unit under test.

class Authorizer {
public:
  virtual ~Authorizer() = default;
  virtual auto is_allowed(const std::string& token) const -> bool = 0;
};

auto handle(const Authorizer& auth, const std::string& token) -> int;
```

```cpp
// A stub: answers is_allowed with a fixed, preset result.
class StubAuthorizer final : public Authorizer {
public:
  explicit StubAuthorizer(bool allowed) : m_allowed(allowed) {}

  auto is_allowed(const std::string&) const -> bool override { return m_allowed; }

private:
  bool m_allowed;
};

TEST_CASE("handle serves a permitted request") {
  // Arrange
  const StubAuthorizer auth{true};

  // Act
  const int status = handle(auth, "token");

  // Assert
  REQUIRE(status == 200);
}

TEST_CASE("handle forbids a denied request") {
  // Arrange
  const StubAuthorizer auth{false};

  // Act
  const int status = handle(auth, "token");

  // Assert
  REQUIRE(status == 403);
}
```

{{< /tab >}}
{{< tab icon="go" label="Go" >}}

```go
package access

// Authorizer reports whether a token may perform the request.
type Authorizer interface {
  IsAllowed(token string) bool
}

// Handle returns the HTTP status for the request carrying the given token.
func Handle(auth Authorizer, token string) int {
  if !auth.IsAllowed(token) {
    return http.StatusForbidden
  }
  return http.StatusOK
}
```

```go
package access_test // external package: only the exported API is visible

// stubAuthorizer answers IsAllowed with a fixed, preset result.
type stubAuthorizer bool

func (s stubAuthorizer) IsAllowed(string) bool { return bool(s) }

func TestHandle(t *testing.T) {
  t.Parallel()

  testCases := []struct {
    name    string
    allowed bool
    want    int
  }{
    {name: "permitted request is served", allowed: true, want: http.StatusOK},
    {name: "denied request is forbidden", allowed: false, want: http.StatusForbidden},
  }

  for _, tc := range testCases {
    t.Run(tc.name, func(t *testing.T) {
      // Arrange
      auth := stubAuthorizer(tc.allowed)

      // Act
      got := access.Handle(auth, "token")

      // Assert
      if !cmp.Equal(got, tc.want) {
        t.Errorf("Handle() = %d, want %d", got, tc.want)
      }
    })
  }
}
```

{{< /tab >}}
{{< tab icon="python" label="Python" >}}

```python
# access.py -- the abstraction and the unit under test.
from http import HTTPStatus
from typing import Protocol


class Authorizer(Protocol):
  def is_allowed(self, token: str) -> bool: ...


def handle(auth: Authorizer, token: str) -> HTTPStatus:
  if not auth.is_allowed(token):
    return HTTPStatus.FORBIDDEN
  return HTTPStatus.OK
```

```python
from http import HTTPStatus

from access import handle


# A stub: answers is_allowed with a fixed, preset result.
class StubAuthorizer:
  def __init__(self, allowed: bool) -> None:
    self._allowed = allowed

  def is_allowed(self, token: str) -> bool:
    return self._allowed


def test_handle_serves_a_permitted_request():
  # Arrange
  auth = StubAuthorizer(allowed=True)

  # Act
  status = handle(auth, "token")

  # Assert
  assert status == HTTPStatus.OK


def test_handle_forbids_a_denied_request():
  # Arrange
  auth = StubAuthorizer(allowed=False)

  # Act
  status = handle(auth, "token")

  # Assert
  assert status == HTTPStatus.FORBIDDEN
```

{{< /tab >}}
{{< tab icon="rust" label="Rust" >}}

```rust
// src/lib.rs -- the abstraction and the unit under test.

pub trait Authorizer {
  fn is_allowed(&self, token: &str) -> bool;
}

pub fn handle(auth: &impl Authorizer, token: &str) -> u16 {
  if auth.is_allowed(token) {
    200
  } else {
    403
  }
}
```

```rust
// tests/access.rs -- a separate crate that sees only the public API.
use access::{handle, Authorizer};

// A stub: answers is_allowed with a fixed, preset result.
struct StubAuthorizer(bool);

impl Authorizer for StubAuthorizer {
  fn is_allowed(&self, _token: &str) -> bool {
    self.0
  }
}

#[test]
fn handle_serves_a_permitted_request() {
  // Arrange
  let auth = StubAuthorizer(true);

  // Act
  let status = handle(&auth, "token");

  // Assert
  assert_eq!(status, 200);
}

#[test]
fn handle_forbids_a_denied_request() {
  // Arrange
  let auth = StubAuthorizer(false);

  // Act
  let status = handle(&auth, "token");

  // Assert
  assert_eq!(status, 403);
}
```

{{< /tab >}}
{{< /tabs >}}

{{< note >}}
A stub that answers a query is fine. A stub standing in for a command that you
then assert was issued has quietly become a {{< glossary term="spy" text="spy" >}}
or {{< glossary term="mock" text="mock" >}} -- keep the roles distinct.
{{< /note >}}

## References

* {{< link
    url="https://martinfowler.com/articles/mocksArentStubs.html"
    text="Mocks Aren't Stubs"
    icon="link"
    hover="Martin Fowler" >}} -- the reference on stubs versus mocks, and state
  versus interaction verification.
* {{< link
    url="http://xunitpatterns.com/Test%20Stub.html"
    text="Test Stub"
    icon="link"
    hover="xUnit Patterns" >}} -- Gerard Meszaros' catalog entry for the pattern.
* {{< link
    url="https://en.wikipedia.org/wiki/Test_double"
    text="Test double"
    icon="link"
    hover="Wikipedia" >}} -- overview of the double family the stub belongs to.
