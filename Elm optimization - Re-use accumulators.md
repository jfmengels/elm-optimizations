[[Elm optimization techniques]]

Instead of `List.map f x ++ y`, we could use `y` as the initial accumulator.

```elm
List.map f list ++ right
-->
List.foldr (\x acc -> cons f x :: acc) [] list ++ right
-->
List.foldr (\x acc -> cons f x :: acc) right list
```

This skips the entire `++` step and avoids creating an intermediate list before the 

A simple solution would be to create a copy of `List.map` that takes an accumulator. This can be done in user-land, at least for user-land functions, but bloats the build and has to be done manually.

This idea probably works for other types or other operations, but I haven't thought about it much.

## Applying this automatically

(this is pure spit-balling)

It would be nice if we could somehow define an initial accumulator conditionally.

```js
var $elm$core$List$map = F2(
 function (f, xs, initial = _List_Nil) {
  return A3(
   $elm$core$List$foldr,
   F2(function (x, acc) { /* ... */ }),
   initial,
   xs);
 });
```

and then have the compiler inject the initial accumulator from the surrounding code.

I'm thinking of default parameters but they really don't fit well with currying.
Also, default parameters were introduced in ES2015, so it's not compatible with Elm's ES5 target.

## Using TRMC

In `List.map f list ++ right`, we basically wish to change the end of the left-hand list to point to `right`. But the end of a list is hard to get.

However with [[Elm optimization - Tail Recursion Modulo Cons]], we do have access to the end of the list, but we don't return it.

We could swap to another implementation of TRMC'd `List.map` where we return both.