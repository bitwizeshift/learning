+++
title = 'Name tests after the behavior under test'
date = 2026-07-27
id = 'TEST-007'
slug = 'TEST-007'
resources = ['best-practice']
toolchains = ['cpp', 'go', 'python', 'rust']
tags = ['unit-testing', 'naming', 'readability']
concepts = ['testing']
description = 'Name each test after the scenario and expected result, so a failure identifies the broken behavior on its own.'
toc = true
[focus]
concepts = ['testing']
+++

Name a test after the **behavior** it verifies: the scenario under test and the
expected result. The name is the first thing you see when a test fails, so it
should identify the broken behavior without opening the file. A name like
`test_withdraw_1` or `TestAccount` forces you to read the body to learn what
broke; a name that states the behavior does not.

{{< tabs >}}
{{< tab icon="cplusplus" label="C++" >}}

```cpp
TEST_CASE("withdraw more than the balance is rejected") {
  // Arrange
  auto account = Account{/*balance=*/100};

  // Act
  const bool ok = account.withdraw(150);

  // Assert
  REQUIRE_FALSE(ok);
  REQUIRE(account.balance() == 100);
}
```

{{< /tab >}}
{{< tab icon="go" label="Go" >}}

```go
func TestWithdraw_RejectsAmountAboveBalance(t *testing.T) {
  t.Parallel()

  // Arrange
  account := NewAccount(100)

  // Act
  ok := account.Withdraw(150)

  // Assert
  if got, want := ok, false; got != want {
    t.Errorf("Withdraw(150) = %v, want %v", got, want)
  }
  if got, want := account.Balance(), 100; got != want {
    t.Errorf("Balance() = %d, want %d", got, want)
  }
}
```

{{< /tab >}}
{{< tab icon="python" label="Python" >}}

```python
def test_withdraw_more_than_balance_is_rejected():
  # Arrange
  account = Account(balance=100)

  # Act
  ok = account.withdraw(150)

  # Assert
  assert ok is False
  assert account.balance() == 100
```

{{< /tab >}}
{{< tab icon="rust" label="Rust" >}}

```rust
#[test]
fn withdraw_more_than_balance_is_rejected() {
  // Arrange
  let mut account = Account::new(100);

  // Act
  let ok = account.withdraw(150);

  // Assert
  assert!(!ok);
  assert_eq!(account.balance(), 100);
}
```

{{< /tab >}}
{{< /tabs >}}

<!--more-->

## Motivation

Test names are read in two situations, and a body-free name serves both:

* **In a failing run:** a CI log or test runner prints the name of the test that
  failed, and very little additional context. If the name is `test_case_3`, you
  learn only that something failed, and you must open the file, read the setup,
  and infer the intent.

  If the name is `withdraw_more_than_balance_is_rejected`, the log already
  tells you which behavior regressed.

* **Reading the suite as a catalogue**: test names, listed together, enumerate
  the behaviors a unit guarantees. Names built from the method plus a number
  describe the code's structure, not its behavior, so the list documents
  nothing. Names built from scenario and result read as a specification.

## Justification

The value of a test name is the information it carries at the moment of failure.
A name derived from the behavior -- the input condition and the expected outcome
-- carries that information for free, because it restates the contract the test
checks. A name derived from a method or an index carries none of it, and pushes
the work of understanding the failure onto whoever reads the log.

A consistent scheme makes the names predictable. One common shape is
_subject_, _condition_, _expected result_ (`withdraw`, _more than balance_,
_is rejected_). The exact convention matters less than applying one uniformly, so
that every name answers the same question in the same order.

## Examples

The unit under test is an `Account` whose `withdraw` refuses to overdraw. The bad
name says nothing about that behavior; the good name states the scenario and the
result.

{{< tabs >}}
{{< tab icon="cplusplus" label="C++" >}}

### ❌ Bad Example

```cpp
// The name describes neither the scenario nor the expected result.
TEST_CASE("withdraw test 2") {
  Account account{100};
  REQUIRE_FALSE(account.withdraw(150));
}
```

### ✅ Good Example

```cpp
// The name states the scenario (overdraw) and the result (rejected).
TEST_CASE("withdraw more than the balance is rejected") {
  // Arrange
  auto account = Account{/*balance=*/100};

  // Act
  const bool ok = account.withdraw(150);

  // Assert
  REQUIRE_FALSE(ok);
  REQUIRE(account.balance() == 100);
}
```

{{< /tab >}}

{{< tab icon="go" label="Go" >}}

Go convention is `Test<Subject>_<Condition>_<Expectation>`; the leading `Test` is
required by the toolchain, and the suffix carries the behavior.

Go also supports subtests with `t.Run`, which help provide additional details.

### ❌ Bad Example

```go
// A method name plus a number; the log tells you nothing on failure.
func TestWithdraw2(t *testing.T) {
  account := NewAccount(100)
  if account.Withdraw(150) {
    t.Errorf("withdraw succeeded")
  }
}
```

### ✅ Good Example

```go
func TestWithdraw_AmountAboveBalance_RejectsWithdrawal(t *testing.T) {
  t.Parallel()

  // Arrange
  account := NewAccount(100)

  // Act
  ok := account.Withdraw(150)

  // Assert
  if got, want := ok, false; got != want {
    t.Errorf("Withdraw(150) = %v, want %v", got, want)
  }
  if got, want := account.Balance(), 100; got != want {
    t.Errorf("Balance() = %d, want %d", got, want)
  }
}
```

{{< /tab >}}

{{< tab icon="python" label="Python" >}}

### ❌ Bad Example

```python
# Numbered and opaque; the report names the function but not the behavior.
def test_withdraw_2():
  account = Account(balance=100)
  assert not account.withdraw(150)
```

### ✅ Good Example

```python
def test_withdraw_more_than_balance_is_rejected():
  # Arrange
  account = Account(balance=100)

  # Act
  ok = account.withdraw(150)

  # Assert
  assert ok is False
  assert account.balance() == 100
```

{{< /tab >}}

{{< tab icon="rust" label="Rust" >}}

### ❌ Bad Example

```rust
// Says nothing about the condition being tested.
#[test]
fn withdraw_works() {
  let mut account = Account::new(100);
  assert!(!account.withdraw(150));
}
```

### ✅ Good Example

```rust
#[test]
fn withdraw_more_than_balance_is_rejected() {
  // Arrange
  let mut account = Account::new(100);

  // Act
  let ok = account.withdraw(150);

  // Assert
  assert!(!ok);
  assert_eq!(account.balance(), 100);
}
```

{{< /tab >}}
{{< /tabs >}}

## Resources

* **{{< link
    url="https://osherove.com/blog/2005/4/3/naming-standards-for-unit-tests.html"
    text="Naming standards for unit tests"
    icon="link"
    hover="Roy Osherove" >}} by Roy Osherove** - the classic
  _method / state-under-test / expected-behavior_ naming scheme.

* **{{< link
    url="https://abseil.io/resources/swe-book/html/ch12.html"
    text="Software Engineering at Google: Unit Testing"
    icon="link" >}}** - on test names as behavior documentation and writing
  descriptive test names.
