[[Elm optimization techniques]]

Rough idea:
Compile code like
```elm
a |> List.map f |> List.filter condition
```
into code that does as little iterations as possible.
```elm
List.foldr (\a acc -> 
```

This makes the code much faster, but could bloat up bundle size.
## Resources

[Deforestation (computer science) - Wikipedia](https://en.wikipedia.org/wiki/Deforestation_(computer_science)#:~:text=In%20the%20theory%20of%20programming,immediately%20consumed%20by%20a%20program)
https://en.wikipedia.org/wiki/Escape_analysis

[Deforestation: transforming programs to eliminate trees](https://www.sciencedirect.com/science/article/pii/030439759090147A?ref=pdf_download&fr=RR-2&rr=780b91e7a83b2a2c) (original paper on deforestation)

[A short cut to deforestation](https://users.cs.northwestern.edu/~robby/courses/395-495-2017-winter/deforestation-short-cut.pdf)


Static single assignment form: https://en.wikipedia.org/wiki/Static_single_assignment_form
- Something that Elm already has, with a list of optimizations that this enables that could be interesting to look at