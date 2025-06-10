---
title: String performance improvements
updated: 2022-01-07 21:56:20Z
created: 2022-01-07 21:51:26Z
tags:
  - elm-performance
---
[[Elm optimization techniques]]

Chars are strings, so conversion functions should be replaced by an `identity` function.
- [`_Utils_chr`](https://github.com/elm/core/blob/master/src/Elm/Kernel/Utils.js#L145-L146) (not sure why there's a `new String(c)` in debug mode)
- [String.fromChar](https://github.com/elm/core/blob/master/src/String.elm#L533-L535)