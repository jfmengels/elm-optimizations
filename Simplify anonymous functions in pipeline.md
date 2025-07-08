[[Elm optimization techniques]]

The following form
```elm
a
  |> f
  |> (\x -> g x y)
  |> h
```

currently compiles down to

```js
return h(
  (function (x) {
    return A2(g, x, y);
  })(f(a)),
);
```

but it should be equivalent to

```elm
let x =
  a
     |> f
in
g x y
  |> h
```

which compiles to

```js
var x = f(a);
return h(A2(g, x, y));
```
That removes one function definition from the compiled code.

If done at the compiler level, then this should work even if the name of the argument is repeated.
```elm
a
  |> f
  |> (\x -> g x y)
  |> (\x -> h x y)
  |> i
```

```js
var x = f(a);
var x = A2(g, x, y);
return i(A2(h, x, y));
```

Proof of concept that declaring and referencing a `var` twice works in JS:
```js
function a() { var i = 10; var i = i + 1; return i; }
// 11
```

Note: If Elm would ever switch to declaring variables using `let`/`const`, then this would be a syntax error. The variable would have then be declared using `let` and be overwritten in a regular assigment.

---

This proposal is mostly for pipelines, but it should also apply for plain functions calls, though that is a rare style.

```elm
(\x -> f x y) data
```

## Tail-calls

If the last call is a recursive function call, then this should be rewritten like is done for tail-call elimination.

```elm

```

An approach I would recommend looking into is to at (after checks but before generation) internally rewrite/optimize the AST to a form that makes it look like it was originally written using let declarations.
That way, the rest of the existing optimizations would work out automatically.
