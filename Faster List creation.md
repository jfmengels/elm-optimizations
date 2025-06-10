[[Elm optimization techniques]]

If we have [[Elm optimization - Faster List.singleton]], then we could replace `_List_fromArray([x])` by `List.singleton(x)`. Or we could replace it with its inlined version.

---

`elm-optimize-level-2` also has a transformer 

[elm-optimize-level-2/src/transforms/inlineListFromArray.ts at master · mdgriffith/elm-optimize-level-2 · GitHub](https://github.com/mdgriffith/elm-optimize-level-2/blob/master/src/transforms/inlineListFromArray.ts)

Although this optimization is [turned off](https://github.com/mdgriffith/elm-optimize-level-2/blob/master/src/types.ts#L104) in practice. I don't remember the reasoning behind it, whether it was because it did not speed up the code or because it increased bundle size too much.