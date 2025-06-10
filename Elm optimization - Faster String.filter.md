[[Elm optimization techniques]]

[Faster String.map, String.filter and String.reverse by robinheghan · Pull Request #1032 · elm/core · GitHub](https://github.com/elm/core/pull/1032/files#diff-1c6ad04926d98850dc9f41def7cd2dc5300867d30c37c84fcf13d824c03b52d9)

Before:

```js
var _String_filter = F2(function(isGood, str)
{
	var arr = [];
	var len = str.length;
	var i = 0;
	while (i < len)
	{
		var char = str[i];
		var word = str.charCodeAt(i);
		i++;
		if (0xD800 <= word && word <= 0xDBFF)
		{
			char += str[i];
			i++;
		}

		if (isGood(__Utils_chr(char)))
		{
			arr.push(char);
		}
	}
	return arr.join('');
});
```

After:

```js
var _String_filter = F2(function(isGood, str)
{
	var result = '';
	var len = str.length;
	var i = 0;
	while (i < len)
	{
		var char = str[i];
		var word = str.charCodeAt(i);
		i++;
		if (0xD800 <= word && word <= 0xDBFF)
		{
			char += str[i];
			i++;
		}

		if (isGood(__Utils_chr(char)))
		{
			result += char;
		}
	}
	return result;
});
```

We could even remove `__Utils_chr`, as I don't believe it's necessary ([[Elm optimization - Simplify Char operations by identity]]).