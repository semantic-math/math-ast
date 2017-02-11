# math-ast
AST for symbolic math

# Motivation

The goal of this project is to produce a specification for an AST that can be
use to represent mathematical expressions and equations.

There are a number of symbolic math libraries for JavaScript available.
Unfortunately, there doesn't seem to be a good way for these efforts to
interoperate with each other.  They all include their own parsers and produce
their own ASTs.  This makes it difficult to swap out the parser if you want to
handle a different syntax, e.g. treating `abc` as a single identifier vs.
treating it as implicity multiplication between `a`, `b`, and `c`.  The ASTs
themselves usually include methods which makes manipulating the AST by third
party software difficult.

# Key Ideas

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
