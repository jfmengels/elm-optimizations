[[Elm optimization techniques]]

`Dict.update` can be made faster
1. By not doing anything when the value is not found and stays not found (`Nothing` -> `Nothing`)
2. If the node is not found in the traversal, then we know the exact position where it should be inserted (just needs balancing).
3. By going through the tree and calling the `alter` function where the key is found. Then changing the value in place or removing the node.

---

Note: Robin introduced a faster variant of `Dict.update` in: [Faster insert, remove and update for Dict by robinheghan · Pull Request #1033 · elm/core · GitHub](https://github.com/elm/core/pull/1033/files)

It does however require changing the definition of `Dict` to add a third kind of node.