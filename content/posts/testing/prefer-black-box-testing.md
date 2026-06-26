+++
title = 'Prefer black-box testing'
date = 2026-06-25
id = 'TEST-005'
slug = 'TEST-005'
resources = ['best-practice']
toolchains = ['cpp', 'go', 'python', 'rust']
tags = ['unit-testing', 'black-box-testing', 'encapsulation']
concepts = ['testing', 'design']
description = 'Test through public interfaces, not private implementation details.'
toc = true
[focus]
concepts = ['testing']
+++

Prefer testing the **public interface** of a unit over its private
implementation details. A test should exercise what a unit promises to its
callers, not how it achieves it internally. This style is called **black-box
testing**: the tester knows the external behavior but treats the internal
structure as opaque.

{{< tip >}}
If you find yourself needing to reach into private helpers to test them, that is
usually a design smell. Consider extracting those helpers into their own unit
with its own public interface, and test them there.
{{< /tip >}}

<!--more-->

## Motivation

Private helpers are implementation details, and implementation details change.
A test bound to a private function breaks whenever that function is renamed,
split, merged, or inlined -- even when the unit's observable behavior is
unchanged. The test then obstructs refactoring instead of enabling it: every
internal cleanup comes with test edits that prove nothing new.

Testing through the public interface also keeps tests aligned with what actually
matters: callers only ever depend on the public behavior, so that is the surface
worth pinning down. Private helpers are reached transitively through the public
API; exercising the public API with enough cases covers them as a side effect,
without naming them.

## Justification

Black-box tests are resilient to internal change because they depend only on the
contract, which is the part of a unit that is supposed to be stable. This is what
lets a test suite double as a safety net for refactoring: behavior-preserving
changes keep the tests green, and only a genuine change in behavior turns them
red.

The need to test a private detail is frequently a signal that the unit is doing
**too much**. A helper complex enough to warrant its own tests is usually a unit
in its own right that has not been extracted yet. Pulling it out into a separate,
independently testable unit resolves the pressure to test privates and improves
the factoring at the same time. Reaching through encapsulation to test internals
treats the symptom and leaves the design problem in place.

## Examples

The unit under test is a `slugify` function that converts a title into a
URL-friendly slug (`"Hello, World!"` -> `"hello-world"`). It is built from private
helpers -- lowercasing, stripping punctuation, collapsing separators -- that are
implementation details, not part of its contract.

The mechanics of "private" differ by language, so the black-box boundary is
enforced differently in each. The examples below note the relevant semantics.

{{<details summary="C++ example">}}

C++ enforces access at compile time: a test simply cannot call a `private` member
from outside the class. The only way is to declare the test a `friend` which
couple the test to the class's internals. The black-box approach is to exercise
the class through its public methods only.

```cpp
// slug.hpp
class Slugifier {
public:
  auto make(const std::string& title) const -> std::string
private:
  auto strip_punctuation(const std::string& s) const -> std::string;
  auto collapse_separators(const std::string& s) const -> std::string;
};
```

### ❌ Bad Example

```cpp
// Befriending the test leaks internals just to make them testable.
class Slugifier {
public:
  auto make(const std::string& title) const -> std::string
private:
  friend struct SlugifierAccess;  // <-- exposes private members to the test
  auto strip_punctuation(const std::string& s) const -> std::string;
};

struct SlugifierAccess {
  static auto strip(const Slugifier& s, const std::string& in) -> std::string {
    return s.strip_punctuation(in);
  }
};

TEST_CASE("strip_punctuation removes commas") {
  Slugifier slugifier;
  REQUIRE(SlugifierAccess::strip(slugifier, "a,b") == "a b");
}
```

### ✅ Good Example

```cpp
// Tests the public method; the private helpers are covered through it.
TEST_CASE("make produces a url slug") {
  // Arrange
  const Slugifier slugifier;
  const std::string title = "Hello, World!";

  // Act
  const std::string slug = slugifier.make(title);

  // Assert
  REQUIRE(slug == "hello-world");
}
```

{{</details>}}

{{<details summary="Go example">}}

Go enforces the boundary through packages. A test in the external `slug_test`
package can only reach exported identifiers, which makes black-box testing the
default when you use that package. As a bonus, helpers defined in `slug_test` do
not count toward the coverage of the package under test, so they cannot inflate
its coverage numbers. A same-package (`package slug`) test, by contrast, can call
unexported functions directly.

```go
package slug

// Make converts a title into a URL-friendly slug.
func Make(title string) string {
  s := strings.ToLower(title)
  s = stripPunctuation(s)
  s = collapseSeparators(s)
  return strings.Trim(s, "-")
}

func stripPunctuation(s string) string  { /* ... */ }
func collapseSeparators(s string) string { /* ... */ }
```

