[[Elm optimization techniques]]
Builds on top of [[Elm optimization - Direct function calls]]

1. Make sure that direct function calls with more than 9 arguments are "nice"
2. Support aliases: `a :: xs` is still in the form `A2($elm$core$List$cons, a, xs)` instead of `$elm$core$List$Cons(a, xs)` because it's defined as `var $elm$core$List$cons = _List_cons;`. I think in general we should rename aliases everywhere to the original value.
3. Use the `$` prefix for wrapped functions instead, that way the `$` don't show up in performance graphs (to be verified).