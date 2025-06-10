[[Elm optimization techniques]]

Improve the speed of [`_Json_runArrayDecoder`](https://github.com/elm/json/blob/1.1.3/src/Elm/Kernel/Json.js#L313-L327), which is used when [decoding lists or arrays](https://github.com/elm/json/blob/1.1.3/src/Elm/Kernel/Json.js#L214-L226).

```js
case __1_LIST:
	// ...
	return _Json_runArrayDecoder(decoder.__decoder, value, __List_fromArray);

case __1_ARRAY:
	// ...
	return _Json_runArrayDecoder(decoder.__decoder, value, _Json_toElmArray);

function _Json_runArrayDecoder(decoder, value, toElmValue)
{
	var len = value.length;
	var array = new Array(len);
	for (var i = 0; i < len; i++)
	{
		var result = _Json_runHelp(decoder, value[i]);
		if (!__Result_isOk(result))
		{
			return __Result_Err(A2(__Json_Index, i, result.a));
		}
		array[i] = result.a;
	}
	return __Result_Ok(toElmValue(array));
}
```

For Lists, we could skip creating an intermediate array, and fill the value bit by bit (TRMC style).

For Array, I can't see a good improvement, but we need to make sure we don't break it.

```js
case __1_LIST:
	// ...
	return _Json_runArrayToListDecoder(decoder.__decoder, value);

case __1_ARRAY:
	// ...
	return _Json_runArrayToArrayDecoder(decoder.__decoder, value);

function _Json_runArrayToListDecoder(decoder, value)
{
	var len = value.length;
	var out = _List_Cons(0, _List_Nil);
	for (var i = 0; i < len; i++)
	{
		var result = _Json_runHelp(decoder, value[i]);
		if (!__Result_isOk(result))
		{
			return __Result_Err(A2(__Json_Index, i, result.a));
		}
		out = out.b = _List_Cons(i, out);
	}
	return __Result_Ok(out.b);
}

function _Json_runArrayToArrayDecoder(decoder, value, toElmValue) {
    var len = value.length;
    var array = new Array(len);
    for (var i = 0; i < len; i++) {
        var result = _Json_runHelp(decoder, value[i]);
        if (!$elm$core$Result$isOk(result)) {
            return $elm$core$Result$Err($elm$json$Json$Decode$Index_fn(i, result.a));
        }
        array[i] = result.a;
    }
    return $elm$core$Result$Ok(_Json_toElmArray(array));
}
```