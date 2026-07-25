+++
title = 'Fixture'
date = 2026-06-26
slug = 'fixture'
resources = ['glossary']
description = 'The fixed baseline state and setup a test runs against, established before the action under test.'
toc = false
+++

A **fixture** is the fixed, known state a test is run against — the objects,
data, and {{< glossary term="collaborator" text="collaborators" >}} prepared
during the *Arrange* phase before the behavior under test executes. A good
fixture is minimal and explicit, so the reader can see exactly which
preconditions matter to the test.

<!--more-->

Many frameworks provide a mechanism for sharing fixture setup (for example,
pytest fixtures or per-test setup hooks), but the goal is always the same: put
the system into a predictable starting state without leaking that state between
tests.

{{< note >}}
Fixtures do not require first-hand support from the underlying testing framework;
the idea is more abstract than that. More generally, fixtures are just
test-helpers that setup the test in a clear and articulate way to convey the
minimal details needed to convey the conditions of the test.
{{< /note >}}

## Why it matters

A fixture is the premise of every assertion the test makes, so its shape decides
how much the test can tell you. When the arrangement carries only the
preconditions the behavior actually reads, a failure points straight at a known
cause and the test doubles as a specification -- the Arrange section _is_ the
list of conditions under which the result is expected to hold.

The failure mode is a fixture padded with detail that has nothing to do with the
behavior under test. Every extra field and every incidental collaborator is a
value the reader has to consider and then rule out, because nothing distinguishes
the input the assertion depends on from the decoration around it. The test still
passes, but it no longer documents anything -- and when it breaks, you cannot
tell whether the behavior regressed or an unrelated part of the setup drifted.

Keeping a fixture minimal and explicit is what avoids that. State the
preconditions the test relies on where the test can see them, and push the
incidental defaults behind a named builder so each test states only what makes
its case different.

## Example

Consider a pricing rule: Gold-tier customers get 10% off the order subtotal.

{{< tabs >}}
{{< tab icon="cplusplus" label="C++" >}}

```cpp
enum class Tier { Standard, Gold };

struct Customer {
  std::string name;
  std::string email;
  std::string address;
  Tier tier;
  int member_since;
};

struct Item {
  std::string name;
  int price;
};

// Gold-tier customers get 10% off the subtotal.
auto order_total(const Customer& customer, const std::vector<Item>& items)
    -> int {
  int subtotal = 0;
  for (const auto& item : items) {
    subtotal += item.price;
  }
  return customer.tier == Tier::Gold ? subtotal * 9 / 10 : subtotal;
}
```

{{< /tab >}}
{{< tab icon="go" label="Go" >}}

```go
package orders

type Tier int

const (
  Standard Tier = iota
  Gold
)

type Customer struct {
  Name        string
  Email       string
  Address     string
  Tier        Tier
  MemberSince int
}

type Item struct {
  Name  string
  Price int
}

// OrderTotal gives Gold-tier customers 10% off the subtotal.
func OrderTotal(customer Customer, items []Item) int {
  subtotal := 0
  for _, item := range items {
    subtotal += item.Price
  }
  if customer.Tier == Gold {
    return subtotal * 9 / 10
  }
  return subtotal
}
```

{{< /tab >}}
{{< tab icon="python" label="Python" >}}

```python
import enum
from dataclasses import dataclass


class Tier(enum.Enum):
  STANDARD = enum.auto()
  GOLD = enum.auto()


@dataclass
class Customer:
  name: str
  email: str
  address: str
  tier: Tier
  member_since: int


@dataclass
class Item:
  name: str
  price: int


def order_total(customer: Customer, items: list[Item]) -> int:
  # Gold-tier customers get 10% off the subtotal.
  subtotal = sum(item.price for item in items)
  return subtotal * 9 // 10 if customer.tier is Tier.GOLD else subtotal
```

{{< /tab >}}
{{< tab icon="rust" label="Rust" >}}

```rust
pub enum Tier {
  Standard,
  Gold,
}

pub struct Customer {
  pub name: String,
  pub email: String,
  pub address: String,
  pub tier: Tier,
  pub member_since: u32,
}

pub struct Item {
  pub name: String,
  pub price: i64,
}

/// Gold-tier customers get 10% off the subtotal.
pub fn order_total(customer: &Customer, items: &[Item]) -> i64 {
  let subtotal: i64 = items.iter().map(|item| item.price).sum();
  match customer.tier {
    Tier::Gold => subtotal * 9 / 10,
    Tier::Standard => subtotal,
  }
}
```

{{< /tab >}}
{{< /tabs >}}

### Before

A first cut at the test arranges a fully-populated customer and a realistic
basket:

{{< tabs >}}
{{< tab icon="cplusplus" label="C++" >}}

```cpp
TEST_CASE("Gold customers get 10% off") {
  // Arrange
  Customer customer{
      "Ada Lovelace", "ada@example.com", "12 Baker St", Tier::Gold, 2019};
  std::vector<Item> items{
      {"Mechanical Keyboard", 70},
      {"USB-C Cable", 15},
      {"Mouse Pad", 15},
  };

  // Act
  int total = order_total(customer, items);

  // Assert
  REQUIRE(total == 90);
}
```

{{< /tab >}}
{{< tab icon="go" label="Go" >}}

