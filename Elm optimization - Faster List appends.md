[[Elm optimization techniques]]

In [List module performance improvements by robinheghan · Pull Request #1027](https://github.com/elm/core/pull/1027/files), `++` is sped up by calling the underlying function for `List.append`.

The PR also speeds up `List.append`, although it looks fairly equivalent to https://github.com/jfmengels/core/commit/0b8914eccb59bf13e52f18103b9f6773213a9bbd compiled with TRMC. We should benchmark them.