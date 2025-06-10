[[Elm optimization techniques]]

This would remove overhead over some basic operations.
Sometimes the Elm compiler optimizes some operations

## Comparison

Comparisons are currently compiled in 2 different ways:
1. Comparison with a literal

```elm
x =
  a == "str"
```

```js
var x = a === 'str'
```

2. Comparison with non-literal

```elm
str = "str"
x = a == str
```

```js
var str = 'str'
var x = _Utils_eqHelp(a, str);
```

The suggestion would be to use version 1 whenever the type of the value is known to be comparable with `===`.

Sometimes this is forced by developers by doing `a == str ++ ""` or `a == n + 0`.


## String concatenation

String concatenation are currently compiled in 2 different ways:
1. Concatenation with a literal

```elm
x =
  a ++ "str"
```

```js
var x = a + 'str'
```

2. Concatenation with non-literal

```elm
str = "str"
x = a ++ str
```

```js
var str = 'str'
var x = _Utils_ap(a, str);
```

Elm defers back to `_Utils_ap` because it doesn't know whether `++` was used on a `String` or a `List`.

The suggestion would be to use version 1 whenever the type of the value is known to be a `String`.

Sometimes this is forced by developers by doing `a ++ b ++ ""`.

## List concatenation

For `List`, the compiler always uses `_Utils_ap`, even if the value is a literal list.

But we could do something better.
Somewhat similarly, when the type is known to be a `List`, we could use the dedicated function for list concatenation instead of `_Utils_ap` which checks for the types at runtime.

---

When the compiler knows it is doing a comparison with a type that is comparable through `===`, it uses that. But since 