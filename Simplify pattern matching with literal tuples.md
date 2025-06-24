[[Elm optimization techniques]]

```elm
case ( a, b ) of
   ( A, B ) -> 1
   ( B, A ) -> 1
   _ -> ...
```

If there is a name given to the value, only then create a tuple:

```elm
case ( a, b ) of
   ( A, B ) -> ...
   (( B, A ) as tuple) ->
      -- In JS, here we would add `var tuple = _Utils_Tuple2(a, b);`
      tuple
   tuple ->
      -- In JS, here we would add `var tuple = _Utils_Tuple2(a, b);`
      tuple
```

## Example

```elm
case ( a, b ) of
    ( Just r, y ) ->
        r

    ( Nothing, Nothing ) ->
        0

    tuple ->
		Tuple.first tuple |> Maybe.withDefault 0
```

Currently compiles down to (without `--optimize`, but it only changes how the conditions are evaluated):
```js
var _v9 = _Utils_Tuple2(a, b);
if (_v9.a.$ === 'Just') {
	var r = _v9.a.a;
	var y = _v9.b;
	return r;
} else {
	if (_v9.b.$ === 'Nothing') {
		var _v10 = _v9.a;
		var _v11 = _v9.b;
		return 0;
	} else {
		var tuple = _v9;
		return A2($elm$core$Maybe$withDefault, 0, tuple.a);
	}
}
```

Proposal:
```js
// Create "var _v1 = f(a)" declarations if we can't simply refer to `a`. Same for `b`.
if (a.$ === 'Just') {
	var r = a.a;
	var y = b; // Could be removed
	return r;
} else {
	if (b.$ === 'Nothing') {
		return 0;
	} else {
		var tuple = _Utils_Tuple2(a, b);
		return A2($elm$core$Maybe$withDefault, 0, tuple.a);
	}
}
```

It may be interesting to look into whether we should do the same thing for `case SomeWrapper data of`. The use of tuples in pattern matching is the 