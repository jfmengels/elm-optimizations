[[Elm optimization techniques]]

[Optimized function wrappers by robinheghan · Pull Request #51 · mdgriffith/elm-optimize-level-2](https://github.com/mdgriffith/elm-optimize-level-2/pull/51)

Oddly, this does not seem to be implemented into Gren, which uses [the same wrappers](https://github.com/gren-lang/compiler/blob/main/compiler/src/Generate/JavaScript/Functions.hs) as the Elm compiler.

[This article](https://blogg.bekk.no/making-elms-a-functions-more-predictable-a-failed-experiment-2ea4b86b5345) (that came out after Robin's PR to `elm-optimize-level-2`) indicates that the idea did not work out, although I'm not entirely sure it's the exact same solution as the one in the PR.

If they are the same however, then this should be removed from `elm-optimize-level-2`, right?
