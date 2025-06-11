[[Elm optimization techniques]]

[Replacement in `elm-optimize-level-2`](https://github.com/mdgriffith/elm-optimize-level-2/pull/79)

We could use the browser's version
[String.prototype.repeat()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/repeat) if it's available:

```js
var _String_repeat = "".repeat ? F2(function(n, str) { return str.repeat(n); } : F2(function(n, str)
{
	// Implementation from eol2
});
```

We should however check that our implementation does not happen to be faster than the native `String.prototype.repeat`.