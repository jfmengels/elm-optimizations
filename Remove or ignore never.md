[[Elm optimization techniques]]

`never` is only used to prove something to the compiler, but is in practice never used (by definition).

It is used in some common operations such as `Html.map never x` which would gain from being simplified to `x`.

Note that this will likely only have a **very** tiny performance impact, at least on its own.
## Solutions

### At compilation time

Strip operations that are passed `never` as an argument (such as `Html.map`), and replace it with `identity` when needed.

```elm
Html.map never x
--> x
Html.map never
--> identity
```

The tricky part is to know which operations can be stripped off. `Html.map` from `elm/html` is an example, but what do we do about `f never`?
Is there some analysis that can be done on `f` to figure out what to do? Or do we have a set of known operations to apply this optimization on? If the latter, then that means that similar map functions for instance will not benefit from this optimization.


With [[Elm optimization - Inline Basics.identity]], this could lead to removing even more operations:
```elm
List.map (Html.map never) list
-->
List.map identity list
-->
list
```


### At run time

We can also change the internals of specific functions to not do anything when the argument is never.

```js
var $elm$virtual_dom$VirtualDom$map = F2(function (mapper, node) {
  if (mapper === $elm$core$Basics$never) {
    return node;
  }
  // proceed with current behavior
});

// or


var $elm$virtual_dom$VirtualDom$map = function (mapper) {
  if (mapper === $elm$core$Basics$never) {
    return identity;
  }
  return function (node) {
    // proceed with current behavior
  };
});
```

Cons of this approach:
- This has a runtime cost
- This can only be achieved by kernel functions (`==` on functions can lead to crashes)
- This doesn't help with piling on more optimizations (unless we use the second version of the code above, and do the same check thing for `identity`).

## Related optimizations

- [[Elm optimization - Inline Basics.identity]]
- [[Elm optimization - Simplify Char operations by identity]]