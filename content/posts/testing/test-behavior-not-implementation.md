+++
title = 'Test behavior, not implementation'
date = 2026-06-25
id = 'TEST-004'
slug = 'TEST-004'
resources = ['best-practice']
toolchains = ['cpp', 'go', 'python', 'rust']
tags = ['unit-testing', 'behavior']
concepts = ['testing']
description = 'Test the observable behavior of code, not the way that behavior is implemented.'
toc = true
[focus]
concepts = ['testing']
+++

Write tests against the **observable behavior** of code, not the way that
behavior is implemented. Avoid expectations that restate the implementation --
most commonly an expected value copied verbatim from the function body. Such a
test passes by construction and verifies nothing about correctness.

{{< tip >}}
If the only way to derive the expected value is to read the implementation, you
are testing the implementation. Assert a documented property instead.
{{< /tip >}}

<!--more-->

## Motivation

A test that restates the implementation gives a false sense of security. It does
not check that the code is correct; it checks that the code matches itself,
which it always will. A bug in the implementation is copied into the expectation,
so the test stays green while the behavior is wrong.

These tests are also brittle. Because the expectation is tied to *how* the code
works rather than *what* it guarantees, any refactor that changes the
implementation -- even one that preserves behavior -- forces a matching edit to
the test. A test like this is a **change-detector test**: it fails whenever the
code changes, not whenever the behavior is wrong. It reports churn, not
regressions, and stops being a reliable signal.

## Justification

The purpose of a test is to detect when behavior is wrong. An expectation derived
from the implementation cannot do that, because it changes in lockstep with the
code it is meant to police. Asserting a property the code *promises* -- a
documented guarantee, an invariant, or an observable result -- decouples the test
from the implementation, so the test fails when the promise is broken and passes
otherwise, regardless of how the code is written internally.

When a function has no well-established expected values to assert against, test
properties of its output instead. Good candidates are facts that are either
**observable** (a documented prefix, a value in range, a parseable shape) or
**deterministic** (the same input yields the same output, distinct inputs yield
distinct outputs). These hold across implementations, so they survive refactors
and still catch real defects.

## Examples

The function under test returns a temporary file path. Its documented contract is
that the path is under `/tmp/`; the exact filename is an implementation detail,
not a guarantee.

{{< tabs >}}
{{< tab icon="cplusplus" label="C++" >}}

```cpp
// Returns a path to a temporary file using the given seed as the filename.
// Temp files are always created under /tmp.
std::string file_name(int seed) {
  return "/tmp/tempfile-" + std::to_string(seed);
}
```

### ❌ Bad Example

```cpp
// Restates the implementation; passes even if the path is wrong.
TEST_CASE("file_name returns the temp path") {
  REQUIRE(file_name(123) == "/tmp/tempfile-123");
}
```

### ✅ Good Example

```cpp
// Asserts the documented property: the path is under /tmp/.
TEST_CASE("file_name is under /tmp") {
  // Arrange
  const int seed = 123;

  // Act
  const std::string name = file_name(seed);

  // Assert
  REQUIRE_THAT(name, Catch::Matchers::StartsWith("/tmp/"));
}

// Asserts a deterministic property: distinct seeds yield distinct paths.
TEST_CASE("file_name is unique per seed") {
  // Arrange
  const int seed_a = 123;
  const int seed_b = 124;

  // Act
  const std::string name_a = file_name(seed_a);
  const std::string name_b = file_name(seed_b);

  // Assert
  REQUIRE(name_a != name_b);
}
```

{{< /tab >}}

{{< tab icon="go" label="Go" >}}

```go
package tmp

// FileName returns a path to a temporary file using the given seed as the
// filename. Temp files are always created under /tmp.
func FileName(seed int) string {
  return fmt.Sprintf("/tmp/tempfile-%d", seed)
}
```

### ❌ Bad Example

```go
package tmp_test

// Restates the implementation; passes even if the path is wrong.
func TestFileName(t *testing.T) {
  for _, seed := range []int{123, 456} {
    t.Run(fmt.Sprintf("seed=%d", seed), func(t *testing.T) {
      got := tmp.FileName(seed)

      if want := fmt.Sprintf("/tmp/tempfile-%d", seed); got != want {
        t.Errorf("FileName(%d) = %q, want %q", seed, got, want)
      }
    })
  }
}
```

