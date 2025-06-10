[[Elm optimization techniques]]

Similar to [[Elm optimization - Faster Set.insert]], if a key is not found in a `Dict`, then there's no need build a new `Dict`, and we can instead return the original one.

An application of this idea was made in [Faster FastDict.remove by jfmengels · Pull Request #7 · miniBill/elm-fast-dict · GitHub](https://github.com/miniBill/elm-fast-dict/pull/7) (which already returned the original `Dict` but built it anyway).

---

Note: Robin introduced a faster variant of `Dict.remove` in: [Faster insert, remove and update for Dict by robinheghan · Pull Request #1033 · elm/core · GitHub](https://github.com/elm/core/pull/1033/files)