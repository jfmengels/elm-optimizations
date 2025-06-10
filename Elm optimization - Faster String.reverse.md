[[Elm optimization techniques]]

[Faster String.map, String.filter and String.reverse by robinheghan · Pull Request #1032 · elm/core · GitHub](https://github.com/elm/core/pull/1032/files#diff-1c6ad04926d98850dc9f41def7cd2dc5300867d30c37c84fcf13d824c03b52d9)

Before:

```js
function _String_reverse(str)
{
	var len = str.length;
	var arr = new Array(len);
	var i = 0;
	while (i < len)
	{
		var word = str.charCodeAt(i);
		if (0xD800 <= word && word <= 0xDBFF)
		{
			arr[len - i] = str[i + 1];
			i++;
			arr[len - i] = str[i - 1];
			i++;
		}
		else
		{
			arr[len - i] = str[i];
			i++;
		}
	}
	return arr.join('');
}
```

After:

```js

function _String_reverse(str)
{
	var result = '';
	var i = str.length;
	while (i--)
	{
		var char = str[i];
		var word = str.charCodeAt(i);
		if (0xDC00 <= word && word <= 0xDFFF)
		{
			i--;
			char = str[i] + char;
		}
		result += char;
	}
	return result;
}
```

We could even remove `__Utils_chr`, as I don't believe it's necessary.