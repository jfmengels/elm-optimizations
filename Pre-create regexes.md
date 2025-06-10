---
title: Pre-create regexes
updated: 2021-12-05 18:12:20Z
created: 2021-12-05 18:11:17Z
tags:
  - elm-performance
---
[[Elm optimization techniques]]

`Regex`es with string literals could be validated at compile time, and potentially turned into compiler errors.

We can then
- remove the accompanying `Maybe.withDefault`
- Use the JavaScript regex notation (if that's faster or shorter, which I believe it is)
- Apply known regex optimizations. There are probably some knowledge to be found in [eslint-plugin-regexp](https://www.npmjs.com/package/eslint-plugin-regexp) and other repositories.