```go
func TestOrderTotal_GoldDiscount(t *testing.T) {
  t.Parallel()

  // Arrange
  customer := Customer{
    Name:        "Ada Lovelace",
    Email:       "ada@example.com",
    Address:     "12 Baker St",
    Tier:        Gold,
    MemberSince: 2019,
  }
  items := []Item{
    {Name: "Mechanical Keyboard", Price: 70},
    {Name: "USB-C Cable", Price: 15},
    {Name: "Mouse Pad", Price: 15},
  }

  // Act
  total := OrderTotal(customer, items)

  // Assert
  if got, want := total, 90; got != want {
    t.Errorf("OrderTotal() = %d, want %d", got, want)
  }
}
```

{{< /tab >}}
{{< tab icon="python" label="Python" >}}

```python
def test_gold_customer_gets_discount():
  # Arrange
  customer = Customer(
    name="Ada Lovelace",
    email="ada@example.com",
    address="12 Baker St",
    tier=Tier.GOLD,
    member_since=2019,
  )
  items = [
    Item(name="Mechanical Keyboard", price=70),
    Item(name="USB-C Cable", price=15),
    Item(name="Mouse Pad", price=15),
  ]

  # Act
  total = order_total(customer, items)

  # Assert
  assert total == 90
```

{{< /tab >}}
{{< tab icon="rust" label="Rust" >}}

```rust
#[test]
fn gold_customer_gets_discount() {
  // Arrange
  let customer = Customer {
    name: "Ada Lovelace".to_string(),
    email: "ada@example.com".to_string(),
    address: "12 Baker St".to_string(),
    tier: Tier::Gold,
    member_since: 2019,
  };
  let items = vec![
    Item { name: "Mechanical Keyboard".to_string(), price: 70 },
    Item { name: "USB-C Cable".to_string(), price: 15 },
    Item { name: "Mouse Pad".to_string(), price: 15 },
  ];

  // Act
  let total = order_total(&customer, &items);

  // Assert
  assert_eq!(total, 90);
}
```

{{< /tab >}}
{{< /tabs >}}

The test passes, but nothing in it says which inputs the `90` depends on. A
reader has to know that the name, email, address, and membership year are all
irrelevant, and that the three items matter only insofar as their prices sum to
`100`. The one precondition the behavior actually reads -- `Tier::Gold` -- is
buried in the middle of the constructor, and the expected discount is left for
the reader to reverse-engineer from the arithmetic.

### After

The fixture strips the arrangement down to the preconditions the behavior reads,
and moves the incidental defaults behind a named builder:

{{< tabs >}}
{{< tab icon="cplusplus" label="C++" >}}

```cpp
// A fixture builder: irrelevant fields get defaults.
auto a_customer(Tier tier) -> Customer {
  return {"Test Customer", "test@example.com", "N/A", tier, 0};
}

TEST_CASE("Gold customers get 10% off") {
  // Arrange
  Customer customer = a_customer(Tier::Gold);
  std::vector<Item> items{{"Widget", 100}};

  // Act
  int total = order_total(customer, items);

  // Assert
  REQUIRE(total == 90);  // 10% off 100
}
```

{{< /tab >}}
{{< tab icon="go" label="Go" >}}

```go
// A fixture builder: irrelevant fields get defaults.
func aCustomer(tier Tier) Customer {
  return Customer{
    Name:        "Test Customer",
    Email:       "test@example.com",
    Address:     "N/A",
    Tier:        tier,
    MemberSince: 0,
  }
}

func TestOrderTotal_GoldDiscount(t *testing.T) {
  t.Parallel()

  // Arrange
  customer := aCustomer(Gold)
  items := []Item{{Name: "Widget", Price: 100}}

  // Act
  total := OrderTotal(customer, items)

  // Assert
  if got, want := total, 90; got != want { // 10% off 100
    t.Errorf("OrderTotal() = %d, want %d", got, want)
  }
}
```

{{< /tab >}}
{{< tab icon="python" label="Python" >}}

```python
# A fixture builder: irrelevant fields get defaults.
def a_customer(tier: Tier) -> Customer:
  return Customer(
    name="Test Customer",
    email="test@example.com",
    address="N/A",
    tier=tier,
    member_since=0,
  )


def test_gold_customer_gets_discount():
  # Arrange
  customer = a_customer(tier=Tier.GOLD)
  items = [Item(name="Widget", price=100)]

  # Act
  total = order_total(customer, items)

  # Assert
  assert total == 90  # 10% off 100
```

{{< /tab >}}
{{< tab icon="rust" label="Rust" >}}

```rust
// A fixture builder: irrelevant fields get defaults.
fn a_customer(tier: Tier) -> Customer {
  Customer {
    name: "Test Customer".to_string(),
    email: "test@example.com".to_string(),
    address: "N/A".to_string(),
    tier,
    member_since: 0,
  }
}

#[test]
fn gold_customer_gets_discount() {
  // Arrange
  let customer = a_customer(Tier::Gold);
  let items = vec![Item { name: "Widget".to_string(), price: 100 }];

  // Act
  let total = order_total(&customer, &items);

  // Assert
  assert_eq!(total, 90); // 10% off 100
}
```

{{< /tab >}}
{{< /tabs >}}

Now the Arrange section reads as the specification of the case: a Gold-tier
customer and a subtotal of `100`, and nothing else. `a_customer` owns the
defaults every test would otherwise repeat, so each test passes only the field
that makes its case different -- here, the tier -- and the round `100` makes the
expected `90` obvious at a glance. If this test fails, it can only be because the
Gold discount changed, which is exactly the signal a fixture should give.
