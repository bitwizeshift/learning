+++
title = 'Prefer Table-Driven Tests'
date = 2026-05-12
id = 'GO-001'
slug = 'GO-001'
url = '/best-practice/go/go-001'
aliases = [
  '/go-001',
]
resources = ['best-practice']
toolchains = ['go']
concepts = ['testing']
[focus]
toolchains = ['go']
+++

Express related test-cases as data to keep tests readable and exhaustive, and
compare the results using full-structure comparison.

Table-driven tests express each case as a row of data, keeping intent obvious and
making it cheap to add cases.

<!--more-->

## Why this matters

Table-driven tests improve code maintainability and test coverage by separating
test logic from test data. This approach makes it trivial to add new test
cases; simply add a row to the data structure without duplicating test logic.
It also enhances readability by presenting all test scenarios in a clear,
tabular format, making it easier to spot missing cases or inconsistencies.

Additionally, table-driven tests scale better as your test suite grows, reduce
oilerplate code, and make failures more descriptive by automatically including
the test case name in output.

## Example

```go
func TestAbs(t *testing.T) {
  t.Parallel()

  testCases := []struct {
    name string
    in   int
    want int
  }{
    {
      name: "Positive",
      in:   3,
      want: 3,
    }, {
      name: "Negative",
      in:   -3,
      want: 3,
    }, {
      name: "Zero",
      in: 0,
      want: 0,
    },
  }
  for _, tc := range testCases {
    t.Run(tc.name, func(t *testing.T) {
      // Act
      v := math.Abs(tc.in)

      // Assert
      if got, want := v, tc.want; got != want {
        t.Errorf("Abs(%d) = %d, want %d", tc.in, got, want)
      }
    })
  }
}
```