### ❌ Bad Example

```go
package slug // same package: can call unexported helpers directly

// Pins an implementation detail; renaming or inlining the helper breaks this.
func TestStripPunctuation(t *testing.T) {
  if got, want := stripPunctuation("a,b"), "a b"; got != want {
    t.Errorf("stripPunctuation(%q) = %q, want %q", "a,b", got, want)
  }
}
```

### ✅ Good Example

```go
package slug_test // external package: only the exported API is visible

func TestMake(t *testing.T) {
  // Arrange
  testCases := []struct {
    name  string
    title string
    want  string
  }{
    {
      name: "lowercases and joins words",
      title: "Hello World",
      want: "hello-world"
    }, {
      name: "drops punctuation",
      title: "Hello, World!",
      want: "hello-world",
    }, {
      name: "collapses repeated separators",
      title: "a  --  b",
      want: "a-b",
    },
  }

  for _, tc := range testCases {
    t.Run(tc.name, func(t *testing.T) {
      // Act
      s := slug.Make(tc.title)

      // Assert
      if got, want := s, tc.want; got != want {
        t.Errorf("Make(%q) = %q, want %q", tc.title, got, tc.want)
      }
    })
  }
}
```

{{</details>}}

{{<details summary="Python example">}}

Python does not enforce privacy. A leading underscore (`_strip_punctuation`) is a
convention signalling "internal", and a double underscore only triggers name
mangling, not protection -- a test can reach either. Black-box testing is
therefore a discipline: import and exercise only the public names, and treat
underscore-prefixed members as off-limits even though the language allows access.

```python
# slug.py
def slugify(title: str) -> str:
  """Convert a title into a URL-friendly slug."""
  s = title.lower()
  s = _strip_punctuation(s)
  s = _collapse_separators(s)
  return s.strip("-")

def _strip_punctuation(s: str) -> str: ...
def _collapse_separators(s: str) -> str: ...
```

### ❌ Bad Example

```python
# Imports a private helper; the leading underscore says "do not depend on this".
from slug import _strip_punctuation

def test_strip_punctuation():
  assert _strip_punctuation("a,b") == "a b"
```

### ✅ Good Example

```python
from slug import slugify

def test_slugify_produces_a_url_slug():
  # Arrange
  title = "Hello, World!"

  # Act
  result = slugify(title)

  # Assert
  assert result == "hello-world"
```

{{</details>}}

{{<details summary="Rust example">}}

Rust deviates from the others: a child module can see its parent's private items,
so an in-module `#[cfg(test)] mod tests` block *can* call private functions. That
makes white-box tests easy to write by accident. The black-box approach is to put
behavior tests in the `tests/` integration directory, which is compiled as a
separate crate and can only see the public API of your crate.

```rust
// src/lib.rs
pub fn slugify(title: &str) -> String {
  let s = title.to_lowercase();
  let s = strip_punctuation(&s);
  let s = collapse_separators(&s);
  s.trim_matches('-').to_string()
}

fn strip_punctuation(s: &str) -> String { /* ... */ }
fn collapse_separators(s: &str) -> String { /* ... */ }
```

### ❌ Bad Example

```rust
// src/lib.rs -- an in-module test can reach private items.
#[cfg(test)]
mod tests {
  use super::*;

  // Pins a private helper; refactoring its name or signature breaks this.
  #[test]
  fn strip_punctuation_removes_commas() {
    assert_eq!(strip_punctuation("a,b"), "a b");
  }
}
```

### ✅ Good Example

```rust
// tests/slug.rs -- a separate crate that sees only the public API.
use slug::slugify;

#[test]
fn slugify_produces_a_url_slug() {
  // Arrange
  let title = "Hello, World!";

  // Act
  let slug = slugify(title);

  // Assert
  assert_eq!(slug, "hello-world");
}
```

{{</details>}}

## Resources

* **[Black-box testing (Wikipedia)][black-box]** - definition of black-box
  testing and its contrast with white-box testing.

* **[`testing` package documentation][go-testing]** - Go standard library; the
  `_test` external test package used to enforce black-box testing.

* **[X-Unit Test Patterns][x-unit] by Gerard Meszaros** - covers testing through
  the public interface and the smells that arise from coupling tests to
  internals.

[black-box]: <https://en.wikipedia.org/wiki/Black-box_testing>
[go-testing]: <https://pkg.go.dev/testing>
[x-unit]: <https://martinfowler.com/books/meszaros.html>
