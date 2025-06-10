[[Elm optimization techniques]]

[Faster String.map, String.filter and String.reverse by robinheghan · Pull Request #1032 · elm/core · GitHub](https://github.com/elm/core/pull/1032/files#diff-1c6ad04926d98850dc9f41def7cd2dc5300867d30c37c84fcf13d824c03b52d9R37-R54)

Before:

```js
var _String_map = F2(function(func, string)
{
	var len = string.length;
	var array = new Array(len);
	var i = 0;
	while (i < len)
	{
		var word = string.charCodeAt(i);
		if (0xD800 <= word && word <= 0xDBFF)
		{
			array[i] = func(__Utils_chr(string[i] + string[i+1]));
			i += 2;
			continue;
		}
		array[i] = func(__Utils_chr(string[i]));
		i++;
	}
	return array.join('');
});
```

After:

```js
var _String_map = F2(function(func, string)
{
	var len = string.length;
	var result = '';
	var i = 0;
	while (i < len)
	{
		var word = string.charCodeAt(i);
		if (0xD800 <= word && word <= 0xDBFF)
		{
			result += func(__Utils_chr(string[i] + string[i+1]));
			i += 2;
			continue;
		}
		result += func(__Utils_chr(string[i]));
		i++;
	}
	return result;
});
```

We could even remove `__Utils_chr`, as I don't believe it's necessary ([[Elm optimization - Simplify Char operations by identity]]).