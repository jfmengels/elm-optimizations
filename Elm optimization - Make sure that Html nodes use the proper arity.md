---
title: Make sure that Html nodes use the proper arity
updated: 2022-04-13 14:26:39Z
created: 2022-04-13 14:25:45Z
tags:
  - elm-performance
---
[[Elm optimization techniques]]

Some Html functions (div, input, ...) don't use the right arity, meaning they will ALWAYS fail the AX checks.

Changing their definition to respect the arity better should make for a nice performance improvement (as that would reduce the number of function calls).