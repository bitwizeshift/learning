+++
title = 'Arrange, Act, Assert'
date = 2026-06-25
id = 'TEST-001'
slug = 'TEST-001'
resources = ['best-practice']
toolchains = ['cpp', 'go', 'python', 'rust']
tags = ['unit-testing', 'arrange-act-assert']
concepts = ['testing']
description = 'Structure every unit test in three clear phases, regardless of language.'
toc = true
[focus]
concepts = ['testing']
+++

When writing unit-tests, always follow the three-A's: **Arrange, Act, Assert**.
This divides every test into three distinct phases:

* **Arrange**: set up the test-case, including any inputs, mocks, and expected
  outputs.

* **Act**: perform the one action under test.

* **Assert**: check the outcome of the action, including any return values and
  expected side effects.

{{< tip >}}
Keep exactly **one** action in the Act phase. If you need two, that's usually two
tests.
{{< /tip >}}

<!--more-->

## Motivation

A unit test exists to answer one question: given some state, does a single
action produce the expected result? Without a fixed structure, the three
concerns -- setting up state, invoking the action, and verifying the result --
get interleaved. The reader then has to reconstruct which lines are
preconditions, which line is the behavior under test, and which lines are
checks. Arrange-Act-Assert removes that work by assigning each concern a fixed
position in every test.

## Benefits

* **Readability**: the phase boundaries tell a reader where to look. The Act
  phase identifies the exact behavior under test without reading the rest of the
  test.

* **Maintainability**: each concern lives in one place. Changing a precondition
  touches only the Arrange phase; changing an expectation touches only the
  Assert phase. There is no setup logic buried between assertions to find first.

* **Diagnosability**: when a single action is paired with focused assertions, a
  failure points to one cause. A test with several actions and several
  assertions does not tell you which step failed without further investigation.

* **Detecting test smells**: the structure makes problems visible. A large
  Arrange phase signals that the unit has too many dependencies or does too
  much. More than one action in the Act phase signals that the test is covering
  more than one behavior and should be split. Both are common sources of brittle
  tests.

## Justification

The constraints follow from what a test is supposed to isolate. Restricting Act
to a single action is what makes a failure attributable to one behavior; once a
test exercises two actions, a failure no longer identifies which one broke, and
the test stops being a precise signal. Keeping Arrange and Assert in fixed,
separate positions is what lets the structure expose these problems by
inspection rather than by debugging. Applying the pattern uniformly across every
test means the cost of reading any one test is the same regardless of who wrote
it.

## Examples

{{< tabs >}}
{{< tab icon="cplusplus" label="C++" >}}

```cpp
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

```go
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

```python
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

```rust
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

* **[X-Unit Test Patterns][x-unit] by Gerard Meszaros** - a classic book on unit testing
  that covers the AAA pattern.

* **[Clean Code: A Handbook of Agile Software Craftsmanship][clean-code] by
  Robert C. Martin** -
  a book that covers many best practices for writing clean, maintainable code,
  including testing.

* **[Automation Panda: Arrange-Act-Assert][automation-panda]** - a separate
  resource that addresses the AAA pattern in detail.

[x-unit]: <https://martinfowler.com/books/meszaros.html>
[clean-code]: <https://www.amazon.com/dp/0132350882>
[automation-panda]: <https://automationpanda.com/2020/07/07/arrange-act-assert-a-pattern-for-writing-good-tests/>
