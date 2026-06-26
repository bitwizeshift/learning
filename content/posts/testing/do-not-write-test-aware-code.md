+++
title = 'Do not write test-aware code'
date = 2026-06-25
id = 'TEST-006'
slug = 'TEST-006'
resources = ['best-practice']
toolchains = ['cpp', 'go', 'python', 'rust']
tags = ['unit-testing', 'dependency-injection', 'test-aware']
concepts = ['testing', 'design']
description = 'Never let production code detect that it is under test and change its behavior.'
toc = true
[focus]
concepts = ['testing']
+++

Do not write code that detects whether it is running under test and changes its
behavior -- **test-aware code**. A unit that branches on a test flag, the
presence of a test runner, the presence of an environment variable, or a
test-only build runs a different path under test than it does in production, so
the test verifies behavior you do not ship.
Instead, make code testable through design -- typically **dependency injection** --
so that tests substitute collaborators without the unit ever knowing.

{{< tip >}}
If a test only passes because the code skipped its real work, the test is
exercising the test-mode path, not the production behavior. It has little value
as a test.
{{< /tip >}}

<!--more-->

## Motivation

The purpose of a test is to give confidence that the shipped code works.
Test-aware code defeats that purpose: the branch the test takes is, by
construction, not the branch production takes. The production branch -- the
database write, the network call, the real side effect -- is usually the riskiest
part of the unit, and it is exactly the part a test flag skips. This enables the
test to pass while the untested path is the one that runs in real life, giving
a false sense of security.

## Justification

A test is only meaningful if it exercises the same code that runs in production.
Removing test-awareness restores that property: with the test flag gone, there is
a single code path, and the test drives it the same way a real caller would. The
difference between test and production then lives entirely in the *collaborators*
that are supplied from the outside, **not** in the unit's own logic.

Dependency injection is what makes this possible without test-awareness; by
accepting its collaborators -- a datastore, a clock, a network client -- through
a constructor, parameter, or interface, a unit lets a test pass a substitute and
production pass the real thing, while the unit's behavior stays identical in
both. The seam is at the boundary, where it belongs, instead of inside the
logic. The need for a test flag is a sign that such a seam is missing.

## Examples

Each example is an `Account` whose `deposit` updates an in-memory balance and then
persists it. The naive, test-aware version suppresses the persistence under test;
the injected version supplies the datastore from the outside so the same code
runs everywhere.

The *mechanism* that tempts test-awareness differs by language -- a flag, a
preprocessor guard, runner detection, or a build configuration. Each dropdown
notes the relevant one.

{{< tabs >}}
{{< tab icon="cplusplus" label="C++" >}}

The common test-aware mechanism in C++ is a preprocessor guard (`#ifdef
UNIT_TEST`) or a runtime flag that strips real work from test builds. Both make
the test build compile different code than production. Prefer injecting an
abstract interface that a test can implement.

### ❌ Bad Example

```cpp
class Account {
public:
  // The persistence call is compiled out of test builds entirely.
  auto deposit(int amount) -> void {
    balance_ += amount;
#ifndef UNIT_TEST
    save_to_database(balance_);
#endif
  }

  auto balance() const -> int { return balance_; }

private:
  int balance_ = 0;
};

// Built with -DUNIT_TEST, so deposit() never persists here.
TEST_CASE("deposit increases the balance") {
  auto account = Account{};
  account.deposit(100);
  REQUIRE(account.balance() == 100);
}
```

### ✅ Good Example

```cpp
// An injected interface lets the test substitute the collaborator.
class Store {
public:
  virtual ~Store() = default;
  virtual auto save(int balance) -> void = 0;
};

class Account {
public:
  explicit Account(Store& store) : m_store{&store} {}

  auto deposit(int amount) -> void {
    m_balance += amount;
    m_store->save(m_balance);
  }

  auto balance() const -> int { return m_balance; }

private:
  Store* m_store;
  int m_balance = 0;
};
```

```cpp
struct RecorderStore : Store {
  int saved = 0;
  auto save(int balance) -> void override { saved = balance; }
};

TEST_CASE("deposit increases the balance and persists it") {
  // Arrange
  auto store = RecorderStore{};
  auto account = Account{store};

  // Act
  account.deposit(100);

  // Assert
  REQUIRE(account.balance() == 100);
}
```

{{< /tab >}}

{{< tab icon="go" label="Go" >}}

Go encourages dependency injection through interfaces, and collaborators can be
supplied as struct fields, constructor arguments, or functional options. The
test-aware temptation is a boolean field like `TestMode`; the fix is to inject the
datastore behind an interface so production and tests run the same method body.

### ❌ Bad Example

```go
package bank

type Account struct {
  balance  int
  TestMode bool
}

// Deposit skips persistence when TestMode is set, so the test takes a different
// path than production.
func (a *Account) Deposit(amount int) {
  a.balance += amount
  if a.TestMode {
    return
  }
  saveToDatabase(a.balance)
}

func (a *Account) Balance() int { return a.balance }
```

