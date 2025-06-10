[[Elm optimization techniques]]

I often see code like the following:
```elm
case f x of
    Ok value ->
        g value

    Err error ->
        Err error
```

What bothers me here is that we destructure `Err error` and we recreate the same value again right away.

It would be more memory efficient to re-use the same value

```elm
case f x of
    Ok value ->
        g value

    (Err _ as error) ->
        error
```

Unfortunately, sometimes the former is mandatory because the types do not match. i.e. if the type of `f x` is `Result SomeError SomeValue` while the expression evaluates to `Result SomeError SomeOtherValue`.

However, I think we can go around the memory waste even in this case. Let's say we have the first piece of code again. Since the compiler knows the code type-checks, 
Therefore, I think the compiler could improve the code generation 

## Inference

Another potential solution is to have the compiler refine the type in the pattern match. If we take the second example, if the compiler was able to figure out that `error` has an unbound type variable for the `value` (from `Result error value`), then it could consider both branches to be of the same type.

However, I doubt that this is easily do-able in Elm. Maybe it is, but maybe it is not desirable because of other consequences this might have?
I think this is closer to how inference could work in Roc?

---

I'm wondering something. Given this code (https://ellie-app.com/vhJtpk4bzs4a1)

```
f : () -> Result () A
g : A -> Result () B

doesntCompile : Result () B
doesntCompile =
    case f () of
        Ok value ->
            g value

        (Err _) as error ->
            error
```

this doesn't compile because the first branch returns a `Result () B` but the second branch returns a `Result () A`.
It does compile if we change the second branch to:
```
        Err error ->
            Err error
```

Does anyone know whether it would be possible for the Elm compiler to figure out that (in the first example `error` is "in practice" a `Result () unbounded`, because the variant does not have `value` in its definition, and therefore consider both branches to be compatible? Or would this have a different nefarious consequence?