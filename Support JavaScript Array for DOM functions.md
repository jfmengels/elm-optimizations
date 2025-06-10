[[Elm optimization techniques]]

Implementation PR in [Add supportArraysForHtml transformer by jfmengels · Pull Request #108 · mdgriffith/elm-optimize-level-2 · GitHub](https://github.com/mdgriffith/elm-optimize-level-2/pull/108)
## Goal

The goal is to avoid unnecessary Elm `List` to JavaScript `Array` conversions when it comes to `elm/html` functions.

Here is a small example:
```elm
Html.div
	[ Html.Attributes.class "some"
	, Html.Attributes.class "class"
	]
	[ a, b, c ]
```
gets compiled to
```js
A2(
  $elm$html$Html$div,
  _List_fromArray([
      $elm$html$Html$Attributes$class("some"),
      $elm$html$Html$Attributes$class("class")
  ]),
  _List_fromArray([a, b, c])
);
```
Through `_List_fromArray`, the attributes and children are converted from JavaScript `Array`s to Elm `List`s.

This conversion has a runtime cost, and Elm `List`s are slower to iterate through as well.

The idea behind this optimization is to remove these `_List_fromArray` calls and to keep the JavaScript `Array`s as such, and to have the underlying functions able to iterate through JavaScript `Array`s as well.

Taking the example from before, the result would end up being:
```js
A2(
  $elm$html$Html$div,
  [
      $elm$html$Html$Attributes$class("some"),
      $elm$html$Html$Attributes$class("class")
  ],
  [a, b, c]
);
```
This change will only be applied when the arguments are literal lists, not when they are variables or more complex expressions.
Further work could potentially increase the number of cases that we apply this change.
Elm Lists will have to be supported still, because this transformer will not be able to replace all attributes and children.

## Benchmark

[Benchmark](https://github.com/jfmengels/elm-benchmarks/blob/master/src/NativeJsArrayExploration/ArraysForHtml.elm)

The results indicate a 60% performance increase on Chrome, and a 50% increase on Firefox.

---

This could apply to more functions than only the DOM.