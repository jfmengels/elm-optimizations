[[Elm optimization techniques]]

Gren uses JavaScript [Array](https://developer.mozilla.org/fr/docs/Web/JavaScript/Reference/Global_Objects/Array) under the hood, which is way faster than Elm's custom `Array` type.

## Resources

[De-throning the List: Summary - DEV Community](https://dev.to/robinheghan/de-throning-the-list-summary-3f3c)
[Array improvements by robinheghan · Pull Request #1029 · elm/core](https://github.com/elm/core/pull/1029/files#diff-3c7d56e03822b8cadc7b54640d53b0ee33955b6fc1fee07d71ed6601aa09cbd3)

Gren's Array implementation
- [core/src/Array.gren at main · gren-lang/core · GitHub](https://github.com/gren-lang/core/blob/main/src/Array.gren)
- [core/src/Gren/Kernel/Array.js at main · gren-lang/core · GitHub](https://github.com/gren-lang/core/blob/main/src/Gren/Kernel/Array.js)
- [core/src/Array/Builder.gren at main · gren-lang/core · GitHub](https://github.com/gren-lang/core/blob/main/src/Array/Builder.gren)

## Benefits

A lot of operations are therefore faster.

There's no cost at declaring `List` or `Array` like in Elm:

```elm
Array.fromList [ 1, 2, 3 ]
```
compiles to
```js
$elm$core$Array$fromList( // iterates over the Elm List
	_List_fromArray( // iterates over the JS Array
		[1, 2, 3]
	)
);
```