+++
title = 'Avoid creating side-effects in tests'
date = 2026-06-25
id = 'TEST-003'
slug = 'TEST-003'
resources = ['best-practice']
toolchains = ['cpp', 'go', 'python', 'rust']
tags = ['unit-testing', 'isolation']
concepts = ['testing']
description = 'Avoid creating side-effects in unit-tests, including modifying global state or shared resources.'
toc = true
[focus]
concepts = ['testing']
+++

A unit test must not depend on or alter state that lives outside its own scope.
Do not read or mutate global variables, singletons, static fields, the
filesystem, environment variables, or any other resource that another test can
also reach.

{{< tip >}}
If a test only passes when the suite runs in a particular order, it is sharing
state with another test.
{{< /tip >}}

<!--more-->

## Motivation

Tests in a suite run together. When one test mutates shared state, it changes the
starting conditions of every other test that touches that state. The result is a
test whose outcome depends on what ran before it, rather than on the code it is
meant to verify. Such a test can pass alone and fail in the suite, or fail alone
and pass in the suite, with no change to the code under test.

This coupling also blocks parallel execution. Test runners parallelize by
assuming tests are independent; once two tests write to the same state, running
them concurrently introduces a data race and non-deterministic results. The only
way to keep such a suite stable is to force it to run serially in a fixed order,
which is slower and fragile.

## Justification

A test is only useful if its result depends on one thing: the code under test.
Shared state breaks that property by adding a hidden input — the residue left by
earlier tests — that the test does not control or declare. Removing side effects
restores the guarantee that a failure means the code is wrong, not that the
tests ran in an unexpected order.

Independence is also what makes a suite scalable. Tests that own their state can
run in any order and in parallel, so the suite stays fast as it grows and stays
trustworthy when a single test is run in isolation to reproduce a failure.
Order-dependence sacrifices both.

## Examples

{{< tabs >}}
{{< tab icon="cplusplus" label="C++" >}}

### ❌ Bad Example

```cpp
// Shared across tests. Order now matters and parallel runs race.
Account g_account{/*balance=*/100};

TEST_CASE("deposit increases the balance") {
  g_account.deposit(50);
  REQUIRE(g_account.balance() == 150);
}

TEST_CASE("withdraw reduces the balance") {
  // Only correct if the test above ran first and left a balance of 150.
  g_account.withdraw(40);
  REQUIRE(g_account.balance() == 110);
}
```

### ✅ Good Example

```cpp
// Each test owns its state.
TEST_CASE("deposit increases the balance") {
  // Arrange
  Account account{/*balance=*/100};

  // Act
  account.deposit(50);

  // Assert
  REQUIRE(account.balance() == 150);
}

TEST_CASE("withdraw reduces the balance") {
  // Arrange
  Account account{/*balance=*/100};

  // Act
  account.withdraw(40);

  // Assert
  REQUIRE(account.balance() == 60);
}
```

{{< /tab >}}

{{< tab icon="go" label="Go" >}}

### ❌ Bad Example

```go
// Package-level state shared across tests; mutations leak between them.
var account = NewAccount(100)

func TestDeposit(t *testing.T) {
  account.Deposit(50)
  if got, want := account.Balance(), 150; got != want {
    t.Errorf("Balance() = %d, want %d", got, want)
  }
}

func TestWithdraw(t *testing.T) {
  // Only correct if TestDeposit ran first.
  account.Withdraw(40)
  if got, want := account.Balance(), 110; got != want {
    t.Errorf("Balance() = %d, want %d", got, want)
  }
}
```

### ✅ Good Example

```go
// Each test owns its state and can run in parallel.
func TestDeposit(t *testing.T) {
  t.Parallel()

  // Arrange
  account := NewAccount(100)

  // Act
  account.Deposit(50)

  // Assert
  if got, want := account.Balance(), 150; got != want {
    t.Errorf("Balance() = %d, want %d", got, want)
  }
}

func TestWithdraw(t *testing.T) {
  t.Parallel()

  // Arrange
  account := NewAccount(100)

  // Act
  account.Withdraw(40)

  // Assert
  if got, want := account.Balance(), 60; got != want {
    t.Errorf("Balance() = %d, want %d", got, want)
  }
}
```

{{< /tab >}}

{{< tab icon="python" label="Python" >}}

### ❌ Bad Example

```python
# Module-level state shared across tests; mutations leak between them.
account = Account(balance=100)

def test_deposit():
  account.deposit(50)
  assert account.balance() == 150

def test_withdraw():
  # Only correct if test_deposit ran first.
  account.withdraw(40)
  assert account.balance() == 110
```

### ✅ Good Example

```python
# Each test owns its state.
def test_deposit():
  # Arrange
  account = Account(balance=100)

  # Act
  account.deposit(50)

  # Assert
  assert account.balance() == 150

def test_withdraw():
  # Arrange
  account = Account(balance=100)

  # Act
  account.withdraw(40)

  # Assert
  assert account.balance() == 60
```

{{< /tab >}}

{{< tab icon="rust" label="Rust" >}}

### ❌ Bad Example

```rust
// Shared mutable state forces `unsafe` and leaks between tests.
static mut ACCOUNT: Account = Account::new(100);

#[test]
fn deposit_increases_balance() {
  unsafe {
    ACCOUNT.deposit(50);
    assert_eq!(ACCOUNT.balance(), 150);
  }
}

#[test]
fn withdraw_reduces_balance() {
  unsafe {
    // Only correct if deposit_increases_balance ran first.
    ACCOUNT.withdraw(40);
    assert_eq!(ACCOUNT.balance(), 110);
  }
}
```

### ✅ Good Example

```rust
// Each test owns its state.
#[test]
fn deposit_increases_balance() {
  // Arrange
  let mut account = Account::new(100);

  // Act
  account.deposit(50);

  // Assert
  assert_eq!(account.balance(), 150);
}

#[test]
fn withdraw_reduces_balance() {
  // Arrange
  let mut account = Account::new(100);

  // Act
  account.withdraw(40);

  // Assert
  assert_eq!(account.balance(), 60);
}
```

{{< /tab >}}
{{< /tabs >}}

## Resources

* **[X-Unit Test Patterns][x-unit] by Gerard Meszaros** - describes the *Shared
  Fixture* and *Erratic Test* smells that result from tests sharing state.

* **[Eradicating Non-Determinism in Tests][non-determinism] by Martin Fowler** -
  covers shared state as a primary cause of flaky, order-dependent tests.

[x-unit]: <https://martinfowler.com/books/meszaros.html>
[non-determinism]: <https://martinfowler.com/articles/nonDeterminism.html>
