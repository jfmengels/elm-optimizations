[[Elm optimization techniques]]

When compiling a tail-call recursive function, Elm uses the `continue` keyword to force going back the beginning of the loop.

```elm
goToEnd : List a -> List b  
goToEnd list =  
    case list of  
        [] ->  
            []  
  
        _ :: xs ->  
            fn xs
```

```js
var $author$project$A$goToEnd = function (list) {  
 goToEnd:  
 while (true) {  
  if (!list.b) {  
   return _List_Nil;  
  } else {  
   var xs = list.b;  
   var $temp$list = xs;  
   list = $temp$list;  
   continue goToEnd;  
  }  
 }  
};
```

The `continue` keyword can be useful if you have code like this

```js
if (condition) {
  // Do something
  continue label;
}
// Code that should be skipped if condition is true
```

but in Elm's case, since all branches are exclusive (through the use of `else` code blocks), the `continue` keyword serves no purpose.

From benchmarking the function (in June 2025 with Chrome and Firefox, not Safari) above with and without the `continue` instruction, the results seem to be the same.

![[continue-benchmark.png]]

I therefore think this instruction should be removed as well as the label it refers to, in order to reduce the bundle size. The gain will be almost non-existent after minification and gzip, but if there's no reason to keep it, we should not have it.