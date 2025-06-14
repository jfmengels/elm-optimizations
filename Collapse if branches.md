[[Elm optimization techniques]]

**This has been explored in the `FallthroughExploration` folder of `elm-benchmarks`. The results are very mixed: Sometimes much better for small inputs but much worse for large inputs.**

This may be best solved through [[Elm optimization - Avoid redundant checks in pattern matches (like Gleam)]].

---

```elm
fn x y =
  if condition x then
    if condition y then
	    Just (x, y)
	else
		Nothing
  else
    Nothing
```

compiles to
```js
function fn(x, y) {
		return condition(x)
			? (condition(y)
			   ? $elm$core$Maybe$Just(_Utils_Tuple2(x, y))
			   : $elm$core$Maybe$Nothing)
		  : $elm$core$Maybe$Nothing;
	})
```

Because the else cases result in the same value, there is some code duplication. We could combine the conditions, and remove the else:
```js
function fn(x, y) {
		return condition(x) && condition(y)
			? $elm$core$Maybe$Just(_Utils_Tuple2(x, y))
		  : $elm$core$Maybe$Nothing;
	})
```

Same for if/else compiled code:
```js
function fn(x, y) {
	if (condition(x)) {
		if (condition(y)) {
			return $elm$core$Maybe$Just(_Utils_Tuple2(x, y));
		} else {
			return $elm$core$Maybe$Nothing;
		}
	} else {
		return $elm$core$Maybe$Nothing;
	}
}
// -->
function fn(x, y) {
	if (condition(x) && condition(y)) {
		return $elm$core$Maybe$Just(_Utils_Tuple2(x, y));
	} else {
		return $elm$core$Maybe$Nothing;
	}
}
```

When encountering if/else statements, we could also potentially use the fallthrough mechanism to achieve the same result.
```js
function fn(x, y) {
	if (condition(x)) {
		if (condition(y)) {
			return $elm$core$Maybe$Just(_Utils_Tuple2(x, y));
		}
	}
	return $elm$core$Maybe$Nothing;
}
```


If the similar cases don't align, perfectly, we could negate conditions to make them look better, so that the following code compiles to the same as one of the suggested outputs above.

```elm
fn x y =
  if condition x then
    if not (condition y) then
			Nothing
		else
	    Just (x, y)
  else
    Nothing
```


I would expect the same behavior for pattern matches, and when pattern matches and if expressions are nested inside the other.

For pattern matches, there may be a need to re-order the values extracted from the pattern to be able to combine them in one condition. Note that this is not needed if we use the fallthrough mechanism.

## Expected results

I expect the resulting bundle size to be smaller.

I also hope for improved performance.

Both of these need to be tested and benchmarked of course.

Functions like `List.map5` as implemented in pure Elm could likely be made a lot smaller.

```elm
map5 : (a -> b -> c -> d -> e -> result) -> List a -> List b -> List c -> List d -> List e -> List result
map5 f zs ys xs ws vs =
    case zs of
        z :: zs_ ->
            case ys of
                y :: ys_ ->
                    case xs of
                        x :: xs_ ->
                            case ws of
                                w :: ws_ ->
                                    case vs of
                                        v :: vs_ ->
                                            f z y x w v :: map5 f zs_ ys_ xs_ ws_ vs_

                                        [] ->
                                            []

                                [] ->
                                    []

                        [] ->
                            []

                [] ->
                    []

        [] ->
            []
```

```js
var map5 = F6(
	function (f, zs, ys, xs, ws, vs) {
		if (zs.b) {
			var z = zs.a;
			var zs_ = zs.b;
			if (ys.b) {
				var y = ys.a;
				var ys_ = ys.b;
				if (xs.b) {
					var x = xs.a;
					var xs_ = xs.b;
					if (ws.b) {
						var w = ws.a;
						var ws_ = ws.b;
						if (vs.b) {
							var v = vs.a;
							var vs_ = vs.b;
							return A2(
								$elm$core$List$cons,
								A5(f, z, y, x, w, v),
								A6(map5, f, zs_, ys_, xs_, ws_, vs_));
						} else {
							return _List_Nil;
						}
					} else {
						return _List_Nil;
					}
				} else {
					return _List_Nil;
				}
			} else {
				return _List_Nil;
			}
		} else {
			return _List_Nil;
		}
	});
```

## Problems

- Finding these optimizations could require plenty of checks, a combinatorial number of those.
- Depending on how smart the optimizer is, this could be quite a brittle optimization. For better or worse, users will likely not notice this though.

## Minifiers

It seems that minifiers (`uglifyjs` at least) sometimes alters branches with the same code to use the same body.

```js
case 0:
  return 1;
case 1:
  return 1;

// becomes

case 0:
case 1:
  return 1;
```

I don't know how much work it puts into figuring this optimization, nor whether it re-orders the branches to optimize for this possibility.

When code gets sufficiently complex, minifiers don't do this either, so they might need a bit of help.