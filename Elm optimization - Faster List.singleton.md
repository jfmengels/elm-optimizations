[[Elm optimization techniques]]

Before:

```js
var $elm$core$List$singleton = function (value) {
    return _List_fromArray([value]);
};
```

After:

```js
var $elm$core$List$singleton = function (value) {
    return _List_Cons(value, _List_Nil);
};
```