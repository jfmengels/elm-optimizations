[[Elm optimization techniques]]

Data structures have a `$` field. While it may be useful for some data structures, it doesn't seem useful for all of them.

Take `List` for instance:

```js
var _List_Nil = { $: 0 };
var _List_Nil_UNUSED = { $: '[]' };

function _List_Cons(hd, tl) { return { $: 1, a: hd, b: tl }; }
```

The way we check whether a value is `[]` or `x :: xs` in Elm is by checking the value of the `list.b`: if it's falsy it's `[]`, otherwise it's `x :: xs`.

We never use the `$` field for this type, therefore we should remove it. We might want to keep it for `Debug.log` but in `--optimize` mode there is no reason to keep it.

## Custom types

Other types might benefit from the same idea.

```elm
type Maybe a
  = Just a
  | Nothing

case maybe of
  Just x ->
    x :: xs
  Nothing ->
    xs
```

This is compiled down to

```js
if (!_v0.$) { // Just
  var x = _v0.a;
  return A2($elm$core$List$cons, x, xs);
} else { // Nothing
  return xs;
}
```

In this particular case we can't check for the value of the `a` field, because it might be a falsy value (ex: `Just 0`).

But when we know that the contained value is never falsy and that in all the other constructors it is missing, then we can.

```elm
type A
  = User Role
  | Guest

case user of
  User role ->
    Just Role
  Guest ->
    Nothing
```

could be compiled down to

```js
if (_v0.a) { // User
  var role = _v0.a;
  return Just(role);
} else { // Guest
  return Nothing;
}
```

lue suggested that we could compare with `undefined` instead of checking for falsiness, as `undefined` is never created elsewhere in Elm. It is more verbose but doesn't require knowing the contents of the types (ideal for `Maybe` for instance where the type is unknown).

---

This becomes less and less applicable when there are more constructors, because if some of them have the same arity then this isn't possible, we need `$` again.
