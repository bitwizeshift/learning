+++
title = 'Flaky Test'
date = 2026-06-26
slug = 'flaky-test'
aliases = ['/glossary/flaky-test']
resources = ['glossary']
tags = ['testing', 'unit-testing']
description = 'A test that passes or fails non-deterministically without any change to the code under test.'
toc = false
+++

A **flaky test** is one that yields different results — sometimes passing,
sometimes failing — without any change to the code it exercises. Flakiness
erodes trust in the suite: once a failure might be "just flaky", real
regressions get ignored.

<!--more-->

Common causes include reliance on wall-clock time, ordering assumptions,
shared state between tests, network or filesystem access, and unsynchronized
concurrency. The fix is almost always to remove the hidden source of
non-determinism rather than to retry the test until it passes.
