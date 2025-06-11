[[Elm optimization techniques]]

The current implementation is the following:

```elm
{-| Keep only the key-value pairs that pass the given test.  
-}  
filter : (comparable -> v -> Bool) -> Dict comparable v -> Dict comparable v  
filter isGood dict =  
    foldl  
        (\k v d ->  
            if isGood k v then  
                insert k v d  
  
            else  
                d  
        )  
        empty  
        dict
```

This loops the entire dict (`O(n)`), and gradually inserts (`O(log n)`) elements that pass the predicate. This therefore has a `O(n×log(n))` complexity (TODO toverify).

That creates a lot of intermediate dictionaries that will each be discarded at the next insertion, therefore using up a lot of memory.

## Idea

Instead of folding and then re-inserting, we go through the tree once and remove elements as we go.

## Implementation

```elm
filter : (comparable -> Bool) -> Set comparable -> Set comparable
filter isGood dict =
    -- Root node is always Black
    case filterHelp isGood dict of
        RBNode_elm_builtin Red k v l r ->
            RBNode_elm_builtin Black k v l r

        x ->
            x


filterHelp : (comparable -> Bool) -> Set comparable -> Set comparable
filterHelp isGood set =
    case set of
        RBEmpty_elm_builtin ->
            set

        RBNode_elm_builtin nColor nKey value nLeft nRight ->
            let
                newLeft : Set comparable
                newLeft =
                    filterHelp isGood nLeft

                newRight : Set comparable
                newRight =
                    filterHelp isGood nRight
            in
            if isGood nKey value then
                balance nColor nKey value newLeft newRight

            else
                case getMin newRight of
                    RBNode_elm_builtin _ minKey minValue _ _ ->
                        balance nColor minKey minValue newLeft (removeMin newRight)

                    RBEmpty_elm_builtin ->
                        RBEmpty_elm_builtin
```

Note: This version likely doesn't respect the invariants of red-black trees, and this therefore needs to be fixed.
## Related

[PR for `miniBill/elm-fast-dict`](https://github.com/miniBill/elm-fast-dict/pull/5) where the same idea has been applied, but I failed to respect the invariants of the tree. Maybe there's an answer on how to do that in https://en.m.wikipedia.org/wiki/Red%E2%80%93black_tree#Set_operations_and_bulk_operations

In that package, the following version also performs really well compared to its previous implementation and against `elm/core`'s version, and is what is being used in the package now ([PR](https://github.com/miniBill/elm-fast-dict/pull/11)).

```elm
filter : (comparable -> a -> Bool) -> Dict comparable a -> Dict comparable a
filter f dict =
    foldr
        (\k v acc ->
            if f k v then
                ListWithLength.cons ( k, v ) acc
        
            else
                acc
        )
        ListWithLength.empty
        dict
        |> Internal.fromSortedList
```

## Consequences

Improving this would also improve the performance of `Set.filter`.