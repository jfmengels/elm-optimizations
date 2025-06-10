[[Elm optimization techniques]]

```elm
Builder.init
  |> Builder.withX x
  |> Builder.enableY
```

```elm
-- Builder.elm
init = Builder { x = Nothing, y = False, ... }

withX x (Builder b) =
  Builder { b | x = Just x }
  
enableY (Builder b) =
  Builder { b | y = True }
```

The builder pattern is a decently common pattern in Elm.

It's not slow, but I've always figured we could probably have it be much faster, by inlining and collapsing the code through the compiler.

```elm
Builder.init
  |> (\(Builder b) -> Builder { b | x = Just x })
  |> (\(Builder b) -> Builder { b | y = True })
-->
Builder.init
  |> (\(Builder b) -> Builder { b | x = Just x, y = True })
```

---

Would this be less relevant if [[Elm optimization - Opportunistic mutation]] is implemented? As that should already speed up perf.