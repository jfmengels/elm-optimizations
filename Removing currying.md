[[Elm optimization techniques]]

I'm thinking we can probably remove currying altogether in the compiled output, by transforming an partial application call into a lambda.

```elm
add a b c =
  a + b + c

List.foldl (add 1) 0 list
```

We know statically that `add` takes 3 arguments and is given 1, therefore we know it requires 2 more arguments.
We can uncurry it by wrapping it in a lambda that takes the 2 remaining arguments:

```elm
add a b c =
  a + b + c

List.foldl (\b c -> add 1 b c) 0 list
```

If we then also remove all the `AX` calls, this should yield better performance and allow for more inlining by the JIT (potentially again increasing improving performance).

```js
List$foldl(
  function(b, c) { return add(1, b, c); },
  0,
  list
)
```

Problem: If some of the arguments are slow to compute, we don't want them to be computed many many times. Therefore we may need to declare a variable outside the function.

The easiest is likely through a closure.

```js
List$foldl(
  function() {
    var arg1 = 1;
    return function(b, c) {
      return add(arg1, b, c); }
    }
  }(),
  0,
  list
)
```

Or through variables declared before the expression. Each variable would have to be unique in this scope.

```js
var arg1 = 1;
return List$foldl(
  function(b, c) { return add(arg1, b, c); },
  0,
  list
)
```

The value could also be set in an expression like this, delaying when it is computed.

```js
var arg1;
return List$foldl(
  arg1 = 1, function(b, c) { return add(arg1, b, c); },
  0,
  list
)
```

This could be useful when the variables have to be declared before an `if` but the value is computed in a branch. That way we don't compute the value unnecessarily, which could potentially be terrible for performance. Or in the case of a `case` expression, impossible.

If the function only takes a single argument, then we don't need to do define separate variables, the function is already in the ideal shape.

Also, if we notice some arguments are not complex expressions (e.g: string/number literals, constants), then we can skip defining a variable for those too.

## Knowing the number of arguments

The idea above esteems we know the number of missing arguments.

I think an applicative is such as instance, although that one may simply be a missing single argument 🤷‍♂️

What do we do when we don't know? We could keep the current behavior of Fx/Ax functions, that would be easiest.

Is it even possible for the compiler to not really know in practice how many arguments a function expects?
 
TODO How should those be handled?


## Benefits

Performance improvements.

If we have this capability, then compiling to Roc or other languages without currying becomes possible (or at least easier).