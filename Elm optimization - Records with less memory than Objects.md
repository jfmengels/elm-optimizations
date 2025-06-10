---
title: Records with less memory than Objects
updated: 2022-03-13 19:19:15Z
created: 2022-03-13 18:42:08Z
tags:
  - elm-performance
---
[[Elm optimization techniques]]

Note: Unfleshed idea, not tried or benchmarked.
## Idea

In Elm, we use records like `{ a = 1, b = 2 }` which translates to the JavaScript object literal `{ a : 1, b : 2 }`.
Objects have a lot of wasteful methods and properties attached to them, so it would be interesting to look at whether removing them (or ideally preventing them to be added in the first place) would improve memory usage and indirectly execution speed.

## Creating barebone records

Creating these doesn't seem to be as easy a one would expect.
The simplest mean is to do `Object.create(null)`, but you then need to add properties manually
```js
var obj = Object.create(null);
obj.a = 1;
obj.b = 2;
```

which is do-able, but not practical to use as an expression and also increases the code size.

## Explorations

### Prototype-less objects by default

Here is how to have `{}` create a close-to-barebone record by emptying out the object prototype: https://stackoverflow.com/a/36117912/2247266
But this would be the case for **everything**. Meaning it will probably break several things if you have any other JS code lying around.

AFAIK, there is no syntax to create prototype-less records.

What comes closest are Shadow Realms (ECMAScript proposal stage 2.7): https://github.com/tc39/proposal-shadowrealm
Explanation of the concept: https://github.com/tc39/proposal-shadowrealm/blob/main/explainer.md

We could have Elm code run in a shadow realm where records have no prototypes, but it's hard to tell whether that would cause performance decreases as I don't know how this would work in practice.

### Using Object.create(null)

You can actually use `Object.create(null)` to create a record like `{ a : 1, b : 2 }` by writing the following code:

```js
Object.create(null, { a : { value: 1 }, b : { value : 2 } }
```

This is a lot more verbose (with a potential impact on the bundle size) and might impact the creation time for the record.
To be benchmarked.