### ✅ Good Example

```go
package tmp_test

// Asserts the documented property: the path is under /tmp/.
func TestFileNameIsUnderTmp(t *testing.T) {
  // Arrange
  for _, seed := range []int{123, 456} {
    t.Run(fmt.Sprintf("seed=%d", seed), func(t *testing.T) {
      // Act
      got := tmp.FileName(seed)

      // Assert
      if !strings.HasPrefix(got, "/tmp/") {
        t.Errorf("FileName(%d) = %q, want prefix %q", seed, got, "/tmp/")
      }
    })
  }
}

// Asserts a deterministic property: distinct seeds yield distinct paths.
func TestFileNameIsUniquePerSeed(t *testing.T) {
  // Arrange
  const seedA, seedB = 123, 124

  // Act
  a, b := tmp.FileName(seedA), tmp.FileName(seedB)

  // Assert
  if a == b {
    t.Errorf("FileName(%d) and FileName(%d) both = %q, want different", seedA, seedB, a)
  }
}
```

{{< /tab >}}

{{< tab icon="python" label="Python" >}}

```python
def file_name(seed: int) -> str:
  """Return a path to a temporary file using the given seed as the filename.

  Temp files are always created under /tmp.
  """
  return f"/tmp/tempfile-{seed}"
```

### ❌ Bad Example

```python
# Restates the implementation; passes even if the path is wrong.
def test_file_name():
  assert file_name(123) == "/tmp/tempfile-123"
```

### ✅ Good Example

```python
# Asserts the documented property: the path is under /tmp/.
def test_file_name_is_under_tmp():
  # Arrange
  seed = 123

  # Act
  name = file_name(seed)

  # Assert
  assert name.startswith("/tmp/")

# Asserts a deterministic property: distinct seeds yield distinct paths.
def test_file_name_is_unique_per_seed():
  # Arrange
  seed_a, seed_b = 123, 124

  # Act
  name_a, name_b = file_name(seed_a), file_name(seed_b)

  # Assert
  assert name_a != name_b
```

{{< /tab >}}

{{< tab icon="rust" label="Rust" >}}

```rust
/// Returns a path to a temporary file using the given seed as the filename.
/// Temp files are always created under /tmp.
pub fn file_name(seed: i32) -> String {
  format!("/tmp/tempfile-{seed}")
}
```

### ❌ Bad Example

```rust
// Restates the implementation; passes even if the path is wrong.
#[test]
fn file_name_returns_the_temp_path() {
  assert_eq!(file_name(123), "/tmp/tempfile-123");
}
```

### ✅ Good Example

```rust
// Asserts the documented property: the path is under /tmp/.
#[test]
fn file_name_is_under_tmp() {
  // Arrange
  let seed = 123;

  // Act
  let name = file_name(seed);

  // Assert
  assert!(name.starts_with("/tmp/"));
}

// Asserts a deterministic property: distinct seeds yield distinct paths.
#[test]
fn file_name_is_unique_per_seed() {
  // Arrange
  let (seed_a, seed_b) = (123, 124);

  // Act
  let (name_a, name_b) = (file_name(seed_a), file_name(seed_b));

  // Assert
  assert_ne!(name_a, name_b);
}
```

{{< /tab >}}
{{< /tabs >}}

## Resources

* **[Testing on the Toilet: Test Behavior, Not Implementation][gtb]** - Google
  Testing Blog on writing tests against behavior rather than implementation
  details.

* **[Change-Detector Tests Considered Harmful][change-detector]** - Google
  Testing Blog, the origin of the *change-detector test* term.

* **[X-Unit Test Patterns][x-unit] by Gerard Meszaros** - describes the
  *Sensitive Equality* and *Overspecified Software* smells that result from
  coupling tests to implementation.

[gtb]: <https://testing.googleblog.com/2013/08/testing-on-toilet-test-behavior-not.html>
[change-detector]: <https://testing.googleblog.com/2015/01/testing-on-toilet-change-detector-tests.html>
[x-unit]: <https://martinfowler.com/books/meszaros.html>