```go
package bank_test

func TestDeposit(t *testing.T) {
  account := &bank.Account{TestMode: true}

  account.Deposit(100)

  if got, want := account.Balance(), 100; got != want {
    t.Errorf("Balance() = %d, want %d", got, want)
  }
}
```

### ✅ Good Example

```go
package bank

// Store persists account balances.
type Store interface {
  Save(balance int) error
}

type Account struct {
  balance int
  store   Store
}

func NewAccount(store Store) *Account {
  return &Account{store: store}
}

func (a *Account) Deposit(amount int) error {
  a.balance += amount
  return a.store.Save(a.balance)
}

func (a *Account) Balance() int { return a.balance }
```

```go
package bank_test

type recorderStore struct{ saved int }

func (s *recorderStore) Save(balance int) error {
  s.saved = balance
  return nil
}

func TestDeposit(t *testing.T) {
  // Arrange
  store := &recorderStore{}
  account := bank.NewAccount(store)

  // Act
  err := account.Deposit(100)

  // Assert
  if got, want := err, (error)(nil); !errors.Is(got, want) {
    t.Fatalf("Deposit(100) returned error: %v", err)
  }
  if got, want := account.Balance(), 100; got != want {
    t.Errorf("Balance() = %d, want %d", got, want)
  }
}
```

{{< /tab >}}

{{< tab icon="python" label="Python" >}}

The Python temptation is to detect the runner at runtime --
`"pytest" in sys.modules`, an environment variable, or an `is_test` flag. Prefer
passing the collaborator into the constructor so the same `deposit` body runs
under test and in production.

### ❌ Bad Example

```python
# bank.py
import sys

class Account:
  def __init__(self, balance: int = 0):
    self.balance = balance

  def deposit(self, amount: int) -> None:
    self.balance += amount
    if "pytest" in sys.modules:  # behaves differently under test
      return
    self._save_to_database()
```

```python
def test_deposit():
  account = Account()
  account.deposit(100)
  assert account.balance == 100
```

### ✅ Good Example

```python
# bank.py
class Account:
  def __init__(self, store: Store, balance: int = 0):
    self._store = store
    self.balance = balance

  def deposit(self, amount: int) -> None:
    self.balance += amount
    self._store.save(self.balance)
```

```python
class RecorderStore:
  def __init__(self):
    self.saved = None

  def save(self, balance: int) -> None:
    self.saved = balance


def test_deposit_increases_balance_and_persists_it():
  # Arrange
  store = RecorderStore()
  account = Account(store)

  # Act
  account.deposit(100)

  # Assert
  assert account.balance == 100
  assert store.saved == 100
```

{{< /tab >}}

{{< tab icon="rust" label="Rust" >}}

`#[cfg(test)]` is the idiomatic way to compile *test-only modules*, but using
`#[cfg(not(test))]` to branch production logic makes the test build behave
differently than the release build -- the test-aware antipattern. Prefer injecting
the collaborator as a generic type parameter (or trait object) on the struct.

### ❌ Bad Example

```rust
pub struct Account {
  balance: i64,
}

impl Account {
  // The persistence call is compiled out of test builds.
  pub fn deposit(&mut self, amount: i64) {
    self.balance += amount;

    #[cfg(not(test))]
    self.save_to_database();
  }

  pub fn balance(&self) -> i64 {
    self.balance
  }
}
```

### ✅ Good Example

```rust
pub trait Store {
  fn save(&mut self, balance: i64);
}

pub struct Account<S: Store> {
  balance: i64,
  store: S,
}

impl<S: Store> Account<S> {
  pub fn new(store: S) -> Self {
    Self { balance: 0, store }
  }

  pub fn deposit(&mut self, amount: i64) {
    self.balance += amount;
    self.store.save(self.balance);
  }

  pub fn balance(&self) -> i64 {
    self.balance
  }

  pub fn store(&self) -> &S {
    &self.store
  }
}
```

```rust
struct RecorderStore {
  saved: Option<i64>,
}

impl Store for RecorderStore {
  fn save(&mut self, balance: i64) {
    self.saved = Some(balance);
  }
}

#[test]
fn deposit_increases_balance_and_persists_it() {
  // Arrange
  let mut account = Account::new(RecorderStore { saved: None });

  // Act
  account.deposit(100);

  // Assert
  assert_eq!(account.balance(), 100);
  assert_eq!(account.store().saved, Some(100));
}
```

{{< /tab >}}
{{< /tabs >}}

## Resources

* **[Working Effectively with Legacy Code][feathers] by Michael Feathers** -
  covers *seams* and dependency injection for making code testable without
  changing its behavior.

* **[Dependency injection (Wikipedia)][di]** - the pattern used to supply
  collaborators from the outside instead of branching on a test flag.

* **[Functional options for friendly APIs][options] by Dave Cheney** - a Go
  pattern for injecting behavior without exposing it as test-aware configuration.

[feathers]: <https://www.oreilly.com/library/view/working-effectively-with/0131177052/>
[di]: <https://en.wikipedia.org/wiki/Dependency_injection>
[options]: <https://dave.cheney.net/2014/10/17/functional-options-for-friendly-apis>
