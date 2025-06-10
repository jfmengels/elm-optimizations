[[Elm optimization techniques]]


```js
function _String_fromList(chars)
{
	return __List_toArray(chars).join('');
}
```

We could try to use a loop to build a String directly, so that we don't build an intermediate JavaScript Array?

```js
function _String_fromList(chars)
{
    var s = '';
    
	for (var out = []; chars.b; chars = chars.b) // WHILE_CONS
	{
		s += chars.a;
	}
	return s;
}
```

To be benchmarked.