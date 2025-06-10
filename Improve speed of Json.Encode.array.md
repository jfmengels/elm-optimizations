[[Elm optimization techniques]]

```elm
{-| Turn an `Array` into a JSON array.  
-}  
array : (a -> Value) -> Array a -> Value  
array func entries =  
    Elm.Kernel.Json.wrap  
        (Array.foldl (Elm.Kernel.Json.addEntry func) (Elm.Kernel.Json.emptyArray ()) entries)
```

```js
function _Json_addEntry(func)  
{  
  return F2(function(entry, array)  
  {  
   array.push(_Json_unwrap(func(entry)));  
   return array;  
  });  
}
```


We can probably pre-create a JS `Array` with the same length as the array (which is fast to get), and then set the element at the index in a loop (following is pseudocode):

```js
var array = Array.from({length: <baseArray length>})
for (element, index in baseArray) { array[index] = element; }`
```

To be experimented with:
- `new Array({length: <baseArray length>}).fill(<first element>)`: to have the right type in the array to start with
- `Array.from({length: <baseArray length>}, (x, i) => baseArray[i]);`

We also need to check which methods are compatible with ES5.

https://2ality.com/2018/12/creating-arrays.html

---

If we have [[Elm optimization - Use JavaScript Arrays under the hood of Array]], then we can use the array as is. We would still need to create a copy so that the outside world can't mutate the array.