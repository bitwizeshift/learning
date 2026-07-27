+++
title = 'FIRST Principles (Testing)'
date = 2026-07-27
slug = 'first-principles'
aliases = ['/glossary/first-principles']
resources = ['glossary']
tags = ['testing', 'unit-testing']
toc = false
+++

**FIRST** is a set of five properties that a good unit test should have, coined by
Brett Schuchert and Tim Ottinger and popularized by Robert C. Martin's *Clean
Code*. The acronym is a checklist: a test that is slow, order-dependent,
environment-sensitive, manually-checked, or written too late has given up one of
the properties that make a suite worth running.

<!--more-->

The properties are not independent goals; they describe one well-built test seen
from five angles. A test that owns its state is both **I**ndependent and
**R**epeatable; removing its I/O makes it both **F**ast and repeatable. Together
they are what let a suite run on every change and be trusted when it does.

The five properties are:

* **Fast** -- a test runs in milliseconds, so the whole suite runs in seconds and
  is run constantly. Slow tests get run less often, and a test that is not run
  catches nothing.
* **Independent** (also *Isolated*) -- tests do not depend on each other or on
  execution order; each sets up its own state and leaves nothing behind, so any
  test can run alone or in parallel.
* **Repeatable** -- the same test yields the same result on every run, in any
  environment. A test that relies on the clock, the network, or random data is a
  {{< glossary term="flaky-test" text="flaky test" >}}; a test that is a pure
  function of its inputs is {{< glossary term="idempotent" text="idempotent" >}}
  across runs.
* **Self-validating** (also *Self-verifying*) -- a test decides pass or fail on its
  own, through assertions, with no manual inspection of output or logs.
* **Timely** -- tests are written at the right time, ideally just before the
  production code they cover (as in test-driven development), so the design stays
  testable and the tests actually get written.

## References

* {{< link
    url="http://agileinaflash.blogspot.com/2009/02/first.html"
    text="F.I.R.S.T"
    icon="link"
    hover="Agile in a Flash" >}} -- the reference card by Tim Ottinger and Jeff
  Langr where the acronym was published.
* {{< link
    url="https://www.amazon.com/dp/0132350882"
    text="Clean Code: A Handbook of Agile Software Craftsmanship"
    icon="link"
    hover="Robert C. Martin" >}} -- chapter 9 presents the FIRST rules for clean
  tests.
* {{< link
    url="https://medium.com/pragmatic-programmers/unit-tests-are-first-fast-isolated-repeatable-self-verifying-and-timely-a83e8070698e"
    text="Unit Tests Are FIRST"
    icon="link"
    hover="The Pragmatic Programmers" >}} -- a walkthrough of each property with
  examples.
