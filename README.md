# ppx_parser
`ppx_parser` is an `OCaml` ppx extension that lets you write parsers *à la* `Camlp4` parser notation. It can be used as a replacement for projects that still use Camlp4 stream parsers. For new projects, it is advised to use lexer and parser generators.

## Example
Following `Camlp4` stream parser
```ocaml
type tok = Int of int | True | False | If | Then | Else | Let | In | Equal | Ident of string

type expr = IntLit of int | BoolLit of bool | IfThenExpr of expr * expr * expr | LetInExpr of expr * expr

let rec parse_expr = parser
    | [< 'If; i = expr; 'Then; t = expr; 'Else; e = expr >] -> IfThen (i, t, e)
    | [< 'Let; 'Ident x; 'Equal; e = expr; 'In; i = expr >] -> LetIn (l, i)
    | [< 'Int i >] -> IntLit i
    | [< 'True >] -> BoolLit true
    | [< 'False >] -> BoolLit false
```
can be written as
```ocaml
let rec parse_expr = function%parser
    | [ If; expr as i; Then; expr as t; Else; expr as e] -> IfThen (i, t, e)
    | [ Let; Ident x; Equal; expr as e; In; expr as i] -> Let (l, i)
    | [Int i] -> IntLit i
    | [True] -> BoolLit true
    | [False] -> BoolLit false
```
## Installation
`ppx_parser` can be installed with [opam](https://opam.ocaml.org/):
```
$ opam install ppx_parser
```
Add the following field to your `library`, `executable` or `test` stanza in your `dune` file:
```
(preprocess (pps ppx_parser))
```
## Usage
### `%parser`
The `%parser` extension can be used to mark a `function` as a stream parser. Each function case represents a sequence of stream elements that have to be matched in order to produce the result on the right-hand side:
```ocaml
let stupid_int_parser = function%parser
    | [1; 2; 3; 4] -> "1, 2, 3, 4"
    | [3; 2; 1] -> "3, 2, 1"
    | [] -> ""

let stream = Stream.of_list [3; 2; 1]

let result = stupid_int_parser stream
(* "3, 2, 1" *)
```
It does so by peeking at the first element of the stream.
The function case where the first element of the list matches the peeked element is selected (`[3; 2; 1]` in the example). A `Stream.Error "Parse exception."` is thrown if the remainder of the selected list case does not match the remainder of the stream. Whenever an element is matched, it is removed from the stream.

If no function case is found where the first element of the list matches the first element of the stream, a `Stream.Failure` is raised.

### Empty list `[]`
The empty list can be used to produce a result without removing anything from the stream. This means that the stream can also be empty without raising `Stream.Failure` because nothing is removed from the stream:
```ocaml
let no_match_parser = function%parser
    | [1; 2; 3] -> "1, 2, 3"
    | [] -> "no match"

let stream = Stream.of_list [5; 6]

let result = no_match_parser stream
(* "no match" *)

(* stream remains [5; 6] because nothing was removed *)
```
### Binding variables
#### `%let` or `%l`
Extension `%let` (shorthand `%l`) can be used inside a parser function list case to bind variables:
```ocaml
type tok = Int of int | Add | Sub

let rec parse_op lhs = function% parser
    | [Add; [%let rhs = parse_expr]] -> lhs + rhs
    | [Sub; [%let rhs = parse_expr]] -> lhs - rhs
    | [] -> lhs

and parse_expr = function%parser
    | [Int i; [%let op = parse_op i]] -> op

let stream = Stream.of_list [Int 1; Add; Int 3; Sub; Int 2]

let result = parse_expr stream
(* 2 *)
```
#### ... `as` ...
An alias can be used when the right-hand sing of the binding is a simple function name:
```ocaml
let rec parse_op lhs = function% parser
    | [Add; parse_expr as rhs] -> lhs + rhs
    | [Sub; parse_expr as rhs] -> lhs - rhs
    | [] -> lhs

and parse_expr = function%parser
    | [Int i; [%let op = parse_op i]] -> op
```
#### `let` ... `in` ...
`let` expressions can be used inside `%let` extensions:
```ocaml
let rec parse_expr = function%parser
    | [Int i; [%let op =
        let parse_op lhs = function%parser
            | [Add; [%let rhs = parse_expr]] -> lhs + rhs
            | [Sub; [%let rhs = parse_expr]] -> lhs - rhs
            | [] -> lhs
        in
        parse_op i
    ]] -> op
```