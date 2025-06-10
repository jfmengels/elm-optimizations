[[Elm optimization techniques]]

https://www.roc-lang.org/functional#opportunistic-mutation

In Elm we don't have reference counting, meaning it is really hard to tell how many references something has.

## Count references in function

```elm
fn data =
  let
    list = [ 1, 2, 3 ]
  in
  if condition data then
    List.map f list
  else if otherCondition data then
    List.map g list
  else
    List.map g list ++ List.map f list
```

In this example, `list` is created locally, and we therefore know that this data structure has nothing that refers to it outside the function.

We can then count the number of references to `list` in every scope (branch). In every scope where there's only a single reference, it is safe to mutate the value. In the example above, we can mutate `list` in the first and second branches, but we can't in the third one because there are two references.

In practice even in that branch we can mutate one of the lists as soon as the other one has been evaluated, as long as we make sure we do it in the right order.

## Identify functions that return new references

The example above used `[ 1, 2, 3 ]`, because it it easy to tell that that is a new reference.

We would also need to identity all the functions which ones create a new reference, such as `List.map`, both from packages and project code.

If we can determine that a function returns a new reference always, then we know we can mutate the result in the call locations, at least at the shallowly.

I don't know if it will be easy to statically determine that `List.map` always returns a new reference.

## Operations

What operations could we apply when we know we can do opportunistic mutation?

### Record update

```elm
createRecord () = { a = 1, b = 2 }
x = createRecord()
y = { x | b = 100 }
```
In the case above, we could do

```js
var x = createRecord()
var y = x.b = 100, x;
```
TODO: The syntax above doesn't work, `y` gets value `100`.

Notably useful for the builder pattern, where many records are created.

### List concatenation

Adding to the end becomes simpler.

---

Note: We should never mutate the constant behind `[]`, gotta be careful about that, and some for constants in general. We could however mutate constants if we know they're only referred to by other constants (non-functions).