An AST can be produced by input that is syntactic valid.  Operations and
functions may (or may not) be semantically valid depending on the operands.
Moreover, the operands may affect the meaning and or the properties of an
operation or function, e.g.
- `a * b` commutes if `a` and `b` are scalars whereas `A * B` does not
  commute if `A` and `B` matrices

Semantic knowledge should be kept out of the AST, but can be used with an AST to
for purposes of validating, evaluating, and transforming a mathematical statement.

A mathematical statement could be an expression, equation, or definition.

Multiple mathematical statements can be grouped into a linear sequence to form a
computation, manipulation, or proof.  These specific types of sequences do not
need to be explicitly modeled, only that there should be a way to describe a
sequence of statements.  Let's call this a `"Program"` for lack of a better term.

Some design goals for the nodes:
- regular, similar mathematical structures should have similar tree structure
- avoid one off nodes if possible

`Program` is a sequence of statements which can be any other node.  This
- type: `"Program"`
- body: an array of statements which could be any other node.

`Operation` handles unary, binary, and n-ary operations.  Unary minus is used
to represent negation.
- type: `"Operation"`
- op: a string containing the operation, e.g. `"+"`, `"*"`, `"/"`, `"\u00b7"`,
  `^`, etc.
- args: an array of any node other than `Relation` or whatever we call a sequence

`Relation` is used for equivalence relations, e.g. `=`, `<`, `<=`, etc., set
relations, e.g. "is a subset of", and any other relations that might come up
- type `"Relation"`
- rel: `=`, `<`, `<=`, etc. (TODO: figure out subset)
- args: an array of two or more nodes other than `Statement` or

`Identifier` stores information about variables, constants, and function names.
The reason not to have separate nodes for `Variable` and `Constant` is that an
identifier may be either depending on the context.
- type: `"Identifier"`
- name: string, e.g. `"x"`, `"pi"`, `"atan2"`, etc.
- subscript: `null`, `Identifier`, `Number`, or `Sequence` (sequence is useful
  for matrix indices)

`Function` can be used to represent either a function definition or function
application.
- type: `"Function"` (can be refined to `"FunctionDefinition"` or
  `"FunctionApplication"`)
- label: `Identifier`
- args: any node other than `Relation` or `Program`.

`Number`
- type: `"Number"`
- value: a string representation of the number.  It may make sense to convert to
some other representation, e.g. number, fraction.js instance, etc. before
evaluating or transforming the AST.

`Sequence` comma separated sequence of non-program nodes
- type: `"Sequence"`
- args: one or more non-program nodes, a `Sequence` of `Relation`s could
represent a system of equations, a sequence of numbers could represent an
ordered pair or a vector.

`Brackets` encompasses parentheses as well as other related symbols.  Can be
used for standard parenthesis, open/closed/half intervals, ordered tuples, sets
etc.
- type: `"Brackets"`
- left: string, either `"("`, `"["`, `"{"`, etc.
- right: string, either `")"`, `"]"`, `"}"`, etc.
- content: any non-program node (args: an array of non-program nodes might also
  make sense)

Weird edge cases:
- `^-1` for function, trigonometric function, and matrix inverses
- `^T` for transpose
- ...

Stuff that hasn't been described but should be at some point:
- summation, product operators
- ellipses to indicate an infinite sequence, series
- limits, derivatives, integrals
- matrices and column vectors
- ...