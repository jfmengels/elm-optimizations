[[Elm optimization techniques]]

`String.startsWith` should be replaced by the following:

```js
var $elm$core$String$startsWith = F2(
    function (prefix, str) {
        var prefixLength = prefix.length;
        var strLength = str.length;
        var i;
        if (strLength < prefixLength) {
            return false;
        }
        for (i = 0; i < prefixLength; i++) {
            if (prefix[i] !== str[i]) { return false; }
        }
        return true;
    });
```

## Benchmarks

[Benchmark code](https://github.com/jfmengels/elm-benchmarks/blob/main/src/ImprovingPerformance/ElmCore/StringStartsWith.elm)

Chrome

![](https://github.com/jfmengels/elm-benchmarks/blob/main/src/ImprovingPerformance/ElmCore/StringStartsWith-Results-Chrome.png?raw=true)

Firefox

![](https://github.com/jfmengels/elm-benchmarks/blob/main/src/ImprovingPerformance/ElmCore/StringStartsWith-Results-FF.png?raw=true)

## Further work

The algorithm may potentially be improved through micro-optimizations. I tried a few alternatives already: [Fine-tuning benchmarks](https://github.com/jfmengels/elm-benchmarks/blob/main/src/ImprovingPerformance/ElmCore/StringStartsWith2.elm)

## Resources

- [PR to use replacement in `elm-optimize-level-2`](https://github.com/mdgriffith/elm-optimize-level-2/pull/103)
- [Trying (and Failing) to Speed Up String.startsWith](https://bytes.zone/posts/trying-and-failing-to-speed-up-string-beginswith/)