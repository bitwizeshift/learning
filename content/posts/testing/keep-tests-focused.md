+++
title = 'Keep tests small and focused'
date = 2026-06-25
id = 'TEST-002'
slug = 'TEST-002'
resources = ['best-practice']
toolchains = ['cpp', 'go', 'python', 'rust']
tags = ['unit-testing', 'single-responsibility-principle']
concepts = ['testing', 'design']
description = 'Keep unit-tests small and focused on a single behavior.'
toc = true
[focus]
concepts = ['testing']
+++

Each unit test should verify **one** specific behavior. Avoid exercising
multiple behaviors or features within a single test case.

{{< tip >}}
A useful heuristic: if you cannot describe what a test checks without using
"and", it is testing more than one behavior and should be split.
{{< /tip >}}

<!--more-->

## Motivation

When a test verifies several behaviors at once, a failure only tells you that
_something_ in that group broke, not _which_ behavior. You then have to read the
test and reproduce its steps to find the cause. A test scoped to one behavior
names the failure on its own: the test that broke is the behavior that broke.

Broad tests also fail for more reasons; a test that touches several behaviors
breaks whenever any of them changes, including changes unrelated to what the
test was meant to protect. This produces failures that are noise rather than
signal, and the test gets edited or ignored instead of trusted.

## Justification

The value of a test is the precision of the signal it gives when it fails. One
behavior per test maximizes that precision: a red test maps to a single,
identified behavior, so debugging starts from a known cause instead of a search.
It also keeps the test's reasons to fail aligned with its purpose -- it changes
when its behavior changes, and not otherwise -- which is what makes a suite
worth keeping green.

Focused tests compose better as documentation. A reader scanning the test names
sees an enumerated list of behaviors the unit guarantees. A test that bundles
several behaviors hides them behind one name and that list becomes incomplete.

## Examples

{{< tabs >}}
{{< tab icon="cplusplus" label="C++" >}}

### ❌ Bad Example

```cpp
// One test, two behaviors. A failure does not say which.
TEST_CASE("account operations") {
  Account account{/*balance=*/100};
  account.deposit(50);
  REQUIRE(account.balance() == 150);
  account.withdraw(40);
  REQUIRE(account.balance() == 110);
}
```

### ✅ Good Example

```cpp
// One behavior per test.
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
// One test, two behaviors. A failure does not say which.
func TestAccountOperations(t *testing.T) {
  account := NewAccount(100)
  account.Deposit(50)
  if got, want := account.Balance(), 150; got != want {
    t.Errorf("Balance() = %d, want %d", got, want)
  }
  account.Withdraw(40)
  if got, want := account.Balance(), 110; got != want {
    t.Errorf("Balance() = %d, want %d", got, want)
  }
}
```

### ✅ Good Example

```go
// One behavior per test.
func TestDepositIncreasesBalance(t *testing.T) {
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

func TestWithdrawReducesBalance(t *testing.T) {
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
# One test, two behaviors. A failure does not say which.
def test_account_operations():
  account = Account(balance=100)
  account.deposit(50)
  assert account.balance() == 150
  account.withdraw(40)
  assert account.balance() == 110
```

### ✅ Good Example

```python
# One behavior per test.
def test_deposit_increases_balance():
  # Arrange
  account = Account(balance=100)

  # Act
  account.deposit(50)

  # Assert
  assert account.balance() == 150

def test_withdraw_reduces_balance():
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
// One test, two behaviors. A failure does not say which.
#[test]
fn account_operations() {
  let mut account = Account::new(100);
  account.deposit(50);
  assert_eq!(account.balance(), 150);
  account.withdraw(40);
  assert_eq!(account.balance(), 110);
}
```

### ✅ Good Example

```rust
// One behavior per test.
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

* **[X-Unit Test Patterns][x-unit] by Gerard Meszaros** - a classic book on unit
  testing.

* **[Clean Code: A Handbook of Agile Software Craftsmanship][clean-code] by
  Robert C. Martin** - covers the single-responsibility principle as applied to
  tests, including one assertion concept per test.

[x-unit]: <https://martinfowler.com/books/meszaros.html>
[clean-code]: <https://www.amazon.com/dp/0132350882>
