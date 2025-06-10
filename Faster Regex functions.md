---
title: Regex performance improvements
updated: 2022-09-25 13:11:32Z
created: 2022-09-25 12:49:04Z
tags:
  - elm-performance
---
[[Elm optimization techniques]]

Potential ideas:

## Use /regex/.test() in Regex.contains

```js
var _Regex_contains = F2(function(re, string)
{
	return string.match(re) !== null;
});
```
could maybe be made faster by using `test`
```js
var _Regex_contains = F2(function(re, string)
{
	return re.test(string);
});
```

To be benchmarked

## Simplify _Regex_findAtMost

```js
var _Regex_findAtMost = F3(function(n, re, str)
{
	var out = [];
	var number = 0;
	var string = str;
	var lastIndex = re.lastIndex;
	var prevLastIndex = -1;
	var result;
	while (number++ < n && (result = re.exec(string)))
	{
		if (prevLastIndex == re.lastIndex) break;
		var i = result.length - 1;
		var subs = new Array(i);
		while (i > 0)
		{
			var submatch = result[i];
			subs[--i] = submatch
				? __Maybe_Just(submatch)
				: __Maybe_Nothing;
		}
		out.push(A4(__Regex_Match, result[0], result.index, number, __List_fromArray(subs)));
		prevLastIndex = re.lastIndex;
	}
	re.lastIndex = lastIndex;
	return __List_fromArray(out);
});
```

1. `var string = str;` seems unnecessary, we can (should be able to?) use `str` directly, and remove this line. We could check whether uglifiers worry about this.
 
2. Instead of using `__List_fromArray` (at the end and for `subs`), we could build a list directly (using `__List_cons` I think, or TRMC with holes). Saves a bit of memory maybe.


## Regex.find should use string.matchAll

Instead of doing a manual iteration where we use `exec/match`, we could use `matchAll` which is probably faster.

Not available for IE11 apparently: https://caniuse.com/mdn-javascript_builtins_string_matchall

We can probably do the same thing for the replace functions

## Regex.split

Same as the ideas above:
1. Build the list as we go
2. No need to assign `var string = str`
3. For the split "all", use JS' `str.split`