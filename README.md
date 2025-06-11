## Elm optimization ideas

Over the years, I have written down a number of ideas on how to make Elm code faster, and I figured these should be shared.

That way, these ideas can be discussed, tested or implemented, either for Elm or for similar languages.
Ultimately, the real discussion will have to be in the related repository (compiler or package).

If you have suggestions, let's discuss them in an issue or even a PR editing or adding a file.

Check out [`jfmengels/elm-benchmarks`](https://github.com/jfmengels/elm-benchmarks) for benchmarks of some ideas.

## Applying the ideas in Elm

Some ideas are on particular function implementations, and are ideally best solved through PRs
to [`elm/core`](https://github.com/elm/core) and other core packages,
or to [`elm-janitor`](https://github.com/elm-janitor/manifesto) ([`core`](https://github.com/elm-janitor/core)).

Other ideas are best applied in the compiler. For that, it's best solved through integrating the idea into
the [Lamdera](https://lamdera.com/) [compiler](https://github.com/lamdera/compiler)
which is [open-source](https://dashboard.lamdera.app/releases/open-source-compiler) and more actively developed.
