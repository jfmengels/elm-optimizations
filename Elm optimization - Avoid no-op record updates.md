[[Elm optimization techniques]]

Prior art: https://blogg.bekk.no/improving-the-performance-of-record-updates-cb34cb6d4451

When you do
```elm
update msg model =
  case msg of
    X subMsg ->
      { model | x = X.update subMsg model.x }
```

if `X.update` returns the model changed (TODO give examples), then the model defined in this update function still gets recreated, even though with the exact same data.

This can defeat laziness at the next render, because a new reference is created.

---

TODO return-API?
TODO
```elm
update : Msg -> Model -> Model
update msg model =
  case msg of
    X subMsg ->
      let
	    x =
		  X.update subMsg model.x
      in
      if x == model.x then
        model
      else
        { model | x = x }
```


---

Yeah. We could use a very cheap JS == to check whether it's worth creating a new version of the model (or `MyPagesPage` here), to avoid unnecessary rewrapping or the creation of a new record.
I guess that could be a check in Utils_update but we would probably only want that in certain locations.
But even in this case, it's worth adding a lazy at the root of each page's view for similar reasons.

Maybe! Because it still feels like you have thought more about it than I have, I still feel skeptical that it brings much value … I’m trying to think about what kind of message happens that results in a no-op, and that would benefit from less work being done than if it resulted in an … op. I guess a good, real-world-ish example would be good in such an article!


Yeah, I'm mostly thinking of things that make you trigger a cmd (http request for instance) and no model update (triggered either by an interaction or by a subscription), or the handling of msgs you don't care about but are forced to, such as handling the result of Dom.focus (we should have a version where we can fire and forget IMO).

But yeah, I'll try to find good examples 👍
(From a conversation with [[Simon Lydell]])

---

```js
function _Utils_update(oldRecord, updatedFields)
{
	var newRecord = {};

	for (var key in oldRecord)
	{
		newRecord[key] = oldRecord[key];
	}

	for (var key in updatedFields)
	{
		newRecord[key] = updatedFields[key];
	}

	return newRecord;
}
```

If there's no cost to it, it's always better to reuse a value instead of creating a copy. It reduces lazy check failures, it reduces the amount of memory being used and then garbage collected, and it makes `==` faster.

## Solution

In `_Utils_update`—which is the underlying function for record updates, we could use a very cheap JS `==` to check whether it's worth creating a new version of the model, to avoid the unnecessary rewrapping or the creation of a new record.

The update would check for every field whether the value is the same. If all are the same, then the untouched value is returned. Otherwise, the fields are updated as necessary.

This would very likely have a performance impact, though hopefully it would be small. It would be a loop over the fields to modify , which is rarely large.
The `==` check itself should be rather quick too, though I don't know the performance on very large strings.

If this check is really cheap, then that's great, if it isn't, then we would likely only want it in certain locations. That means we would need new syntax for this kind of record update.

We could annotate this through a function call

```elm
avoidUnnecessaryUpdate { r | x = y } 
```

this should be pretty easily patchable through a regex on the compiled Elm code to use a different version of `_Utils_update`.
And in a compiler (like Lamdera) we could probably reuse whatever mechanism would be used to replace `Array.fromList [...]` (which afaik doesn't exist yet).

```js
function _Utils_update(oldRecord, updatedFields) {
  for (var key in updatedFields) {
    if (updatedFields[key] !== oldRecord[key]) {
      return _Utils_update(oldRecord, updatedFields)
    }
  }
  return oldRecord;
}
```
Mucro-optimization: Could we reuse the iterator so that we only try applying the new fields? Pass the iterator to `_Utils_update`.

---

To see if this improves performance, this could likely be patched into all branches of `update*` functions.

---

This will be of no use if Html.lazy is never used, which would be the case for elm-review for instance. Unless reducing the memory footprint helps performance.

---

It's tricky to know whether this is compatible with Robin's faster record updates.

```elm
newPerson = { person | name = "New name" }
```

```js
// Elm 0.19.1
var newPerson = _Utils_update(person, { name: "New name" });

// Robin's version
var newPerson = (function()
  var $c = person.$clone()
  $c.name = "New name";
  return $c;
)();
```

Proposal:

```js
var newPerson = xyz(person, ["name", "New name"], function() {
  var $c = person.$clone()
  $c.name = "New name";
  return $c;
});

function xyz(oldRecord, fieldAndValues, thunk) {
  for (var i = 0; i < fieldAndValues.length; i += 2) {
    var key = fieldAndValues[i];
    var value = fieldAndValues[i + 1];
    if (oldRecord[key] !== value) {
      return thunk();
    }
  }
}
```

Downsides:
- This could be slower, especially since we're iterating over a list (in order to avoid increasing the bundle size too much, as)
- We're duplicating values. This obviously isn't great, so we'd have to find a nice way to reduce this

```js
var newPerson = (function() {
  var f$name = "New name";
  if (person.name !== f$name
    // || person.otherField !== f$otherField || ...
  ) {
	  var $c = person.$clone()
	  $c.name = f$name;
      // $c.otherField !== f$otherField
      // ...
	  return $c;
  }
  return person;
})();
```

Downsides:
- This is a lot more code than before.

---

## Ideas for Robin's version

### Only generate classes when necessary

We know statically which fields are present in record updates. Therefore, we know which records are never part of a record update because they don't have those fields. For these, we could use regular records, instead of classes. And skip the generation of the class.

Counterpoint: as indicated in Robin's article, using classes makes data/functions monomorphic instead of polymorphic, so there's value in that. Counter-counterpoint: Only records that are updated in record updates defeat monomorphism, so using regular objects still make sense.

There would be some false analysis with duplicate field names, potentially this could be improved by looking at potential paths for record updates, opaque types, etc.
For instance, if a record is not exposed outside of a module, and neither the module nor any of the file it imports (in)directly has a record update for fields the type contains. Then we know this type is never getting updated.

### Avoid wrapping in function for let variables (and more?)

```elm
let
  person = {...}
  newPerson = { person | name = "New name" }
in
...
```

```js
var person = { ... }
var newPerson = (function()
  var $c = person.$clone()
  $c.name = "New name";
  return $c;
)();

// Suggestion
var person = { ... }
var newPerson = person.$clone();
newPerson.name = "New name";
...
```

Maybe this could be applicable in other places. The immediately-invoked function expression is only really useful if there is a need to have it be an expression. If it can be extracted as an instruction, then it's not necesary

Do minifiers already handle this by any chance?
Does removing the wrapper function even improve performance? (Likely but...)