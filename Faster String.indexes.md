---
title: Functions to optimize
updated: 2021-12-14 12:15:21Z
created: 2021-12-10 08:16:28Z
tags:
  - elm-performance
---
[[Elm optimization techniques]]

Original implementation:

```js
var _String_indexes = F2(function(sub, str)
{
 var subLen = sub.length;

 if (subLen < 1)
 {
  return __List_Nil;
 }

 var i = 0;
 var is = [];

 while ((i = str.indexOf(sub, i)) > -1)
 {
  is.push(i);
  i = i + subLen;
 }

 return __List_fromArray(is);
});
```

We could fuse together the pushing to the JS `Array` and `__List_fromArray` to reduce the amount of work.

```js
var _String_indexes = F2(function(sub, str)
{
 var subLen = sub.length;

 if (subLen < 1)
 {
  return __List_Nil;
 }

 var i = 0;
 var out = _List_Cons(0, _List_Nil);

 while ((i = str.indexOf(sub, i)) > -1)
 {
  out = out.b = _List_Cons(i, out);
  i = i + subLen;
 }

 return out.b;
});
```

Implementation should be verified and benchmarked!