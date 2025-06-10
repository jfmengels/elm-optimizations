---
title: 'Prune unused record fields in the Elm compiler '
updated: 2022-09-29 21:36:31Z
created: 2022-09-28 07:29:10Z
tags:
  - elm-performance
---
[[Elm optimization techniques]]

We should try to figure out how do-able it would be, from within the Elm compiler, to prune dead code.

Having the compiler remove this kind of dead code means that it would open up for APIs like
`Rule.moduleRule.new` and `Rule.moduleRule.newWithX`, which are currently considered bad
because the code won't be DCE'd.

Things that the compiler could do:

### Remove unused fields

If a record (from a type mostly) is created but one of it's fields is never used, then we can remove the field.

```elm
type alias A = { b : C, unused : X }

init : A
init = { b = C.new, unused = X.create 1 2 3 4 }

foo : A -> C
foo a =
  a.b

-->

type alias A = { b : C }

init : A
init = { b = C.new }

foo : A -> C -- or foo : { a | b : C } -> C
foo a =
  a.b
```

The compiler should do this while knowing which functions will in practice be included in the bundle. So if there is a function in the code that uses the `unused` field but that that function is never reached from `main`, then we should still be able to remove the field.

If some functions through this change become unreachable from main, then they should be removed from the final bundle. This probably has to be done in some kind of loop.


### Replace unnecessary field assignements

If a record is created but one of it's fields is never used in a specific context, then we can replace it by something that will be smaller in computation time and/or memory.


```elm
type alias A = { b : C, unused : X }

callFoo : C
callFoo =
  foo { b = C.new, unused = X.create 1 2 3 4 }

foo : A -> C
foo a =
  a.b

functionThatUsesUnused : A -> X
functionThatUsesUnused a =
  a.unused

-->
type alias A = { b : C, unused : X }

callFoo : C
callFoo =
  foo { b = C.new, unused = 0 } -- the value of unused is not used, we can replace it by something shorter.

foo : A -> C
foo a =
  a.b

functionThatUsesUnused : A -> X
functionThatUsesUnused a =
  a.unused
```

TODO Check if this is considered as a "shape" change and will therefore cause deoptimizations.


### Kernel code

There may be cases where the Elm compiler can't tell which fields will be used, especially when it calls 
functions defined in kernel code.

If necessary, these functions should annotate which fields they expect, and tuose would be considered as used.

### Expected gains

I hope that this makes some API choixes possible.

I also think it will help reduce the bundle size. It should be fairly noticeable for small projects using elm-ui, which uses a lot of records that will end up bot being used.