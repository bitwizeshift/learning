+++
title = 'Idempotent'
date = 2026-06-26
slug = 'idempotent'
resources = ['glossary']
description = 'An operation that produces the same result whether it is applied once or many times.'
toc = false
+++

An operation is **idempotent** when applying it multiple times has the same
effect as applying it once. Setting a value to `5` is idempotent; incrementing a
value by `1` is not. The first call may change state, but every subsequent call
with the same input leaves the state unchanged.

<!--more-->

Idempotency matters for retries and at-least-once delivery: if a request can be
safely repeated, a client can retry on timeout without risking duplicate effects.
