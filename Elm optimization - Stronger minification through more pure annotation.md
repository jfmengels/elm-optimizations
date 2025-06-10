---
title: Stronger minification
updated: 2022-08-10 10:14:53Z
created: 2022-08-10 08:18:57Z
tags:
  - elm-performance
---
[[Elm optimization techniques]]

Try online: https://skalman.github.io/UglifyJS-online/


Elm recommends minifying code using this command:

```bash
uglifyjs $js --compress 'pure_funcs=[F2,F3,F4,F5,F6,F7,F8,F9,A2,A3,A4,A5,A6,A7,A8,A9],pure_getters,keep_fargs=false,unsafe_comps,unsafe' | uglifyjs --mangle --output $min
```

It contains a list of pure functions.
Because every Elm function is pure, so we could potentially add them ALL to this list. But I don't know what the consequences of that would be.
Maybe eol2 could gather that list and then call your favorite minifier for you with that huge list?

---

All expressions called through Ax are considered pure (because of the annotation above). With [[Elm optimization - Direct function calls]] removing the Ax wrappers, less expressions will be considered pure and be automatically removed by minifiers when considered unnecessary.


---

Resources on existing inline comments: https://medium.com/@LihauTan/dead-code-elimination-hint-uglify-js-that-your-function-is-pure-208efb251c7c
only for assignments though, it doesn't work for annotating a function. I created https://github.com/mishoo/UglifyJS/issues/5614 to discuss that

In practice, for Elm, this would likely only remove unused let declarations, which would not happen often because `elm-review` already does that.