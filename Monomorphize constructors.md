[[Elm optimization techniques]]

[What's up with monomorphism?](https://mrale.ph/blog/2015/01/11/whats-up-with-monomorphism.html)

JavaScript performs better when the data sent to a function always has the same shape.

```js
function fn(data) {
  return data.n + 1;
}

fn({ n: 1 })        // Shape: { n }
fn({ n: 1, other }) // Shape: { n, other }
```

As far as I understand, shapes for objects (at least in the case for Elm) is only about which field names are present, and not about the value those fields they contain.
The order of the fields is also very important, but Elm makes sure those always stay in the same order ([[Elm record fields order]]).

Here, you have two different shapes, which is something that the JIT does not like.
If the JIT only sees the first shape, it's going to optimize for that shape.
Then, when another shape comes in, it will de-optimize the function, which can drastically worsen the performance of the function.

## Pad constructors

Custom types often resemble this in Elm:

```elm
type Maybe a
  = Just a
  | Nothing
```

Here, `Just` and `Maybe` will compile down to the following:

```js
var $elm$core$Maybe$Just = function (a) {
	return {$: 0, a: a};
};
var $elm$core$Maybe$Nothing = {$: 1};
```

These are 2 different shapes:
- Just: `{ $, a }`
- Nothing: `{ $ }`

which means that a function that will take a `Maybe` (and at some point gets both versions) will not be able to be as optimized as we would like it to be.

A simple solution is to pad constructors: Count the number of arguments of each constructor, and add empty fields to every constructor until they all have the same number of arguments.

Effectively, we would be changing the type as if it was like this:

```elm
type Maybe a
  = Just a
  | Nothing ()
```

Here, `Just` and `Maybe` will compile down to the following:

```js
var $elm$core$Maybe$Just = function (a) {
	return {$: 0, a: a};
};
var $elm$core$Maybe$Nothing = {$: 1, a: null};
```

## Cons

This would slightly increase the bundle size but it should be very negligible.

It would also make it slightly slower to iterate through the fields in Elm's `==`. Though this would not a problem for constants (like `Nothing`) since the reference check would yield `True` earlier.