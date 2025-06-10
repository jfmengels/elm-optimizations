[[Elm optimization techniques]]

Inspired from the release in Gleam 1.11.

---
Extracted from [Gleam JavaScript gets 30% faster](https://gleam.run/news/gleam-javascript-gets-30-percent-faster/)

Gleam has a single flow control construct, the case expression. It runs top-to-bottom checking to see which of the given patterns match the value.

```gleam
pub fn greet(person: Person) -> String {
  case person {
    Teacher(students: [], ..) -> "Hello! No students today?"
    Student(name: "Daria", ..) -> "Hi Daria"
    Student(subject: "Physics", ..) -> "Don't be late for Physics"
    Teacher(name:, ..) | Student(name:, ..) -> "Hello, " <> name <> "!"
  }
}
```

Prior to this release, when compiling to JavaScript code results in `if else` chain, as can be seen in the code below.

```gleam
export function greet(person) {
  if (isTeacher(person) && isEmpty(person.students)) {
    return "Hello! No students today?";
  } else if (isStudent(person) && person.name === "Daria") {
    return "Hi Daria";
  } else if (isStudent(person) && person.subject === "Physics") {
    return "Don't be late for Physics";
  } else if (isTeacher(person)) {
    return "Hello, " + person.name + "!";
  } else {
    return "Hello, " + person.name + "!";
  }
}
```

_Disclaimer: This code has been lightly edited for clarity, but all aspects related to this implementation change remain the same._

This is very understandable and human looking code, but it's not as efficient as possible. For example, if the value is a `Student` with a name other than "Daria" and a subject other than "Physics" then the `isTeacher` and `isStudent` functions are each called twice, resulting in wasted work. Similar problems arise with other patterns, especially with the list type as it would need to be traversed multiple times to check various elements within them.

The new and improved approach is to transform the linear sequence of checks into a _decision tree_, where each check is performed the minimum number of times to find the matching clause as quickly as possible. This decision tree is then compiled into a series of nested `if else` statements in JavaScript.

```
export function greet(person) {
  if (isTeacher(person)) {
    if (isEmpty(person.students)) {
      return "Hello! No students today?";
    } else {
      return "Hello, " + person.name + "!";
    }
  } else {
    if (person.name === "Daria") {
      return "Hi Daria";
    } else {
      if (person.subject === "Physics") {
        return "Don't be late for Physics";
      } else {
        return "Hello, " + person.name + "!";
      }
    }
  }
}
```

This does result in a small increase in code size (up to 15% in our tests), but the uniform nature of the extra code is well suited to compression, with minification and brotli compression completely removing this extra size and producing the same application bundle size as with previous Gleam versions.

We have only implemented this optimisation for the JavaScript target and not the Erlang one as the Erlang Virtual machine implements this optimisation itself! It is done for us automatically there.

As part of this work the compilers analysis of pattern matching was enhanced, notably around bit-array patterns. It can now identify when a clause with a bit array pattern is unreachable because it only matches values that a previous clause also matches, such as the second clause here:

```gleam
case payload {
  <<first_byte, _:bits>> -> first_byte
  <<1, _:bits>> -> 1
  _ -> 0
}
```

Efficient compilation of pattern matching is a surprisingly challenging problem, and we would not have had so much success without academic research on the subject. In particular we would like to acknowledge ["How to compile pattern matching"](https://julesjacobs.com/notes/patternmatching/patternmatching.pdf), by Jules Jacobs and ["Efficient manipulation of binary data using pattern matching"](https://user.it.uu.se/~kostis/Papers/JFP_06.pdf), by Per Gustafsson and Konstantinos Sagonas. Thank you.

This is the culmination of work that was started before Gleam v1, and has been desired for much longer. A huge thank you to [Giacomo Cavalieri](https://github.com/giacomocavalieri) for this final piece.