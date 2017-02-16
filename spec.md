This document specifies the math AST.  The syntax use is this document is not
part of the spec.  Parser providers should feel free to define whatever syntax
they think is useful.

- [Node objects](#node-objects)
- [Program](#program)
- [Relation](#relation)
- [Operation](#operation)
  - [Inverse](#inverse)
  - [BoundedOperation](#boundedoperation)
  - [Limit](#limit)
  - [Integral](#integral)
- [Identifier](#identifier)
- [Function](#function)
  - [Derivative](#derivative)
  - [PiecewiseFunction](#piecewisefunction)
- [Number](#number)
  - [Unit](#unit)
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

# Unique Identifiers

- case sensitive
- always lowercase
- no spaces or special characters, e.g. `-` or `_`
- can't start with a number but may include numbers

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

`Relation` is used for equivalence relations and set relations.  Each relation
is identified by a short unique identifier.  We've used HTML entities without
the leading `&` and trailing `;`.  In some case, such as 'equals to', the HTML
entity is a number, i.e. `&61;`.  This doesn't provide any clue as to what the
relation is.  In cases like these we've picked a suitable alternative'.

```
interface Relation <: Node {
    type: "Relation";
    rel: 'eq' | Inequality | SetRelation;
    args: [ Expression ];
}
```

An `Expression` is any `Node` that is neither a `Program` nor a `Relation`.

TODO: be more accurate about Expression.  A `Unit` on its own is not an
`Expression`.  What about an `Interval`?

```
enum Inequality {
    'lt'    // less than
  | 'le'    // less than or equal to
  | 'gt'    // greater than
  | 'ge'    // greater than or equal to
  | 'ne'    // not equal to
  | 'cong'  // congruent to
  | 'equiv' // equivalent to
  | 'prop'  // proportional to
}
```

```
enum SetRelation {
    'isin'  // an element of
  | 'notin' // not an element of
  | 'sub'   // strict subset
  | 'sube'  // subset
  | 'sup'   // strict superset
  | 'supe'  // superset
}
```

# Operation

`Operation` handles unary, binary, and n-ary operations.  Only

```
interface Operation <: Node {
    type: "Operation";
    op: BasicOperator | LogicOperator | SetOperator | VectorOperator;
    args: [ Expression ];
    implicit: boolean;
    subscript: Expression;
    superscript: Expression;
}
```

```
enum BasicOperator {
    'add'   // addition (n-ary)
  | 'neg'   // negation (unary)
  | 'mul'   // multiplication (n-ary)
  | 'div'   // division (binary)
  | 'pow'   // power (binary)
}
```

```
enum LogicOperator {
    'and'   // and (n-ary)
  | 'or'    // or (n-ary)
  | 'not'   // not (unary)
  | 'xor'   // exclusive or (n-ary)
  | 'imp'   // implication (n-ary)
  | 'iff'   // bidirection implication (n-ary)
}
```

```
enum SetOperator {
    'cap'   // intersection (n-ary)
  | 'cup'   // union (n-ary)
  | 'diff'  // set difference
}
```

TODO:
- How to represent `\pm` operator?
- How to represent `1 - 2` vs `1 + -2`?

Notes:
- `implicit` is only used for multiplication.
- The reason why we use string enums for the operators instead of traditional
  strings such as `'+'`, `'-'`, etc. is that there are multiple strings for the
  same operator and the same symbol can represent different operations.
- Syntax may not be enough to determine whether to use one operator versus
  another, e.g. the center dot can represent multiplication or the dot product.
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

## BoundedOperation

```
interface BoundedOperation <: Node {
    type: "BoundedOperation";
    op: BoundedOperator;
    bounds: [ Expression ];
    variable: Identifier;
}
```

```
enum BoundedOperator {
    'sum'   // summation (unary)
  | 'prod'  // product (unary)
  | 'int'   // integral (unary)
  | 'cap'   // intesection (unary)
  | 'cup'   // union (unary)
}
```

Examples:
- `Int_0^1(x^2 dx)`
  ```
  {
      type: "BoundedOperation",
      op: "int",
      bounds: [
          { type: "Number", value: "0" },
          { type: "Number", value: "1" },
      ],
      variable: { type: "Identifier", name: "x" }
  }
  ```

Notes:
- `bounds` must have two values and only two values.

TODO:
- handle bounds such as `x in S` where S is a set of sets.

## Limit

```
interface Limit <: Node {
    type: "Limit";
    arg: Expression;
    variable: Identifier;
    target: Expression;
}
```

# Identifier

`Identifier` stores information about variables, constants, and function names.
The reason not to have separate nodes for `Variable` and `Constant` is that an
identifier may be either depending on the context.

```
interface Identifier <: Node {
    type: 'Identifier';
    name: string;  // e.g. 'x', 'pi', 'atan2', etc.
    subscript: null | Identifier | Number | Sequence;
}
```

Notes:
- `Sequence` is useful for matrix indices

TODO:
- How to specify the type of the identifier, e.g. vector, angle, etc.

# Function

`Function` can be used to represent in the definition of a function or in the
application of a function.

```
interface Function <: Node {
    type: "Function";
    id: Identifier;
    args: [ Expression ];
}
```

Notes:
- localized functions such as `tg` should be store as `tan` but rendered as
  `tg` for any user facing purposes.

Examples:
- `z = f(x, y) = x * y`
- `sin(pi / 2)`

TODO:
- have a list of special functions, e.g. trig, log, ln, etc.
- C(n, k) = n choose k
- P(n, k) = n permute k
- log(base, value)
- root(arg, index)

## Derivative

A `Derivative` node describes taking the derivative of a function or expression.
The `variables` refer to which variables the the derivative is being taken with
respect to.  If it's a second derivative with respect to `x`, then `x` will
appear twice in the `variables` array.

```
interface Derivative <: {
    type: "Derivative";
    arg: Function | Expression;
    variables: [ Identifier ];
}
```

TODO:
- How do we represent evaluation of `f'` at a certain value?

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
    unit: null | Unit;
}
```

Examples:
- The repeating decimal form of 122/99 is representated as
  ```
  {
       type: "Number",
       value: "1.23",
       exact: true,
       repeat: 2,
       unit: null
  }
  ```

Notes:
- The reason for storing the number as a string initial is order to avoid losing
  precision.  This may be of particular importance for science related
  applications.
- Non-repeating decimals cannot be represented exactly as numbers.  If you want
  to store one of these numbers in a `Number` node, the `exact` property Should
  be set to `false`.
- It may make sense to convert to some other representation, e.g. number,
  fraction.js instance, etc. before evaluating or transforming the AST.

## Unit

A `Unit` is either a `CompoundUnit` or a `SimpleUnit`.

```
interface CompoundUnit <: Node {
    type: "CompoundUnit";
    op: 'multiply' | 'divide' | 'power';
    args: [ SimpleUnit | CompoundUnit | Number ];
}
```

```
interface SimpleUnit <: Node {
    type: "SimpleUnit";
    name: "m" | "g" | "s" | "ft" | "in" | "V" | "rad" | "deg" | ...;
    prefix: "M" | "k" | "c" | "m" | ...;
}
```

Note:
- If the `operation` is `power` then the second arg must be a `Number` and more
  specifically a non-zero integer.  Otherwise, it must be either a `BaseUnit` or
  `CompoundUnit`.
- Not all prefixes are used with all units, e.g. `kft` is invalid.

## ComplexNumber

```
interface ComplexNumber <: Node {
    type: "ComplexNumber";
    real: null | Number | Identifier;
    imag: null | Number | Identifier;
}
```

Notes:
- If both `real` and `imag` are `null`, the node is invalid.

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
    left: LeftDelimiter;
    right: RightDelimiter;
    content: Expression | Sequence;
}
```

```
enum LeftDelimiter {
    '(' | '[' | '{' | 'lfloor' | 'lciel' | 'langle'
}
```

```
enum RightDelimiter {
    ')' | ']' | '}' | 'rfloor' | 'rciel' | 'rangle'
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

TODO:
- maybe `leftDelim`, `leftValue`, `rightDelim`, and `rightValue`?

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
- Explicitly indicate whether a vector is a row or column vector?
- Indicate if it's a unit vector or not?

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

The root of the AST can be any type of node.  This avoids a needless deep AST,
but it does me that code that process math ASTs should be able to deal with
this.

A parser does not have to produce all nodes.  Some parsers may choose to parse
`m/s` as a unit while others may parse it as division.
