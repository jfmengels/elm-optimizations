[[Elm optimization techniques]]

Changing `Set.map` to avoid an intermediate list representation.
`Set.insert` always creates a new `Set`, regardless of whether the element was already present or not, which is bad for performance.

```elm
altSetMap : (comparable -> comparable2) -> Set comparable -> Set comparable2
altSetMap func set =
    Set.foldl (\x acc -> Set.insert (func x) acc) Set.empty set
```

This is benchmarked in https://github.com/jfmengels/elm-benchmarks/blob/main/src/ImprovingPerformance/ElmCore/SetMap.elm and shows a significant performance increase.

When having access to the internals we can probably even make it slightly faster by calling `Dict.foldl` directly, which avoids an intermediate anonymous function definition and call.

I haven't benchmarked this version yet.
```elm
-- Name changes are to reflect what's in `Set.foldl`
map : (comparable -> comparable2) -> Set comparable -> Set comparable2
map func set =
    Dict.foldl (\key _ state -> insert (func key) state) empty set
```

---

Prior art:  https://github.com/miniBill/elm-fast-dict/blob/main/src/FastSet.elm#L261-L265