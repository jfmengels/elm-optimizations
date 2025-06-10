[[Elm optimization techniques]]

Original implementations:

```js
function _String_words(str)
{
	return __List_fromArray(str.trim().split(/\s+/g));
}

function _String_lines(str)
{
	return __List_fromArray(str.split(/\r\n|\r|\n/g));
}
```

As far as I can tell, regexes are somewhat slow to create.
In these functions, they will be recreated on every call, because they're defined inside of them.

It would be better to define them outside the function.

```js
var wordsRegex = /\s+/g;
function _String_words(str)
{
	return __List_fromArray(str.trim().split(wordsRegex));
}

var linesRegex = /\r\n|\r|\n/g;
function _String_lines(str)
{
	return __List_fromArray(str.split(linesRegex));
}
```

But regexes with `g` are stateful, they remember the last index of what they were run on (or something like that, not entirely sure), so we got to make sure we exit or enter the function while leaving them in a nice state. By doing it through entering, the code is simpler so I'll go for that.

```js
var wordsRegex = /\s+/g;
function _String_words(str)
{
    wordsRegex.lastIndex = 0;
	return __List_fromArray(str.trim().split(wordsRegex));
}

var linesRegex = /\r\n|\r|\n/g;
function _String_lines(str)
{
    linesRegex.lastIndex = 0;
	return __List_fromArray(str.split(linesRegex));
}
```

List.take: Can create a list without reversing it.