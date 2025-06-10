[[Elm optimization techniques]]

I sometimes see the following code:
```elm
data
  |> fn1
  |> (\x -> fn2 x otherArg)
  |> fn3
```

which compiles to

```js
fn3(
  function (x) {
    return A2(fn2, x, otherArg);
   }(fn1(data))
);
```

In my opinion, this should compile down to the somewhat nicer:

```js
fn3(
  A2(
    fn2,
    fn1(data),
    otherArg
  )
);
```

It's not a huge improvement, and JIT inlining might be smart enough to do just that. To be benchmarked.