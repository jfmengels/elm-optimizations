[[Elm optimization techniques]]

Function composition can be surprisingly slow ([benchmark](https://github.com/jfmengels/elm-benchmarks/blob/main/src/WhatIsFaster/FunctionComposition.elm))

![](https://github.com/jfmengels/elm-benchmarks/blob/main/src/WhatIsFaster/FunctionComposition-Results-Chrome.png?raw=true)

There are multiple ways to improve this.

## Improve the definition of compose functions

`g << f` compiles down to `A2(compose, g, f)`, whose definition is the following:

```elm
composeL : (b -> c) -> (a -> b) -> (a -> c)
composeL g f x =
    g (f x)
```

Unfortunately this makes it a 3-arity function (wrapped in `F3`), but when we call it, it's called with `A2`, a different arity. Getting the arity wrong means in practice that all arguments will be applied one by one: `composeL(g)(f)(x)`, which is 3 function calls instead of one. That makes for bad performance.

A very simple fix that improves performance drastically is to change the definition to the following:

```elm
composeL : (b -> c) -> (a -> b) -> (a -> c)
composeL g f =
	\x ->
		g (f x)
```

That changes the function so that it's wrapped in `F2`, which matches the `A2` call.

This can then benefit from [[Elm optimization - Direct function calls]] to remove the `F2`/`A2` wrapping and just be a single function.


That said, lambda-izing will likely show benefits as it could improve the arity of function calls (e.g. `takes2Arg x >> f`) and by inlining field access (`.f >> .g` --> `function($) {return $.f.g;}`).

---

## Turning composition into a lambda

Another way to improve this is to turn composition into lambdas.

```elm
f >> g
-->
\x -> x |> f |> g
```

That works well but one needs to make sure this doesn't move the computation of some values to inside the lambda (for performance reasons):

```elm
f (veryExpensive n) >> g
-->
\x -> x |> f (veryExpensive n) |> g
```

Here, we made it so that `veryExpensive n` is computed every time the lambda is called (whereas only once ever before), which can be very costly when used in `List.map` for instance.

Therefore, we need to make sure variables are declared earlier, like:

```js
var $a = veryExpensive(n);
return function (x) { return g(f($a, x)); }
```

This option will likely yield the best performance for the specific function, but it will also bloat the bundle the most. It does make for some really nice-looking results:

```js
// f >> .g >> .h
function ($) { return $.f.g.h; }
```

I don't know if this is worth doing when the first option also seems to remove most of the overhead anyway.

I made a branch in the [elm-dev/Lamdera compiler](https://github.com/jfmengels/lamdera-compiler/tree/restore-decompose-composition) but never completed it.

[[Function decomposition work]]

## Resources

Prior discussion on the subject: https://discord.com/channels/1250584603085766677/1257232269685428228