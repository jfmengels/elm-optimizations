[[Elm optimization techniques]]
A lot of functions are in practice equal to the `identity` function.

A lot of core functions (`String.fromChar`), and also type wrapper with a single constructor `type UserId = UserId String` in `--optimize` mode.

I *think* that the presence of `identity` prevents other optimizations and makes it hard to inline code. But I haven't done more research on the topic.

---

Code like this is necessary to prove to the compiler that something is of a given type, but at runtime becomes a cost.

```elm
List.map UserId userIdStrings
-->
A2(List$map, identity, userIdStrings)
--> Ideally we'd like to compile it to plainly
userIdStrings
```