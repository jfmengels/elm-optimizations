[[Elm optimization techniques]]

**NOTE:** This is mostly thoughts written down, that I haven't fact-checked/cross-checked nor polished.

Related Discord thread based on a shorter draft: https://discord.com/channels/534524278847045633/731183487775932478/1273382974733156366

---

I am wondering about adding support for values evaluated at compile time. I think it it technically very feasible and would work very well.

Zig has this concept of `comptime`: https://ziglang.org/documentation/master/#comptime.
I'm not sure how relevant Zig's comptime is here actually, but the idea exists elsewhere at least.

Since we only deal with pure functions, we could pre-compute pretty much everything that is not a function itself. Since that is quite a lot, we should probably do this on demand, through some annotation such as `{- @comptime -}` (up to bikeshedding).

There are at least two benefits : Things that take a lot of time to compute could be done once at compilation time (with `--optimize`) instead of on the user's device, improving their load time.

The other benefit is to help keeping values - that you don't want to spend CPU time to compute at runtime - in sync:

```elm
thing = "abc"

{-| @comptime -}
thingLength = String.length thing
```
which would compile to the following JavaScript code:
```js
var thing = "abc";
var thingLength = 3;
```

We could apply this to inline expressions as well, as long as we make sure the expression does not reference anything dynamic (parameters or values extracted out of a pattern-match). `elm-format` does not remove the parentheses so this works (although it could move them...). Example:

```elm
fn str = str ++ ({- @comptime -} String.repeat 2 "foo")
```
which would compile to the following JavaScript code:
```js
function fn(str) {
  return str + "foofoo";
}
```
or maybe even extracted to a top-level variable (so that if it's like a large list, we don't re-allocate it inside of the function).

This can also help remove code from the bundle. Things that were only used at comptime could be removed by the compiler and/or minifier.

Only values reached by live-code inclusion would be computed. And values referenced by the expression to be computed would not be included in the live-code inclusion (meaning potentially a large pruning of code).

I'm not sure whether comptime should be enabled in development builds. I'd probably say no, or maybe on demand.

---

> imo compile time evaluation alone isn't that useful. But would make it enormously useful is if you could do something like this: (Martin Stewart)

```elm
-- This is allowed for comptime values and will cause a compile error if the value isn't Just in this case
(Just url) = Url.fromString "https://example.com/"
```

Doing compilation time validation could be a very useful feature.

Without a dedicated syntax, this would be achievable by forcing a stack overflow in the error case, with a pretty terrible error though.

This is even already achievable by doing the same thing at runtime on startup, with an even worse experience (but catchable early by having tests reference the variable). That or you have a dedicated test.

## How to

The compiler would create a JavaScript file with only the compiled values and the things they reference (live-code inclusion once again), probably something like
```elm
-- A.elm
thing = "abc"

{-| @comptime -}
thingLength = String.length thing

fn n = n + ({- @comptime -} String.length "foo")

unreferedAndNotIncluded = 1
```

```js
// generated.js
var thing = "abc"
module.exports =
  [ $string$length(thing)
  , $string$repeat("foo")
  ].map(stringifyOrThrow)
```

then Elm would execute this file and read its values. Alternatively, this script generates a JSON file or similar, and the compiler reads that, which is probably simpler. Then, it injects these values at the right location, instead of generating the code the usual way.

