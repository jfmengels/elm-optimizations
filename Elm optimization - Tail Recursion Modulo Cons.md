Full explanation of the technique in [Tail recursion, but modulo cons](https://jfmengels.net/modulo-cons)

This increases performance, as shown by [these benchmarks](https://github.com/jfmengels/elm-benchmarks/tree/master/src/TailCallRecursionExploration).

By re-implementing functions in a TRMC-compatible way ([`elm/core`](https://github.com/jfmengels/core/compare/2fa34772a2575d036c0871b4390379741e6f5f91...new-tail-recursion), [`elm-community/list-extra`](https://github.com/elm-community/list-extra/compare/master...jfmengels:new-tail-recursion)), we can get the same results as what `elm-optimize-level-2` replaces, like a 6-7x faster `List.map`. 