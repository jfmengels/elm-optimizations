[[Elm optimization techniques]]

`Set.insert` always creates a new `Set`, regardless of whether the element was already present or not, which is bad for performance.

```elm
insert : comparable -> Set comparable -> Set comparable
insert a set =
    insertHelp a set
        |> Maybe.withDefault set


insertHelp : comparable -> Set comparable -> Maybe (Set comparable)
insertHelp key set =
    case set of
        RBEmpty_elm_builtin ->
            -- New nodes are always red. If it violates the rules, it will be fixed
            -- when balancing.
            RBNode_elm_builtin Red key () RBEmpty_elm_builtin RBEmpty_elm_builtin
                |> Just

        RBNode_elm_builtin nColor nKey nValue nLeft nRight ->
            case compare key nKey of
                LT ->
                    case insertHelp key nLeft of
	                    Just newLeft ->
		                    balance nColor nKey nValue newLeft nRight

						Nothing ->
							Nothing

                EQ ->
                    Nothing

                GT ->
                    case insertHelp key nRight of
	                    Just newLeft ->
		                    balance nColor nKey nValue nLeft newRight

						Nothing ->
							Nothing
```

We could do the same for `Dict`, but then we would have to do an equality check on the values (which may not be `equatable`).
This is again a reason why I think `Set`s could be best reimplemented from scratch rather than re-using `Dict`.
(Or maybe we keep the data structure but some particular algorithms should be reimplemented to optimize for `Set`s lack of values)

This would make `Set.union` and `Set.fromList` also faster, as they use many `Set.insert` under the hood.

We could use the same technique for `Set.remove`: if it's not there, don't create a copy. If it is there, only then create a copy.
Similarly, this would make `Set.union` faster.

---

An application of this idea was made in [Faster insertNoReplace and FastSet.insert by jfmengels · Pull Request #6 · miniBill/elm-fast-dict · GitHub](https://github.com/miniBill/elm-fast-dict/pull/6)

---

Note: Robin introduced a faster variant of `Dict.insert` in: [Faster insert, remove and update for Dict by robinheghan · Pull Request #1033 · elm/core · GitHub](https://github.com/elm/core/pull/1033/files)