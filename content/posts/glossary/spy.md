+++
title = 'Spy'
date = 2026-07-23
slug = 'spy'
resources = ['glossary']
tags = ['spy', 'test-double', 'testing']
toc = false
+++

A **spy** is a {{< glossary term="test-double" text="test double" >}} that
records how it was called so the test can inspect those calls afterwards. It is a
{{< glossary term="stub" text="stub" >}} with a memory: it can still return canned
answers, but it also captures the arguments, order, and count of the calls it
received, and leaves the judgment to the test.

<!--more-->

## How a spy works

A spy is {{< glossary term="dependency-injection" text="injected" >}} in place of
a real collaborator and quietly captures each call. After the action, the test
reads what the spy recorded and asserts on it. This is _interaction
verification_ -- the test checks the conversation between objects rather than a
returned value -- which makes a spy the natural tool when the behavior under test
is an outgoing effect that leaves no state behind.

A spy and a {{< glossary term="mock" text="mock" >}} both verify interactions;
the difference is where the expectations live. A mock is told up front how it
must be called and fails the test itself when reality disagrees. A spy stays
passive -- it records everything and judges nothing -- and the test decides
afterwards what to assert. That keeps the Arrange step free of expectations and
puts the assertions where a reader expects to find them, at the cost of a little
more test code.

## The tradeoff

Reach for a spy when a unit's job is to call a collaborator and there is no
observable state to check instead -- publishing an event, enqueuing a job,
emitting a metric. Recording the calls and asserting after the fact keeps the
test readable and the double simple.

The coupling that makes a spy useful also makes it a liability when overused. An
assertion on exact arguments, call count, or order pins the unit's current
implementation in place, so a behavior-preserving refactor can still turn the
test red. Spy only the calls that are genuinely part of the contract, and prefer
asserting on observable state with a {{< glossary term="fake" text="fake" >}}
whenever the outcome is visible there instead.

## Example

Closing an account must leave an audit trail, but that trail is written to a
collaborator rather than returned. A spy captures the entries the service records
so the test can assert exactly what was written.

{{< tabs >}}
{{< tab icon="cplusplus" label="C++" >}}

```cpp
// accounts.hpp -- the abstraction and the unit under test.

struct AuditEntry {
  std::string action;
  std::string subject;
};

class Auditor {
public:
  virtual ~Auditor() = default;
  virtual auto record(const AuditEntry& entry) -> void = 0;
};

class Accounts {
public:
  explicit Accounts(Auditor& auditor) : m_auditor(&auditor) {}

  auto close(const std::string& id) -> void;

private:
  Auditor* m_auditor;
};
```

```cpp
// A spy: records each entry so the test can inspect them afterwards.
class SpyAuditor final : public Auditor {
public:
  auto record(const AuditEntry& entry) -> void override { m_entries.push_back(entry); }

  auto entries() const -> const std::vector<AuditEntry>& { return m_entries; }

private:
  std::vector<AuditEntry> m_entries;
};

TEST_CASE("close records an audit entry") {
  // Arrange
  SpyAuditor auditor;
  Accounts accounts{auditor};

  // Act
  accounts.close("acc-1");

  // Assert
  REQUIRE(auditor.entries().size() == 1);
  REQUIRE(auditor.entries().front().action == "account.closed");
  REQUIRE(auditor.entries().front().subject == "acc-1");
}
```

{{< /tab >}}
{{< tab icon="go" label="Go" >}}

```go
package accounts

type AuditEntry struct {
  Action  string
  Subject string
}

// Auditor records notable actions for later review.
type Auditor interface {
  Record(entry AuditEntry)
}

// Accounts manages the lifecycle of customer accounts.
type Accounts struct {
  auditor Auditor
}

func New(auditor Auditor) *Accounts {
  return &Accounts{auditor: auditor}
}

// Close closes the account and records the action.
func (a *Accounts) Close(id string) {
  // ... close the account ...
  a.auditor.Record(AuditEntry{Action: "account.closed", Subject: id})
}
```

