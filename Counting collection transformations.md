[[Elm optimization techniques]]

It can be quite wasteful in terms of performance to do work like
```elm
list
	|> List.map f1
	|> List.map f2
	|> List.map f3
```

as this will traverse the list 3 times (~6 times if we count the underlying `List.foldr` as traversing the list twice). It is a lot more efficient to traverse the list once:
```elm
list
	|> List.map (\a -> f3(f2(f1(a))))
```

This will spend less CPU time traversing the lists and creating intermediary lists that will be thrown away, while also allocating less memory to be garbage-collected later.

While this example above is fairly easy to detect using either human eyes or a static analysis tool, it becomes a lot more complicated to figure out how many transformations are done across functions, from the input of the program to the end.

Here's where we could use dynamic analysis to more easily count things.

JavaScript transformation:

```js
var $elm$core$List$foldr = F3(
	function (fn, acc, ls) {
		return A4($elm$core$List$foldrHelper, fn, acc, 0, ls);
	}
);
// -->
var globalTransformationsId = 0;
var globalListOfTransformations = {}

var $elm$core$List$foldr = F3(
	function (fn, acc, ls) {
		var list = A4($elm$core$List$foldrHelper, fn, acc, 0, ls);
		var parents = ls.parents || [{collection: ls, fn: null, id: globalTransformationsId++}];
		var id = globalTransformationsId++;
		list.parents = parents.concat([{collection, fn: fn, id: id}]);
		globalListOfTransformations[id] = list;
		return list;
	}
);
```

With this, we should be able to register all `map`-like transformations, and at the end we can traverse it to find the collections that have received the most transformations. With some luck, we can then identify places where we can collapse some of the iterations into one.
 
This should only be used in development mode, as this will have obvious performance implications. Also, while in development mode, printing out the `globalListOfTransformations` to the console will contain more readable identification information (that gets stripped down in optimized mode).

Again, only do this if you notice a bottleneck.