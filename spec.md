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

# Key Ideas
- similar structure
- avoid one-off nodes
- ASTs should be data only
- parsers will produce most general nodes in the absense of semantic information
- semantic information can be used to refine general nodes to more specific ones
- different parsers can parse expressions differently to produce different, but
  equally valid AST, e.g. `xyz` could be parsed as a single identifier with the
  name `'xyz'` or implicit multiplication between three separate identifiers
  named `'x'`, `'y'`, and `'z'`.
- there is common semantic knowledge which is useful for K-12 which can be
  packaged into a set of transforms that can be applied to ASTs to make them
  more useful.

# Node objects

Borrowed from https://github.com/estree/estree/blob/master/es5.md#node-objects.

```
interface Node {
    type: string;
    loc: SourceLocation | null;
}
```

```
interface SourceLocation {
    source: string | null;
    start: Position;
    end: Position;
}
```

```
interface Position {
    line: number; // >= 1
    column: number; // >= 0
}
```

# Program

`Program` is a sequence of statements which can be any other node.  This is
useful in describe a set of steps to evaluate an expression, solve and equation,
or describe a proof.

```
interface Program <: Node {
    type: "Program";
    body: [ Node ];  // except Program
}
```

# Relation

`Relation` is used for equivalence relations, e.g. `=`, `<`, `<=`, etc., set
relations, e.g. "is a subset of", and any other relations that might come up.

```
interface Relation <: Node {
    type: "Relation";
    rel: string;  // '=', '<', '<=', etc.
    args: [ Expression ];
}
```
An expression is any `Node` that is neither a `Program` nor a `Relation`.

TODO:
- figure out subset, superset, etc.


# Operation

`Operation` handles unary, binary, and n-ary operations.  Unary minus is used
to represent negation.

```
interface Operation <: Node {
    type: "Operation";
    op: string; '+', '-', '*', '/', '\u00b7' (middle dot), '^', etc.
    implicit: boolean,
    args: [ Expression ];
}
```

Notes:
- It may make sense to flatten certain operations such as `'+'` and `'*'`.  An
  expression such as `1 + 2 + 3` has two different valid parse trees when using
  binary operations.  If we treat `'+'` as n-ary then it has only one valid
  parse tree.  The spec allows n-ary operations but does not require it.  The
  reason is that in certain circumstances you may want to represent `1 + 2 + 3`
  with binary operations to highlight that the expression can be evaluated in
  different orders.

# Identifier

`Identifier` stores information about variables, constants, and function names.
The reason not to have separate nodes for `Variable` and `Constant` is that an
identifier may be either depending on the context.

```
interface Identifier <: Node {
    type: 'Identifier',
    name: string;  // e.g. 'x', 'pi', 'atan2', etc.
    subscript: null | Identifier | Number | Sequence
}
```

Notes:
- `Sequence` is useful for matrix indices

# Function

`Function` can be used to represent in the definition of a function or in the
application of a function.

```
interface Function <: Node {
    type: "Function",
    label: Identifier,
    args: [ Expression ]
}
```

Examples:
- `z = f(x, y) = x * y`
- `sin(pi / 2)`

# Number

`Number`

```
interface Number <: Node {
    type: "Number";
    value: string;
}
```

Notes:
- The reason for storing the number as a string initial is order to avoid losing
  precision.  This may be of particular importance for science related
  applications.
- It may make sense to convert to some other representation, e.g. number,
  fraction.js instance, etc. before evaluating or transforming the AST.

# Sequence

`Sequence` comma separated sequence of non-program nodes

```
interface Sequence <: Node {
    type: "Sequence";
    sep: ',' | ';';
    args: [ Expression | Relation ];
}
```

Examples:
- `2x - y = 5, x + y = -1`

# Brackets

`Brackets` encompasses parentheses as well as other related symbols.  Can be
used for standard parenthesis, open/closed/half intervals, ordered tuples, sets
etc.

```
interface Brackets <: Node {
    type: "Brackets";
    left: string;  // '(' | '[' | '{', left floor, left triangular bracket, etc.
    right: string;  // ')' | ']' | '}', right floor, right triangular bracket, etc.
    content: Expression | [ Expression ];
}
```

Notes:
- think about changing `Brackets` to `Delimiters` to support absolute value
- is there any situation which having a relation inside of delimiters makes sense

## Parentheses

```
interface Parentheses <: Brackets {
    type: "Parentheses";
    left: '(';
    right: ')';
    content: Expression;
}
```

Examples:
- `2 * (3 + 4)`
- `x - (x + 1)`
- `(3 + 4)`
- `(5)`

## Interval

```
interface Interval <: Brackets {
    type: "Interval";
    left: '(' | '[';
    right: ')' | ']';
    content: [ Expression ];  // only two values
}
```

Examples:
- `[0, Infinity)` half-closed interval from 0 to infinity.
- `[0, Infinity]` valid parse tree, would be rejected by semantic validation
- `(-1, 1)` open interval around 0

## Tuple

```
interface Interval <: Brackets {
    type: "Interval";
    left: '(';
    right: ')';
    content: [ Expression ];
}
```

Examples:
- `(5, 10)`
- `(0, 0, 1)`

## Vector

```
interface Vector <: Brackets {
    type: "Interval";
    left: '[' | '(';
    right: ']' | ')';
    content: [ Expression ];
}
```

Examples:
- row vector: `[1, 2, 3]`
- column vector: `[1; 2; 3]`
- unit vector: `(0, 0, 1)`

TODO:
- Should we explicitly indicate whether a vector is a row or column vector?

Notes:
- You'll notice overlap between `Tuple` and `Vector`.  Which one application
  decideds to use is based on additional semantic information the application
  has about the context of the math statement.  If the content is geometry then
  it makes sense to us a `Tuple`.  If it's linear algebra then a `Vector` makes
  more sense.

## Set

```
interface Set <: Brackets {
    type: "Set";
    left: '{';
    right: '}';
    content: Node;
}
```

TODO:
- What is contained within a `Set` could be anything, e.g. `{ names of fruit }`.
- What about `{ x | x > 5, x != 10 }`?

# Ellipsis

```
interface Ellipsis <: Node {
    type: "Ellipsis";
}
```

Examples:
- `S(n) = 1 + 2 + ... + n`
- `{ 1/2, 3/2, 5/2, ... }`


# Edge cases:
- `^-1` for function, trigonometric function, and matrix inverses will be parsed
  as an exponent on the function.

Weird edge cases:
- `^-1` for function, trigonometric function, and matrix inverses
- `^T` for transpose
- ...

Stuff that hasn't been described but should be at some point:
- summation, product operators
- limits, derivatives, integrals
- matrices and column vectors
- ...
