---
title: Elm-performance improvements
updated: 2023-01-04 22:57:41Z
created: 2021-11-12 09:51:19Z
tags:
  - elm-performance
---
[[Elm optimization techniques]]

-  Concatenating strings ("a" + "b" -> "ab")
- Concatenating JS arrays before being turned into lists
- Computing operations on numbers (60 * 60 for instance)
