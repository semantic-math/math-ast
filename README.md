# math-ast
AST for symbolic math

# Motivation

The goal of this project is to enable software that manipulates math expressions
to interoperate.

There are a number of free symbolic math libraries for JavaScript.  Each has its
own syntax and more important its own AST.

Having a common AST is an ideal way to enable interoperability as evidenced by
Mozilla's Parser API and all of the tools that sprung up around it.

For symbolic math, a common AST would allow for different parsers, renderers,
solvers, and other tools to all interoperate with each other.

# Scope

The hope is to cover K-12 and undergrad math and science.

# Key Ideas

- AST is data only
- use similar node structure
- avoid one-off nodes
- any node type can be the root of the AST

# Parser Notes

- An AST produced by a parser must only be syntatically valid although a parser
  may choose to enforce some level semantic correctness as well.
- In the absense of semantic information, parsers are encouraged to generate
  the most general nodes, but are free to make their own assumptions.
- Parsers are free to define their own syntax.  One parser may decide to treat
  `abc` as a single identifier whereas another might treat it as implicit
  multiplication between `a`, `b`, and `c`.  Software that manipulate ASTs
  should ensure that they can handle multi-character strings for identifiers
  because this is what's in the spec.

# TODO

- A [reference parser](https://github.com/kevinbarabash/math-parser) (wip) that
  produces a math AST according to the spec.
- A formatter that accepts a math AST object and outputs TeX code.
- A set of helper functions:
  - isExpression, isProduct, isFraction, etc.
  - hasCommonDenominators
  - findCommonTerms
- A function to manipulate math ASTs by [replacing certain nodes](https://github.com/kevinbarabash/math-parser/blob/4b77bb327be493bfc50e58694b59357c97aec261/lib/replace.js).
- A set of useful transforms:
  - adding `0` is a no-op and can be revemoved
  - multiplication by `1` is a no-op and can be removed
  - intervals can't be closed when either end is `+/- infinity`
  - division by `0` is `undefined` or `+infinity` or `-infinity` depending on the
    exact situation.
  - raising `0` to the `0` power is undefined
- Semantic knowledge that can be used to determine whether a transform is valid:
  - addition/multiplication are commutative and associative for these operands
  - multplication does not commute for matrices
