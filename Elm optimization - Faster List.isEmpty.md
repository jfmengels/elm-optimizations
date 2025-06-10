---
tags:
  - elm-performance
---
[[Elm optimization techniques]]

Simplify `$elm$core$List$isEmpty`

```js
var $elm$core$List$isEmpty = function (xs) {
	if (!xs.b) {
		return true;
	}
	else {
		return false;
	}
};
// -->
var $elm$core$List$isEmpty = function (xs) {
	return !!xs.b;
};
```

I'm not sure how this would be achieved in practice though. Maybe through [[Elm optimization - Speed up comparisons by compiling to pattern matches]]?