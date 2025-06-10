[[Elm optimization techniques]]

```elm
lambda.expression == Expression.FunctionOrValue [] firstVariableName
```

This will call

```js
_Utils_eqHelp(lambda.expression, A2(Expression.FunctionOrValue, _List_Nil, firstVariableName))
```

which is not very fast.

Writing the code like a case expression is much more efficient.

```elm
case lambda.expression of
  Expression.FunctionOrValue [] name ->
    x == name
  _ -> False
```

```js
var x = lambda.expression;
/* return */ x.$ === 1 && x.a.$ === 0 && _Utils_eqHelp(x.b, name)
```

That's because it knows the shape of the thing to compare it to. But for users it is also a lot more code.

## Proposal

When we know the shape of the thing to compare a value to, we should compile down to the same checks as case expression.

At this location we know the shape of `Expression.FunctionOrValue` and can therefore make something much faster, even going directly for the `$` field (maybe that's done as well in `_Utils_eqHelp`?)

```js
var x = lambda.expression;
/* return */ x.$ === 1 && x.a.$ === 0 && _Utils_eqHelp(x.b, firstVariableName)

// If the compiler remembers types or knows about the
// local type of firstVariableName (a String in this case)
// we could even have:

var x = lambda.expression;
/* return */ x.$ === 1 && x.a.$ === 0 && x.b === firstVariableName
```

## Use-case

Make it faster and more memory-efficient to do `x == Just y`, `a == ( b, c )` or `list == [ 1, 2 ]`. Or even the opposite of that: `x /= Just y`.

A potential side-effect is that people will write code like this more often, which is usually not preferable to using case expressions. But I have doubts on whether this will happen, I believe people will write code like this when they find it more convenient anyway, disregarding performance considerations.

## Cons

This might lead to more code and therefore bigger bundle sizes.

## Implementation

In the compiler, we could have a pass/step where we try to transform comparisons (as much as possible) to case expressions, which should then automatically yield the intended results.