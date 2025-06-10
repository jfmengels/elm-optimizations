[[Elm improvements]]
[[Elm optimization techniques]]

Elm has never supported CPS when used with tail-recursion even though CPS is kind of meant to be used with TCO.
- [Closures in recursive calls get shadowed, causing stack overflows · Issue #2017](https://github.com/elm/compiler/issues/2017)
- [Tail Call Optimization produces bad closures · Issue #2268](https://github.com/elm/compiler/issues/2268)

(A quick fix for this is to use `let` instead of `var` for declarations, but that is not part of ES5)

Supporting this could potentially lead to better-performing APIs, and/or novel API approaches.