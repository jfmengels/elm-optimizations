[[Elm optimization techniques]]

Make `String.startsWith` use https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/startsWith when available:
```js
var _String_startsWith = "".startsWith ? F2(function(sub, str) { return str.startsWith(sub); } : F2(function(sub, str)
{
	return str.indexOf(sub) === 0;
});
```

Also, faster version here: https://github.com/mdgriffith/elm-optimize-level-2/pull/103
We should benchmark the two, because if the version in eol2 is faster, then it might not be useful to use `String.prototype.startsWith`.