Not everything is stringifiable though. For instance, functions are likely not stringifiable (`add 100`, given `add a b = a + b`). Very naively and improvable later on, we could probably use `JSON.stringify` with a `replacer` that throws when a value is or contains a function. (TODO try out https://stackoverflow.com/questions/12651977/why-cant-you-stringify-a-function-expression)

For the live-code inclusion phase, this likely means that Elm has to hold two live-code inclusion sets: one for the values referenced by `main` (except code annotated by `comptime`), and one for everything annotated by `comptime`.
## Dangers

### Bundle size increase

This could increase the bundle size a lot if the compiled code ends up being very large, for instance when computing large lists. Some users could decide it's still better for them.

We could potentially report warnings if some compiled code is very large. And maybe we could give some kind of budget in terms of number of bytes occupied by the resulting string, like `{-| @comptime 1kB -}`.


### Runtime dependencies

This requires running code when compiling, so Node.js or another JS runtime may need to be a dependency.

This can definitely slow down compilation times. Libraries could also choose to add these annotations, meaning they could impact the user's compilation speed (although only for the functions that are used).

If the evaluating environment is different than the target environment, some results could be different. For instance, numbers stored with 64 bits or 32 bits. In practice in Elm this would rather be rare. Some known cases: the error message when `Json.Decode.decodeValue` changed the error message from Node.js v18 to v20, and storing that result would lead to slightly different results.

Maybe we could lose some precision when serializing/de-serializing some numbers as well.

### Caching and record mangling

If the code in comptime hasn't changed, we could cache it (with the hash of the code to run as the cache key).

However, Elm's record name mangling becomes a problem. Record mangling changes the record field names in a way that reduces bundle size. `{ fieldName1 = 1, fieldName2 = 2 }` might get turned into `{ a = 1, z = 2 }`, or into `{ bEr = 1, y = 2 }`.

The code being evaluated at compile time should ideally not get mangled (although it could be done), but would have to be re-mangled to match the ones in non-evaluated code, meaning there would likely need to be some analysis of the JavaScript code.

To choose the most optimal record of the final bundle (with evaluated and non-evaluated code), it probably makes sense to count and store in the cached file the number of occurrences per record (I imagine the compiler doing the same thing under the hood).

### Nested records

A long List would likely be a giant nested record (`{$:1,a:x,b:{...}}`). We could try to improve the code generation by figuring out whether it's a list and compile it to `_List_fromArray([x,...])` instead). That could have a overhead cost (the same as for list constants currently, to be clear), but be a lot shorter for the bundle size.

We could do the same for other core types, although we probably couldn't for custom types, as we couldn't know what is an efficient way of constructing them, at least without additional information from the user.

We currently do something similar in the caching of `elm-review` results, for `List` and `Dict` if I remember correctly.

## Stop condition

There is also the question of where to stop in the serialization. For instance, if we evaluate the following:

```elm
x = { a = 1, b = 2 }
y = { a = 0, b = 2 }
z = { a = -1, b = 2 }
{-| @comptime -}
list = List.filter (\{a} -> a >= 0) [ x, y, z ]
```

then what should the result be? It should be the following, right?
```elm
x = { a = 1, b = 2 }
y = { a = 0, b = 2 }
[ x, y ]
```
I think so too, but if we do this naively, the code will likely be:

```elm
x = { a = 1, b = 2 }
y = { a = 0, b = 2 }
[ { a = 1, b = 2 }
, { a = 0, b = 2 }
]
```

This is because when re-serializing the results, we get plain records (or JavaScript objects) and we lose track of whether they were already assigned names.
(Another trivial example would be when sorting values at compile time).

If there are no more references in the rest of the code to `x` or `y`, they would disappear, and having it inlined like this would not be an issue. If there are still references, then we end up with duplication, which means bigger bundle sizes and a lack of memory reuse (with a potential impact on `Html.Lazy`, maybe?).

We could solve this by verifying at each step of the serialization of a value, whether it's equal to another variable in scope (and of a matching type). If it is
that, then we stop the serialization for the step and reference the variable instead. And then we force that value to be available in the generated code.

(Could we get false negatives with lazy here? Where we pass a record with the same value as another lazy but with a different memory address?)

We could even push this last optimization so that any literals that have the same value are extracted to a top-level variable. This could be a compiler optimization separate from the compile idea.

```elm
[ { a = 1, b = 2 }
, { a = 1, b = 2 }
]
-->
x = { a = 1, b = 2 }
[ x, x ]
```

But I would imagine that this could be quite slow to figure out though. This is potentially do-able without an interpreter or code runner.

This last idea would not be useful for some primitive types such as numbers or booleans, though it might be for strings?

---

What do you think? Does this seem interesting? How useful do you think this could be in practice? I think it would definitely be very interesting, but I'm unsure how often this would be used in practice.

---

Potentially put a cap on how long the computation can last for?
https://discuss.ocaml.org/t/what-is-the-consensus-on-compile-time-evaluation/8338/8

----

Reading:
- https://discuss.ocaml.org/t/what-is-the-consensus-on-compile-time-evaluation/8338
- https://oliviernicole.github.io/about_macros.html
- https://okmij.org/ftp/ML/MetaOCaml.html
- https://github.com/thierry-martinez/metapp

---

## Virgil

https://news.ycombinator.com/item?id=28121394

https://github.com/titzer/virgil

The way it solves the termination problems seems to be that is doesn’t. If your compilation does not end it is your problem.

Paper: https://escholarship.org/uc/item/13r0q4fc

---

Jonathan Blow’s programming language allows almost anything to execute during compilation as well. It’s neat. Here’s an old video where he demonstrated this: https://youtu.be/UTqZNujQOlA