[[Elm optimization techniques]]

Chars are strings in JavaScript, so some things can be removed.

`_Utils_chr` can be re-implemented as `identity`.

---

```
var $elm$core$String$fromChar = function (_char) {
	return A2($elm$core$String$cons, _char, '');
};
```
=> `identity`
