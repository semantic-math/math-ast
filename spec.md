This document specifies the math AST.

- [Node objects](#node-objects)
- [Program](#program)
- [Relation](#relation)
- [Operation](#operation)
  - [Inverse](#inverse)
  - [Summation](#summation)
  - [Product](#product)
  - [Limit](#limit)
  - [Integral](#integral)
- [Identifier](#identifier)
- [Function](#function)
  - [Derivative](#derivative)
  - [PiecewiseFunction](#piecewisefunction)
- [Number](#number)
  - [ComplexNumber](#complexnumber)
  - [Quaternion](#quaternion)
- [Sequence](#sequence)
- [Group](#group)
  - [Parentheses](#parentheses)
  - [Interval](#interval)
  - [Tuple](#tuple)
  - [Vector](#vector)
  - [InnerProduct](#innerproduct)
  - [Set](#set)
- [Matrix](#matrix)
- [Ellipsis](#ellipsis)
- [Edge cases](#edge-cases)
- [Parser details](#parser-details)

# Node objects

Borrowed from https://github.com/estree/estree/blob/master/es5.md#node-objects.

All subsequent nodes inherit from this interface.

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
    op: string; // '+', '-', '*', '/', '\u00b7', '^', 'sum', 'prod', 'int', etc.
    args: [ Expression ];
    implicit: boolean;
    sub: Expression;
    sup: Expression;
}
```

Notes:
- `implicit` is only used for multiplication.
- `sub` and `sup` can be used together to indicate the lower/upper limits of an
  integral or the starting/stopping points of summation or product notation.
- It may make sense to flatten certain operations such as `'+'` and `'*'`.  An
  expression such as `1 + 2 + 3` has two different valid parse trees when using
  binary operations.  If we treat `'+'` as n-ary then it has only one valid
  parse tree.  The spec allows n-ary operations but does not require it.  The
  reason is that in certain circumstances you may want to represent `1 + 2 + 3`
  with binary operations to highlight that the expression can be evaluated in
  different orders.
- There may be situations in which it may make sense to have an operator on its
  own without any arguments, e.g. one-sided limits `lim_(x -> a^-)`.

## Inverse

Used to represent function inverses and inverse operations such as matrix
inverses.

```
interface Inverse <: Node {
    type: "Inverse";
    arg: Function | Identifier;
}
```

It can be derived from n `Operation` node under the following conditions:
- the `op` property is a `"^"`
- there are exactly two args
- the second arg is a `Number` whose value is `"-1"`

## Summation

TODO

## Product

TODO

## Limit

```
interface Limit <: Node {
    type: "Limit";
    expression: Expression;
    variable: Identifier;
    target: Expression;
}
```

## Integral

TODO
- include what is being variable is the expression being integrated by

# Identifier

`Identifier` stores information about variables, constants, and function names.
The reason not to have separate nodes for `Variable` and `Constant` is that an
identifier may be either depending on the context.

```
interface Identifier <: Node {
    type: 'Identifier';
    name: string;  // e.g. 'x', 'pi', 'atan2', etc.
    sub: null | Identifier | Number | Sequence;
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
    id: Identifier,
    args: [ Expression ]
}
```

Examples:
- `z = f(x, y) = x * y`
- `sin(pi / 2)`

## Derivative

```
interface Derivative <: {
    type: "Derivative",

}
derivatives: prime, Leibnitz,
```

## PiecewiseFunction

TODO

# Number

`Number` represents an element of the Reals.  This includes +/- infinity.

```
interface Number <: Node {
    type: "Number";
    value: string;
    exact: boolean;
    repeat: number;
}
```

Notes:
- The reason for storing the number as a string initial is order to avoid losing
  precision.  This may be of particular importance for science related
  applications.
- Repeating decimals can be represented by setting the `repeat` property of the
  number to a positive integer representing the number of digits at the end of
  the decimal that should be repeated.
- Non-repeating decimals cannot be represented exactly as numbers.  If you want
  to store one of these numbers in a `Number` node, the `exact` property Should
  be set to `false`.
- It may make sense to convert to some other representation, e.g. number,
  fraction.js instance, etc. before evaluating or transforming the AST.

## ComplexNumber

TODO

## Quaternion

TODO

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

# Group

A `Group` is a way to surround another expression or sequence of expressions
with delimiters such as parenthesis, square brackets, braces, etc.  It can be
refined into more specific nodes such as `Interal`, `Tuple`, `Vector`, or `Set`.

```
interface Group <: Node {
    type: "Group";
    left: string;  // '(' | '[' | '{', left floor, left triangular bracket, etc.
    right: string;  // ')' | ']' | '}', right floor, right triangular bracket, etc.
    content: Expression | Sequence;
}
```

Notes:
- think about how this could be used to support absolute value
- is there any situation which having a relation inside of delimiters makes sense

## Parentheses

```
interface Parentheses <: Node {
    type: "Parentheses";
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
interface Interval <: Node {
    type: "Interval";
    left: {
        value: Expression;
        type: "open" | "closed";
    };
    right: {
        value: Expression;
        type: "open" | "closed";
    };
}
```

It can be derived from a `Group` node under the following conditions:
- `content` was a `Sequence` with two args
- `left` and `right` delimiters are either parentheses or square brackets or
  some combination thereof.

Examples:
- `[0, Infinity)` half-closed interval from 0 to infinity.
- `[0, Infinity]` valid parse tree, would be rejected by semantic validation
- `(-1, 1)` open interval around 0

## Tuple

```
interface Tuple <: Node {
    type: "Tuple";
    content: [ Expression ];
}
```

Examples:
- `(5, 10)`
- `(0, 0, 1)`

## Vector

```
interface Vector <: Node {
    type: "Interval";
    delimiters: "round" | "square";
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

## InnerProduct

TODO

## Set

`Set` is an unordered grouping of objects.

```
interface Set <: Node {
    type: "Set";
    content: Node;
}
```

It can be derived from a `Group` node under the following conditions:
- `left` and `right` must both be braces.

TODO:
- What is contained within a `Set` could be anything, e.g. `{ names of fruit }`.
- What about `{ x | x > 5, x != 10 }`?

# Matrix

```
interface Matrix <: Node {
    type: "Matrix";
    delimiters: "round" | "square";
    rows: [ Sequence ];
}
```

Notes:
- all rows must be `Sequence`s of the same length

# Ellipsis

```
interface Ellipsis <: Node {
    type: "Ellipsis";
}
```

Examples:
- `S(n) = 1 + 2 + ... + n`
- `{ 1/2, 3/2, 5/2, ... }`

# Units

TODO

# Edge cases:
- `^-1` for function, trigonometric function, and matrix inverses will be parsed
  as an exponent on the function, e.g. `f^-1(x)` would be parsed like so:
```
{
    type: "Operation",
    op: "^",
    args: [
        {
            type: "Function",
            id: {
                type: "Identifier",
                name: "f"
            },
            args: [
                {
                    type: "Identifier",
                    name: "x"
                }
            ]
        },
        {
            type: "Number",
            value: "-1",
        }
    ]
}
```

# Parser details
A parser can choose to represent `sin^-1(x)` as either `1 / sin(x)` or as
`arcsin(x)`.  It can also choose to interpret `v^T` as `v` to the power `T` or
as the transpose of `v`.
