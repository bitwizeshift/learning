+++
title = 'Test Double'
date = 2026-06-26
slug = 'test-double'
resources = ['glossary']
tags = ['test-double', 'testing']
toc = false
+++

**Test double** is the umbrella term for any object that stands in for a real
collaborator during a test -- just as a stunt double stands in for an actor. The
family includes dummies, stubs, spies, fakes, and
{{< glossary term="mock" text="mocks" >}}, each substituting a real dependency to
keep the unit under test isolated and deterministic.

<!--more-->

Test doubles allow a collaborator to be
{{< glossary term="dependency-injection" text="injected" >}} rather than
constructed internally.

## The family

The term covers a spectrum of substitutes, ordered here roughly from least to
most behavior:

* **{{< glossary term="stub" text="Stub" >}}** -- returns canned answers to the
  calls made during a test. It has no assertions of its own.
* **{{< glossary term="spy" text="Spy" >}}** -- a stub that also records _how_ it
  was called, so the test can inspect those calls afterwards.
* **{{< glossary term="fake" text="Fake" >}}** -- a real, working implementation
  that takes a shortcut unsuitable for production, such as an in-memory store
  standing in for a database.
* **{{< glossary term="mock" text="Mock" >}}** -- preconfigured with
  _expectations_ about the calls it should receive, and fails the test if those
  expectations are not met.

## Prefer realistic behavior

There are two ways to verify a unit's work. _State verification_ asks "what is
the observable result of the action?" and inspects the outcome. _Interaction
verification_ asks "which calls did the unit make?" and inspects the
conversation between collaborators.

State verification is almost always the better default, because it asserts on
the observable _behavior_ of the system rather than its _implementation_. A test
that only checks "method `X` was called with argument `Y`" is coupled to _how_
the code currently works; rename the method, batch two calls into one, or reorder
the steps and the test breaks even though the behavior is unchanged. That is the
core problem with leaning on {{< glossary term="mock" text="mocks" >}}: they push
you toward interaction verification and bind the test to the current shape of the
code.

