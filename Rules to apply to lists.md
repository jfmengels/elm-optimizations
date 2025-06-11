---
title: Rules to apply to lists
updated: 2022-01-08 23:27:12Z
created: 2021-11-20 17:58:22Z
tags:
  - elm-performance
---
[[Elm optimization techniques]] [[Blog drafts]]
Uncategorized:

```elm
{- _List_fromArray -}[ 1, 2 ] ++ list -- or using List.append
```
We can probably avoid creating the list AND then calling `_List_fromArray`, instead we could use (a new function) `_List_appendToLiteral([1, 2], list)` which creates a list for `[1, 2]` and sets the end to be `list`

```js
function _List_appendToLiteral(jsArray, elmList)
{
	var out = elmList;
	for (var i = arr.length; i--; )
	{
		out = _List_Cons(arr[i], out);
	}
	return out;
}
```
(in practice, this is `_List_fromArray` but with the initial value for `out` that comes from an argument). This seems like a common operation so this could be quite useful.

---

I made quite a few benchmark explorations recently, and here is what I came up.


### Merge all List.map together, merge all List.filter together, etc.

```elm
[ 1, 2, 3 ]
	|> List.map (fn1 >> fn2)
	|> List.filter (fn3 >> fn4)
	|> List.map fn5
```

This shows [nice performance improvements](https://github.com/jfmengels/elm-benchmarks/blob/main/src/WhatIsFaster/ListFusion.elm) (but I somehow deleted the benchmark?)

As shown in a different benchmark, function composition [is pretty slow](https://github.com/jfmengels/elm-benchmarks/blob/main/src/WhatIsFaster/FunctionComposition.elm), so the transformation can be made even more performant.
(Btw, in almost all benchmarks in [`jfmengels/elm-benchmarks`](https://github.com/jfmengels/elm-benchmarks) I provide a screenshot of benchmark results in a file next to it, please check them out)

This optimization, which I've heard named "stream fusion" could be a proposal on its own (but this is just an initial exploration draft). Among other questions, we could ponder which functions could/should support this, whether it makes sense to enable this for custom functions and data structures (or only for core elements), and if so how developers could let the compiler/optimizer know about this. You will notice the same questions apply to pretty much for every proposal here.

Then we can have multiple optimizations, which I think are all pretty interesting, some extremely so.

### Using native JS operations

Every time a list gets created through a List literal (e.g. `[ 1, 2, 3 ]`), it gets compiled to a JavaScript Array (`[ 1, 2, 3 ]` again) wrapped in a `_List_fromArray`.

For instance, the following Elm code

```elm
someValue =
	[ 1, 2, 3 ]
		|> List.map fn1
		|> List.map fn2
```

gets compiled to the following

```js
var $author$project$Api$someValue = A2(
	$elm$core$List$map,
	fn2,
	A2(
		$elm$core$List$map,
		fn1,
		_List_fromArray(
			['1', '2'])));
```

But JavaScript Arrays are a lot faster to run functions like `map` on than `List.map` is. So an improvement we could do is, whenever we detect `$elm$core$List$map` being applied to anything wrapped in `_List_fromArray`, we instead apply a native JavaScript`map` function, and wrap the result in `_List_fromArray`.


```js
var $author$project$Api$someValue = A2(
	$elm$core$List$map,
	fn2,
	_List_fromArray(
		['1', '2'].map(fn1)));
```

This is a lot faster already (More than 2x on list of 10 elements, more than 4x on 1000 elements), but we can do better, because we can actually use a mutating map function. Since `_List_fromArray` works on JavaScript values, that are never read in Elm-land (nor in lazy at the moment), we can be sure that mutating its argument never causes a side-effect, and therefore it's safe to mutate (I'm pretty sure, but would love for someone to triple-check that, considering the impact).

```js
function _mutatingJsArrayMap(mapper, arr) {
  var len = arr.length;
  for (var i = 0; i < len; i++) {
      arr[i] = mapper(arr[i]);
  }
  return arr;
}

var $author$project$Api$someValue = A2(
	$elm$core$List$map,
	fn2,
	_List_fromArray(
		_mutatingJsArrayMap(fn1, ['1', '2'])));
```

[I benchmarked this](https://github.com/jfmengels/elm-benchmarks/blob/main/src/NativeJsArrayExploration/OpportunisticJsArrayLoop.elm) and the results are pretty wild.

![](https://github.com/jfmengels/elm-benchmarks/blob/main/src/NativeJsArrayExploration/OpportunisticJsArrayLoop-Results-Chrome.png)

The cost of this is an additional function to add to the bundle (for every function that we want to make use of this, and for which we found a need to use it).

And this improvement bubbles up nicely, so the example above can evolve simply to use `_mutatingJsArrayMap` twice (or other non-stream-fusion-able function) before we finally convert it to a list.

```js
var $author$project$Api$someValue =
	_List_fromArray(
		_mutatingJsArrayMap(
				fn2,
				_mutatingJsArrayMap(
					fn1,
					['1', '2'])));
```

I already have a branch in my fork of `elm-optimize-level-2` that applies this optimization on several functions ([implementation](https://github.com/jfmengels/elm-optimize-more/blob/native-list/src/transforms/nativeListTransformer.ts) and [tests/examples](https://github.com/jfmengels/elm-optimize-more/blob/28bd346dc2dedf02a83075d87618048920797ba8/test/useNativeListTransformer.test.ts)).

### Skip extra List_JsFromArray

The result of our implementation brought us the following steps
1. Creation of a JavaScript Array literal
2. Call of a mutating map
3. Call of a mutating map
4. Call of  `_List_fromArray`

(Let's imagine the mutating map in step 2 is not combinable with step 3 as we've seen in the stream fusion proposal. Sorry if it's distracting.)

Here is the implementation of the last function:

```js
function _List_fromArray(arr) {
	var out = _List_Nil;
	for (var i = arr.length; i--; )
	{
		out = _List_Cons(arr[i], out);
	}
	return out;
}
```

What we notice, is that we loop an additional time over the array just to create a list. Instead, we could have dedicated functions that do whatever they need but store their results in a list directly, basically merging their aim (mapping values) with creating a proper list.


```js
function _List_nativeMapAndFromArray(mapper, arr) {
    var out = _List_Nil;
    for (var i = arr.length; i--; )
    {
        out = _List_Cons(mapper(arr[i]), out);
    }
    return out;
}

var $author$project$Api$someValue =
	_List_nativeMapAndFromArray(
			fn2,
			_mutatingJsArrayMap(
				fn1,
				['1', '2']));
```

Here is a [benchmark on a simple List.map call](https://github.com/jfmengels/elm-benchmarks/blob/main/src/NativeJsArrayExploration/NativeMapAndFromArray.elm):

![](https://github.com/jfmengels/elm-benchmarks/blob/main/src/NativeJsArrayExploration/NativeMapAndFromArray-Results-Chrome.png)

### Skip List_JsFromArray entirely

We can apply this technique on other things. For instance, `Set.fromList [1, 2, 3]` compiles to the following code:

```elm
$elm$core$Set$fromList(
	_List_fromArray(
		[1, 2, 3]));
```

Another `_List_fromArray`. Anytime you need to create a `Set`, you take the JavaScript Array, fold over it to turn it into a `List`, then loop over it again to turn it into a `Set`.

We can make this faster by creating a [dedicated function](https://github.com/jfmengels/elm-benchmarks/blob/main/src/NativeJsArrayExploration/SetFromLiteral.elm) function [(~10%-30% faster)](https://github.com/jfmengels/elm-benchmarks/blob/main/src/NativeJsArrayExploration/SetFromLiteral-Results-Chrome.png).

And then we can do the same thing for [`Dict`](https://github.com/jfmengels/elm-benchmarks/blob/main/src/NativeJsArrayExploration/DictFromLiteral.png), and `Array`, and more?

### Making use of native functions in other places

This is all cool and all, but if the only place where we can apply this on list literals, it will not be applied very often. Thankfully, there are some other places.

There are a few functions that call `_List_toArray` and/or `_List_fromArray` internally, such as `_List_sortBy` and `_List_sortWith`, and String functions like `_String_fromList`. Let's take `_List_sortBy` (which `$elm$core$List$sortBy` aliases to) for instance.

```js
var _List_sortBy = F2(function(f, xs)
{
	return _List_fromArray(_List_toArray(xs).sort(function(a, b) {
		return _Utils_cmp(f(a), f(b));
	}));
});
```


```elm
someList
  |> List.sortBy fn1
  |> List.map fn2
```

which translates to the following JS code:

```js
A2(
	$elm$core$List$map,
	fn2,
	A2(
		$elm$core$List$sortBy,
		fn1,
		_List_fromArray(
			someList)));
```


Here, with a tiny bit of inlining of `$elm$core$List$sortBy`, we would get the following code:


```js
var f = fn1

A2(
	$elm$core$List$map,
	fn2,
	_List_fromArray(_List_toArray(someList).sort(function(a, b) {
		return _Utils_cmp(f(a), f(b));
	})));
```

And through this inlining, we have, once again, a `List.map` being applied on data wrapped in `_List_fromArray`, meaning we can apply the optimizations around using the native functions.

A different exploration could be to check whether it makes sense, for longer and more complex chains of list-related functions, to pre-emptively switch to JavaScript arrays before applying faster native functions.

We could try that out with a function that aliases to `identity` but would be compiled to `_List_toArray` (and then back somewhere).

```elm
list
	|> List.useNativeJsArrayPrettyPlease
	|> List.map fn
```


### Skipping to- and from- array

Another benefit of this inlining here would be that we could potentially skip `_List_fromArray` and `_List_toArray`

```elm
[1, 2, 3]
  |> List.map fn1
  |> List.sortBy fn2
```

that would be transformed, using the previous optimization of using a native map on an array literal, to

```js
A2(
	$elm$core$List$sortBy,
	fn2,
	_List_fromArray(
		_mutatingJsArrayMap(fn1, [1, 2, 3])));
```

If we inline `$elm$core$List$sortBy` we would get

```js
var f = fn2;

_List_fromArray(_List_toArray(
	// The mutating map call
	_List_fromArray(
		_mutatingJsArrayMap(fn1, [1, 2, 3]))
).sort(function(a, b) {
	return _Utils_cmp(f(a), f(b));
}));
```


And what we notice here is that we have a `_List_fromArray` followed by a `_List_toArray`. They cancel each other out, meaning we can remove these entirely, saving two iterations of the list.

```js
var f = fn2;

_List_fromArray(
	_mutatingJsArrayMap(fn1, [1, 2, 3])
		.sort(function(a, b) {
			return _Utils_cmp(f(a), f(b));
		}));
```


### Opportunistic mutations on List

I previously mentioned that we can use mutating functions on the JS arrays. Well, in some cases, we could also do that on Elm Lists. For instance, since the core `List.map` functions creates a new list with a new reference, mutating this one will have no noticeable impact (if it's not referenced elsewhere). So if you have `list |> List.map fn |> List.filter`, you can use a mutating version of `List.filter` that works on Elm `List`.

And these versions are so much faster! I made benchmarks for a mutating [`List.filter`](https://github.com/jfmengels/elm-benchmarks/blob/main/src/MutationExploration/ListFilter.elm), [`List.take`](https://github.com/jfmengels/elm-benchmarks/blob/main/src/MutationExploration/ListTake.elm) and [`List.map`](https://github.com/jfmengels/elm-benchmarks/blob/main/src/MutationExploration/ListMap.elm), but this applies to I think most List functions, and likely even more.

![](https://github.com/jfmengels/elm-benchmarks/blob/main/src/MutationExploration/ListFilter-Results-Chrome.png)
![](https://github.com/jfmengels/elm-benchmarks/blob/main/src/MutationExploration/ListTake-Results-Chrome.png)
![](https://github.com/jfmengels/elm-benchmarks/blob/main/src/MutationExploration/ListMap-Results-Chrome.png)

Thanks to Roc for inspiring this idea! (yay, mutual inspiration!)

A first step to make this work, is to write down a list of functions that we know create new references, and create a bunch of faster mutating functions to be applied when encountering the results of the functions that create new references or the mutating functions themselves.


### Non-reversing operations

There are several reasons why the opportunistic mutation from before is so fast. First of all, we skip creating plenty of new records for the list elements. Second of all, we iterate the list twice to keep the order right. Or we kind of do.

```elm
simpleListMap fn list =
	List.foldl (\n acc -> fn a :: acc) [] list
```

If you run `simpleListMap identity [ 1, 2, 3 ]`, you will receive `[ 3, 2, 1 ]`. Because the first element you encounter while iterating is the first one you add to the (end of the) resulting list, the result will be in inversed order. You will have the same issue with most approaches using recursive functions as well.
A naive simple solution is to apply a `List.reverse` on the result, which adds a second iteration through the list.

A more performant solution is to use `List.foldr` which iterates in the reverse order, which is faster than the additional reverse. But [looking at the implementation](https://github.com/elm/core/blob/master/src/List.elm#L172-L206), you instinctively know that it does more work and will likely be slower than `foldl` (and you'd be right).

If we notice that we are chaining multiple operations that we can't combine, but that both attempt to keep a list in order, then there is some duplicate overhead.

```elm
list
	|> List.map fn1 -- Uses the slower foldr to keep the list in order
	|> List.filter fn2 -- Uses the slower foldr to keep the list in order
```

Instead, we could use alternatives that use `foldl`. If we use them twice, then there is no problem regarding the order.

```elm
list
  |> _reversingListMap fn1 -- After this the list is in the inverse order...
  |> _reversingListFilter fn2 -- After this the list is back in the right order!
```

Which would remove some overhead.

We could also apply this optimization when their result is passed to a function where the order of the list doesn't impact the behavior, such as `Set.fromList` or `List.sum`.

```elm
list
	|> List.map fn
	|> Set.fromList
-- will have the same behavior than the faster
list
	|> _reversingListMap fn
	|> Set.fromList
```


### Using combined operations

Let's go back to stream fusion. That idea is about merging two of the same operations into one, when doing so would achieve the same results. The gain from that is mostly the cost of iterating over of the list multiple times and creating a list that will be discarded.

When you care a lot about the performance in part of your codebase ([note that you rarely will in Elm](https://youtu.be/mmiNobpx7eI?t=1091)), then what you might do, as I did in `elm-review`, is to combine multiple functions into one.

For instance, if we think that "map then filter" is a common operation (i.e. being executed a lot, not necessarily in plenty of different parts of a codebase), then we can combine them into a more efficient operation:

```elm
[ 1, 2, 3 ]
	|> List.map (fn1 >> fn2)
	|> List.filter (fn3 >> fn4)
	|> List.map fn5

-->

[ 1, 2, 3 ]
	|> _List_mapThenFilter (fn1 >> fn2) (fn3 >> fn4)
	|> List.map fn5

_List_mapThenFilter mapper predicate list =
	List.foldr (\value acc ->
		let
			mappedValue = mapper value
    in
	  if predicate mappedValue then
	    mappedValue :: acc
		else
			acc
	)
	[]
	list
```

This improves the performance, very likely to similar levels than stream fusion (no benchmarks for this one though, sorry).

We could have this function applied for plenty of other combinations, even a simple "filter then map" would work, or "map then filter then map", or
["indexMap Tuple.pair + List.filterMap"](https://github.com/jfmengels/elm-benchmarks/blob/main/src/WhatIsFaster/IndexMap.elm) (this one I have a benchmark for!).

```elm
_List_filterThenMap predicate mapper list =
	List.foldr (\value acc ->
	  if predicate value then
	    mapper value :: acc
		else
			acc
	)
	[]
	list
```

I haven't researched this one much (so proposal is a bit handwavy), but the compiler could try to merge common functions to achieve this result. I believe it will have a very hard time figuring out the execution hotspots and will therefore likely do a bad job at it. So either it will do so automatically (increasing the bundle size for rare performance gains) or it's up to the developer to tell the compiler how and where to do so.


### Downsides

These improvements are most efficient when you apply on list literals and/or when you have longer chains of list functions. This means that if you have such a function which might be a bit complex and you then break it up, we will likely end up using non-optimized versions, meaning the more readable code will end up performing worse, which is not a good incentive.

We could remediate that by a mix of
- Inlining code: restoring the nice chain of operations even when they are cut into several functions
- Making the optimizer smarter, such as having it notice that function X is only called in places where opportunistic mutation would apply and therefore applying the mutation.

### Pushing towards the edge

In practice, and if we push things very very far, I think that you can use opportunistic mutation and or native functions for a lot of the code.

Opportunistic mutation could be used for any value that is not referenced more than once, meaning most values except top-level values and what comes from the `Model` and things that (in)directly reference multiple times.

As for native functions, you can likely keep a lot of the values as arrays until they need to actually be used.

```elm
someConstant =
		List.map increment [ 1, 2, 3 ] -- This could stay as a native JS array

view model =
	someConstant
		|> List.map fn1 -- This could use a native but non-mutating JS array function (non-mutating because we reference a constant)
		-- Somewhere we would need to translate this to a native List probably, but we could have that be as late as possible.
		|> div []
```

In this case, if we make a version of `div` that accepts JS Arrays, then we don't even need to transform this to a List.
[[Elm optimization - Support JavaScript Array for DOM functions]]

### ???

- Is the data a native JS Array or an Elm List?
- Is the data mutable (no other reference, mutation has no noticeable impact) or immutable (potential other references, mutation may change behavior)?

- Can the operations be merged into one?
- Can we use non-order preserving functions?

Would it make sense in some cases to transform a list back into a JS array?

`List.sortBy` for instance turns the list into a JS array, sorts it then turns it into a List again. Here we could take the opportunity to not re-wrap out of the box and re-wrap as late as possible.

How much does this conflict with improvements like changing `_List_JsFromArray([1])` to `{ $: 1, a: 1, b: _List_Nil }`?

What basic primitive can we change these functions to? Recursive functions could work but would a lot of code (and increase asset size). Changing to a foldl/foldr could work but
from testing, foldl is slower than recursion, though maybe because of the A/F wrapping (could we get rid of that for these?)

How do we support adding more optimizations? How do we support custom optimizations that are safe? Haskell-like `RULES`?