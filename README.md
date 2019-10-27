# Composable Error Handling

Slides for an internal lightning talk at [N26](https://www.n26.com).

[Deck Markdown source](/src/main/ank/README.md).

To run it in a local environment using [reveal.js](https://github.com/hakimel/reveal.js/) execute:

```
cd deck
npm install
npm start
```

## Abstract

As Software Developers we solve complex problem. To solve them we first split them into smaller, simpler problems. Then we write code to solve the smaller problems. We combine those pieces of code to solve the larger problem. Ideally we aim to write 0 glue code to compose the code that solves the small problems into a program that solves the big one.

In this talk we will take the example of validation. It can be validating a form on the front end, validating input data on back end etc... We will explore how different validation strategies work in terms of error reporting, composition and ease of use. Along the way we will create a simple data type and a few utility functions. We will see that this nice home-grown solution works well and ticks all the boxes. Finally we move to a battle proven library.