For that reason, prefer the most _realistic_ double that still keeps the test
deterministic. A {{< glossary term="fake" text="fake" >}} is usually that double: it behaves like the real
collaborator -- returning real answers to real inputs -- so the test can assert
on a genuine outcome instead of a call log. Reserve mocks for the cases where the
interaction genuinely _is_ the behavior under test (for example, "an email is
sent when an order ships"), where there is no resulting state to observe.

## Example

Consider a `total` function that prices an order through a `Catalog` abstraction.
A {{< glossary term="mock" text="mock" >}} would assert that the catalog was
asked for each SKU -- pinning the lookup logic in place. A _fake_ catalog with
real prices lets the test assert what actually matters: the total that comes out.
The double stays out of the assertion entirely; only the observable result is
checked.

{{< tabs >}}
{{< tab icon="cplusplus" label="C++" >}}

```cpp
// checkout.hpp -- the abstraction and the unit under test.

// Catalog yields the unit price, in cents, of a product.
class Catalog {
public:
  virtual ~Catalog() = default;
  virtual auto unit_price(const std::string& sku) const -> int = 0;
};

struct Line {
  std::string sku;
  int quantity;
};

auto total(const Catalog& catalog, const std::vector<Line>& lines) -> int;
```

```cpp
// A fake: a working Catalog backed by a map of fixed prices.
class FakeCatalog final : public Catalog {
public:
  explicit FakeCatalog(std::map<std::string, int> prices)
    : m_prices(std::move(prices)) {}

  auto unit_price(const std::string& sku) const -> int override {
    return m_prices.at(sku);
  }

private:
  std::map<std::string, int> m_prices;
};

TEST_CASE("total sums the priced line items") {
  // Arrange
  const FakeCatalog catalog{{{"APPLE", 50}, {"BREAD", 220}}};
  const std::vector<Line> lines{{"APPLE", 3}, {"BREAD", 1}};

  // Act
  const int result = total(catalog, lines);

  // Assert
  REQUIRE(result == 370);
}
```

{{< /tab >}}
{{< tab icon="go" label="Go" >}}

```go
package checkout

// Catalog yields the unit price, in cents, of a product.
type Catalog interface {
  UnitPrice(sku string) int
}

// Line is a requested quantity of a product.
type Line struct {
  SKU      string
  Quantity int
}

// Total returns the price, in cents, of the given lines.
func Total(catalog Catalog, lines []Line) int {
  var total int
  for _, line := range lines {
    total += catalog.UnitPrice(line.SKU) * line.Quantity
  }
  return total
}
```

```go
package checkout_test // external package: only the exported API is visible

// fakeCatalog is a working Catalog backed by a table of fixed prices.
type fakeCatalog map[string]int

func (c fakeCatalog) UnitPrice(sku string) int { return c[sku] }

func TestTotal(t *testing.T) {
  t.Parallel()

  // Arrange
  catalog := fakeCatalog{"APPLE": 50, "BREAD": 220}
  testCases := []struct {
    name  string
    lines []checkout.Line
    want  int
  }{
    {
      name:  "sums the priced line items",
      lines: []checkout.Line{{SKU: "APPLE", Quantity: 3}, {SKU: "BREAD", Quantity: 1}},
      want:  370,
    }, {
      name:  "an empty order costs nothing",
      lines: nil,
      want:  0,
    },
  }

  for _, tc := range testCases {
    t.Run(tc.name, func(t *testing.T) {
      // Act
      total := checkout.Total(catalog, tc.lines)

      // Assert
      if got, want := total, tc.want; !cmp.Equal(got, want) {
        t.Errorf("Total(%v) = %d, want %d", tc.lines, got, want)
      }
    })
  }
}
```

{{< /tab >}}
{{< tab icon="python" label="Python" >}}

```python
# checkout.py -- the abstraction and the unit under test.
from dataclasses import dataclass
from typing import Protocol


# Catalog yields the unit price, in cents, of a product.
class Catalog(Protocol):
  def unit_price(self, sku: str) -> int: ...


@dataclass(frozen=True)
class Line:
  sku: str
  quantity: int


def total(catalog: Catalog, lines: list[Line]) -> int:
  return sum(catalog.unit_price(line.sku) * line.quantity for line in lines)
```

```python
from checkout import Line, total


# A fake: a working Catalog backed by a dict of fixed prices.
class FakeCatalog:
  def __init__(self, prices: dict[str, int]) -> None:
    self._prices = prices

  def unit_price(self, sku: str) -> int:
    return self._prices[sku]


def test_total_sums_the_priced_line_items():
  # Arrange
  catalog = FakeCatalog({"APPLE": 50, "BREAD": 220})
  lines = [Line("APPLE", 3), Line("BREAD", 1)]

  # Act
  result = total(catalog, lines)

  # Assert
  assert result == 370
```

{{< /tab >}}
{{< tab icon="rust" label="Rust" >}}

```rust
// src/lib.rs -- the abstraction and the unit under test.

/// Catalog yields the unit price, in cents, of a product.
pub trait Catalog {
  fn unit_price(&self, sku: &str) -> u32;
}

pub struct Line {
  pub sku: String,
  pub quantity: u32,
}

/// Returns the price, in cents, of the given lines.
pub fn total(catalog: &impl Catalog, lines: &[Line]) -> u32 {
  lines
    .iter()
    .map(|line| catalog.unit_price(&line.sku) * line.quantity)
    .sum()
}
```

```rust
// tests/checkout.rs -- a separate crate that sees only the public API.
use std::collections::HashMap;

use checkout::{total, Catalog, Line};

// A fake: a working Catalog backed by a map of fixed prices.
struct FakeCatalog(HashMap<String, u32>);

impl Catalog for FakeCatalog {
  fn unit_price(&self, sku: &str) -> u32 {
    self.0[sku]
  }
}

#[test]
fn total_sums_the_priced_line_items() {
  // Arrange
  let catalog = FakeCatalog(HashMap::from([
    ("APPLE".to_string(), 50),
    ("BREAD".to_string(), 220),
  ]));
  let lines = [
    Line { sku: "APPLE".to_string(), quantity: 3 },
    Line { sku: "BREAD".to_string(), quantity: 1 },
  ];

  // Act
  let result = total(&catalog, &lines);

  // Assert
  assert_eq!(result, 370);
}
```

{{< /tab >}}
{{< /tabs >}}

{{< note >}}
Notice the assertion names neither the catalog nor its `unit_price` calls -- it
checks only the total that comes out. The fake could cache, batch, or reorder its
lookups and the test would not care, because none of that is the behavior under
test.
{{< /note >}}

## References

* {{< link
    url="https://martinfowler.com/articles/mocksArentStubs.html"
    text="Mocks Aren't Stubs"
    icon="link"
    hover="Martin Fowler" >}} -- Fowler's article separating state verification
  from interaction verification, and the double taxonomy behind it.
* {{< link
    url="http://xunitpatterns.com/Test%20Double.html"
    text="Test Double"
    icon="link"
    hover="xUnit Patterns" >}} -- Gerard Meszaros' canonical catalog of dummy,
  stub, spy, fake, and mock.
* {{< link
    url="https://en.wikipedia.org/wiki/Test_double"
    text="Test double"
    icon="link"
    hover="Wikipedia" >}} -- short overview of the term and its members.
