[[Elm optimization techniques]]

Published as a Gist: https://gist.github.com/jfmengels/3843f880622a54d1abce66a0f2590560

Looking at the compiled Elm code, I found that the use of Elm's `Json.Encode` module can be simplified and made more performant.

```elm
Json.Encode.object
  [ ( "a", a )
  , ( "b", b )
  ]
```

(Note: this example assumes `a` and `b` are already `Json.Encode.Value`s and doesn't need to use functions like `Json.Encode.string`).

This currently compiles to a bunch of JavaScript function calls with Elm list creation and iteration, which is quite slow ([benchmark](https://jsbench.me/fsmb0w5cw1/1)).

```js
$elm$json$Json$Encode$object(
	_List_fromArray(
		[
			_Utils_Tuple2('a', a),
			_Utils_Tuple2('b', b)
		]
	)
)

var $elm$json$Json$Encode$object = function (pairs) {
 return _Json_wrap(
  A3(
   $elm$core$List$foldl,
   F2(
    function (_v0, obj) {
     var k = _v0.a;
     var v = _v0.b;
     return A3(_Json_addField, k, v, obj);
    }),
   _Json_emptyObject(_Utils_Tuple0),
   pairs));
};
function _Json_emptyObject() { return {}; }
var _Json_addField = F3(function(key, value, object)
{
	object[key] = _Json_unwrap(value);
	return object;
});
function _Json_wrap(value) { return value; }
function _Json_unwrap(value) { return value; }
```

## Skip List_fromArray

This is an optimization that is possible for many core functions and data structures (`Array.fromList`, `Set.fromList`, etc.). When the argument is a list literal, the compiled JS code consists of a literal JavaScript `Array`, wrapped in a `_List_fromArray` which converts it to an Elm `List`, which then gets consumed by the Elm function, in this case using `List.foldl`.

`_List_fromArray` iterates over the JS array and creates copy, which is in practice less performant than the JS `Array` to iterate through. So all in all, we do 2 iterations and we create 2 copies of the collection (on top of the resulting JS object).

When we notice that `Json.Encode.object` is applied on a list literal, then we can generate and use an alternative version of `Json.Encode.object` that takes as input a JavaScript `Array` instead of an Elm `List`, and remove the `_List_fromArray`.

```js
$elm$json$Json$Encode$object(
	_List_fromArray(
		[
			_Utils_Tuple2('a', a),
			_Utils_Tuple2('b', b)
		]
	)
)
// -->
$elm$json$Json$Encode$object$array(
	[
		_Utils_Tuple2('a', a),
		_Utils_Tuple2('b', b)
	]
)

var $elm$json$Json$Encode$object$array = function (pairs) {
 return _Json_wrap(
  // Or a for-loop, whatever is faster
  pairs.reduce(function(_v0, obj) {
     var k = _v0.a;
     var v = _v0.b;
     return A3(_Json_addField, k, v, obj);
    }),
    _Json_emptyObject(_Utils_Tuple0)
  );
};
```

## Use an object literal

When all keys are known, then 
Instead, we could compile this to a plain JavaScript object, while making sure to preserve the same order for the keys as listed in the Elm code (as changing that could have consequences on the JS side).

The example used so far could with this approach be simplified to the following:
```js
{ a: a, b: b }
```

This would be the ideal result of this optimization.
## Dynamic keys

If there are keys that are not known at compile time. Then we can't replace everything by a simple object literal. And remember that we should maintain the order in which keys are inserted into the object.

If the first key is a literal
```elm
Json.Encode.object
  [ unknown
  , ( "b", b )
  ]
```

then we would only apply the optimization described above in [[#Skip List_fromArray]].

If some of the first keys are known

```elm
Json.Encode.object
  [ ( "a", a )
  , ( "b", b )
  , unknown
  , ( "c", c )
  ]
```

then we can transform those in an object literal, and iterate through the remaining keys as described in [[#Skip List_fromArray]].

```js
$elm$json$Json$Encode$object(
	_List_fromArray(
		[
			_Utils_Tuple2('a', a),
			_Utils_Tuple2('b', b)
		]
	)
)
// -->
$elm$json$Json$Encode$object$array(
	[
		_Utils_Tuple2('a', a),
		_Utils_Tuple2('b', b)
	],
	
)

var $elm$json$Json$Encode$object$array = function (pairs, initial) {
 return _Json_wrap(
  // Or a for-loop, whatever is faster
  pairs.reduce(function(_v0, obj) {
     var k = _v0.a;
     var v = _v0.b;
     return A3(_Json_addField, k, v, obj);
    }),
    initial
  );
};
```

Following this logic, we can make `$elm$json$Json$Encode$object$array` always take an initial JSON value, which if empty (as in the example of [[#Skip List_fromArray]]) would be `_Json_emptyObject(_Utils_Tuple0)` or more simply `{}`.

## Value encoding

We could remove some of the unnecessary wrappers like `Json.Encode.string`.
For these primitives, in production mode they are equivalent to `identity` (because `Elm.Kernel.Json.wrap` is just `identity` in optimized mode). So making sure that they equal `identity` and that `identity` gets removed where necessary would be enough to do the trick (either through us or through a minifier).

(Skipping the `Json.Encode.object` for brevity)
```elm
[ ( "a", Json.Encode.string "abc" ) ]
--> {a: "abc"}

[ ( "a", Json.Encode.int 1 ) ]
--> {a: 1}

[ ( "a", Json.Encode.float 1.1 ) ]
--> {a: 1.1}
```

## List & Array value encoding

We can likely simplify `Json.Encode.list`/`Json.Encode.array` as well, following the same ideas as for `Json.Encode.object`.

```elm
[ ( "a", Json.Encode.list f [a, b, c] ) ]
-- Currently:
-- {a: A2($elm$json$Json$Encode$list, f, _List_fromArray([a, b, c])) }
-- Proposed:
-- {a: [a, b, c].map(function(x) { return f(x); })}

-- Special case list for known primitive functions
[ ( "a", Json.Encode.list Json.Encode.string [a, b, c] ) ]
-- Currently:
-- {a: A2($elm$json$Json$Encode$list, $elm$json$Json$Encode$string, _List_fromArray([a, b, c])) }
-- Proposed:
-- {a: [a, b, c] }

-- Using Json.Encode.array with an `Array.fromList`
[ ( "a", Json.Encode.array f (Array.fromList x) ) ]
-- {a: [a, b, c].map(function(x) { return f(x); })}
```

## Questions

Can we apply these optimizations only with `--optimize`, or can we also apply them in development mode?
(I think we can but should be double checked)

## Applications

JSON encoding is used in several parts, so speeding this up would speed those as well:
- for communicating through JS ports
- setting properties on HTML elements (using `VirtualDom.property`, not sure how much this function is used)
- used to send data through HTTP