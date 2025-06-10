---
title: 'Improved Virtual DOM diffing '
updated: 2022-08-01 07:25:18Z
created: 2022-07-31 19:52:00Z
tags:
  - elm-performance
---
[[Elm optimization techniques]]

---

We define a lot of constant Html code. Since we know that isn't ever going to change, we could assign each of these a unique id and a special node type. When the virtual dom encounters one and it already has a node of the same type and the same unique id, then it knows it doesn't have to diff the rest of the tree.

This would likely not work out of the box with non-elm/html frameworks and other UI frameworks though without having the library define that this optimization is available, probably through a macro-ish feature.

---

The MillionJS VirtualDOM allows annotating parts of the DOM that will not need diffing, or for which we can determine some things at compile time.

This could be worth exploring for Elm, but would need deeper investigation into how Million does things.

https://millionjs.org/docs/api/advanced/flags

---

Maybe we could have the compiler notice all constants of type `Html a` or `VirtualDom a`, and wrap them in a tagger.

```elm
constantThing : Html msg
constantThing =
  Html.div [...] [...]
```

becomes

```js
var constantThing = $$constant$$(
  $Html$div([...], [...])
)
```