```go
package accounts_test // external package: only the exported API is visible

// spyAuditor records the entries it receives for later inspection.
type spyAuditor struct {
  entries []accounts.AuditEntry
}

func (s *spyAuditor) Record(entry accounts.AuditEntry) {
  s.entries = append(s.entries, entry)
}

func TestCloseRecordsAuditEntry(t *testing.T) {
  t.Parallel()

  // Arrange
  auditor := &spyAuditor{}
  service := accounts.New(auditor)

  // Act
  service.Close("acc-1")

  // Assert
  want := []accounts.AuditEntry{{Action: "account.closed", Subject: "acc-1"}}
  if got := auditor.entries; !cmp.Equal(got, want) {
    t.Errorf("recorded %+v, want %+v", got, want)
  }
}
```

{{< /tab >}}
{{< tab icon="python" label="Python" >}}

```python
# accounts.py -- the abstraction and the unit under test.
from dataclasses import dataclass
from typing import Protocol


@dataclass(frozen=True)
class AuditEntry:
  action: str
  subject: str


class Auditor(Protocol):
  def record(self, entry: AuditEntry) -> None: ...


class Accounts:
  def __init__(self, auditor: Auditor) -> None:
    self._auditor = auditor

  def close(self, account_id: str) -> None:
    # ... close the account ...
    self._auditor.record(AuditEntry("account.closed", account_id))
```

```python
from accounts import Accounts, AuditEntry


# A spy: records the entries it receives for later inspection.
class SpyAuditor:
  def __init__(self) -> None:
    self.entries: list[AuditEntry] = []

  def record(self, entry: AuditEntry) -> None:
    self.entries.append(entry)


def test_close_records_an_audit_entry():
  # Arrange
  auditor = SpyAuditor()
  accounts = Accounts(auditor)

  # Act
  accounts.close("acc-1")

  # Assert
  assert auditor.entries == [AuditEntry("account.closed", "acc-1")]
```

{{< /tab >}}
{{< tab icon="rust" label="Rust" >}}

```rust
// src/lib.rs -- the abstraction and the unit under test.
#[derive(Debug, PartialEq)]
pub struct AuditEntry {
  pub action: String,
  pub subject: String,
}

pub trait Auditor {
  fn record(&self, entry: AuditEntry);
}

pub struct Accounts<'a> {
  auditor: &'a dyn Auditor,
}

impl<'a> Accounts<'a> {
  pub fn new(auditor: &'a dyn Auditor) -> Self {
    Self { auditor }
  }

  pub fn close(&self, id: &str) {
    // ... close the account ...
    self.auditor.record(AuditEntry {
      action: "account.closed".to_string(),
      subject: id.to_string(),
    });
  }
}
```

```rust
// tests/accounts.rs -- a separate crate that sees only the public API.
use std::cell::RefCell;

use accounts::{Accounts, AuditEntry, Auditor};

// A spy: records the entries it receives for later inspection.
#[derive(Default)]
struct SpyAuditor {
  entries: RefCell<Vec<AuditEntry>>,
}

impl Auditor for SpyAuditor {
  fn record(&self, entry: AuditEntry) {
    self.entries.borrow_mut().push(entry);
  }
}

#[test]
fn close_records_an_audit_entry() {
  // Arrange
  let auditor = SpyAuditor::default();
  let accounts = Accounts::new(&auditor);

  // Act
  accounts.close("acc-1");

  // Assert
  assert_eq!(
    *auditor.entries.borrow(),
    [AuditEntry { action: "account.closed".to_string(), subject: "acc-1".to_string() }],
  );
}
```

{{< /tab >}}
{{< /tabs >}}

{{< note >}}
The spy asserts on _what_ was recorded, not on how the service arrived there. Pin
down the entry that is part of the audit contract and nothing more -- an
assertion on incidental extra calls would fail the next harmless refactor.
{{< /note >}}

## References

* {{< link
    url="https://martinfowler.com/articles/mocksArentStubs.html"
    text="Mocks Aren't Stubs"
    icon="link"
    hover="Martin Fowler" >}} -- distinguishes spies and mocks from stubs, and
  interaction verification from state verification.
* {{< link
    url="http://xunitpatterns.com/Test%20Spy.html"
    text="Test Spy"
    icon="link"
    hover="xUnit Patterns" >}} -- Gerard Meszaros' catalog entry for the pattern.
* {{< link
    url="https://en.wikipedia.org/wiki/Test_double"
    text="Test double"
    icon="link"
    hover="Wikipedia" >}} -- overview of the double family the spy belongs to.
