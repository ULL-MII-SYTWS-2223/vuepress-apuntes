---
title: Parsing with Nearley.js
clases: [ '2021/05/10', '2021/05/05', '2021/05/04', '2021/05/03', '2021/04/28' ]
---


## Introduction to Nearley.js

This section describes the nearley grammar language, in which you can describe
grammars for nearley to parse. Grammars are conventionally kept in `.ne` files.
You can then use `nearleyc` to compile your `.ne` grammars to JavaScript
modules.

You can find many examples of nearley grammars online, as well as some in the
`examples/` directory of the 
[Github repository](https://github.com/Hardmath123/nearley).

### Vocabulary

- A *terminal* is a single, constant string or a token. For example, the
  keyword `"if"` is a terminal.
- A *nonterminal* describes a set of possible strings. For example, all "if"
  statements can be described by a single nonterminal whose value depends on
  the condition and body of the if statement.
- A *rule* (or production rule) is a definition of a nonterminal. For example,
  
  ```
  ifStatement -> "if" condition "then" statement "endif"
  ```
  is the rule according to which the if statement nonterminal, `ifStatement`,
  is parsed. It depends on the nonterminals `condition` and `statement`. A
  nonterminal can be described by multiple rules. For example, we can add a
  second rule

  ```
  ifStatement -> "if" condition "then" statement "else" statement "endif"
  ```
  
  to support "else" clauses.

By default, nearley attempts to parse the first nonterminal defined in the
grammar. In the following grammar, nearley will try to parse input text as an
`expression`.

```
expression -> number "+" number
expression -> number "-" number
expression -> number "*" number
expression -> number "/" number
number -> [0-9]:+
```

You can use the pipe character `|` to separate alternative rules for a
nonterminal. In the example below, `expression` has four different rules.

```
expression ->
    number "+" number
  | number "-" number
  | number "*" number
  | number "/" number
```

::: warning Empty String 
The keyword `null` stands for the **epsilon rule** $\epsilon$, which matches nothing. The
following nonterminal matches zero or more `cow`s in a row, such as
`cowcowcow`:

```
a -> null | a "cow"
```
::: 

### Postprocessors

By default, nearley wraps everything matched by a rule into an array. For
example, when `rule -> "tick" "tock"` matches the string `"ticktock"`, it
creates the "parse tree" `["tick", "tock"]`. 

Most of the time, however, you
need to process that data in some way: for example, you may want to filter out
whitespace, or transform the results into a custom JavaScript object.

For this purpose, each rule can have a **postprocessor**: a JavaScript function
that transforms the array and returns a "processed" version of the result.
Postprocessors are wrapped in `{% %}`s:

```js
expression -> number "+" number {%
    function(data) {
        return {
            operator: "sum",
            leftOperand:  data[0],
            rightOperand: data[2] // data[1] is "+"
        };
    }
%}
number -> [0-9]:+ {% d => parseInt(d[0].join("")) %}
```

To compile it with nearley:

``` 
➜  examples git:(main) ✗ nearleyc postprocessors-example.ne -o postprocessors-example.js 
```

and we can execute the resulting parser with `nearley-test`:

```
➜  examples git:(main) ✗ nearley-test -i '5+10' postprocessors-example.js               
Table length: 5
Number of parses: 1
Parse Charts
Chart: 0
0: {expression →  ● number "+" number}, from: 0
1: {number →  ● number$ebnf$1}, from: 0
2: {number$ebnf$1 →  ● /[0-9]/}, from: 0
3: {number$ebnf$1 →  ● number$ebnf$1 /[0-9]/}, from: 0

Chart: 1
0: {number$ebnf$1 → /[0-9]/ ● }, from: 0
1: {number$ebnf$1 → number$ebnf$1 ● /[0-9]/}, from: 0
2: {number → number$ebnf$1 ● }, from: 0
3: {expression → number ● "+" number}, from: 0

Chart: 2
0: {expression → number "+" ● number}, from: 0
1: {number →  ● number$ebnf$1}, from: 2
2: {number$ebnf$1 →  ● /[0-9]/}, from: 2
3: {number$ebnf$1 →  ● number$ebnf$1 /[0-9]/}, from: 2

Chart: 3
0: {number$ebnf$1 → /[0-9]/ ● }, from: 2
1: {number$ebnf$1 → number$ebnf$1 ● /[0-9]/}, from: 2
2: {number → number$ebnf$1 ● }, from: 2
3: {expression → number "+" number ● }, from: 0

Chart: 4
0: {number$ebnf$1 → number$ebnf$1 /[0-9]/ ● }, from: 2
1: {number$ebnf$1 → number$ebnf$1 ● /[0-9]/}, from: 2
2: {number → number$ebnf$1 ● }, from: 2
3: {expression → number "+" number ● }, from: 0
```

The rules above will parse the string `5+10` into 

```
Parse results: 
[ { operator: 'sum', leftOperand: 5, rightOperand: 10 } ]
```

The postprocessor can be any function with signature 

```js 
function(data, location, reject)
```

Here,

- `data: Array` is an array that contains the results of parsing each part of
  the rule. Note that it is still an array, even if the rule only has one part!
  You can use the built-in `{% id %}` postprocessor to convert a one-item array
  into the item itself `d => d[0]`.

  For **arrow function** users, a convenient pattern is to decompose the `data`
  array within the argument of the arrow function:

  ```js
  expression ->
      number "+" number {% ([fst, _, snd]) => fst + snd %}
    | number "-" number {% ([fst, _, snd]) => fst - snd %}
    | number "*" number {% ([fst, _, snd]) => fst * snd %}
    | number "/" number {% ([fst, _, snd]) => fst / snd %}
  ```

- `location: number` is the index (zero-based) at which the rule match starts.
  You might use this to show the location of an expression in an error message.

  ::: warning Better forget
  Note: Many [tokenizers](tokenizers) provide line, column, and offset
  information in the Token object. If you are using a tokenizer, then it is
  better to use that information than the nearley-provided variable, which
  would only tell you that it saw the nth _token_ rather than the nth
  _character_ in the string.
  :::

- `reject: Object` is a unique object that you can return to signal that this
  rule doesn't *actually* match its input.

  Reject is used in some edge cases. For example, suppose you want sequences of
  letters to match variables, except for the keyword `if`. In this case, your
  rule may be

  ```js
  variable -> [a-z]:+ {%
      function(d,l, reject) {
          const name = d[0].join('');
          if (name === 'if') {
              return reject;
          } else {
              return { name };
          }
      }
  %}
  ```
  ::: warning Reject is slow 
  Grammars using `reject` are not context-free, and are often much
  slower to parse. 
  ::: 

nearley provides one built-in postprocessor:

- `id` returns the first element of the `data` array. This is useful to
  extract the content of a single-element array: `foo -> bar {% id %}`

### More syntax: tips and tricks

See [Comments and Charsets in Nearley.jss](nearley-comments-charsets)


#### EBNF: Repetitions and parenthesis

nearley supports the `*`, `?`, and `+` operators from
[EBNF](https://en.wikipedia.org/wiki/Extended_Backus–Naur_form) as shown:

```
batman -> "na":* "batman" # nananana...nanabatman
```

You can also use **capture groups with parentheses**. Its contents can be anything
that a rule can have:

```
banana -> "ba" ("na" {% id %} | "NA" {% id %}):+
```

### Macros

See section [Macros](macros)

### Additional JS

For more intricate postprocessors, or any other functionality you may need, you
can include chunks of JavaScript code between production rules by surrounding
it with `@{% ... %}`:

```js
@{%
const cowSays = require("./cow.js");
%}

cow -> "moo" {% ([moo]) => cowSays(moo) %}
```

Note that it doesn't matter where you add these; they all get hoisted to the
top of the generated code.

### Importing other grammars

See section [Importing other grammars](importing-grammars)

## Lexical Analysis with Moo 

* See section [Lexical Analysis with Moo](moo)

## Example

See [examples/calculator/arithmetic-lexer.ne](https://github.com/ULL-ESIT-PL/learning-nearley/blob/main/examples/calculator/arithmetic-lexer.ne)


```js
# How to Implement Lexical Analysis in Tools with a separated Lexer
# To use it run: 
# nearleyc arithmetic-lexer.ne -o grammar.js  && node use-arithmetic.js 'ln (3 + 2*(8/e - sin(pi/5)))'
# It parses valid calculator input
#   ln (3 + 2*(8/e - sin(pi/5)))
# is valid input.

@{%

const lexer = require('./lex.js');

const bin = (([x, op, y]) => op(x,y));
const Null = (d => null);
const fac = n => (n===0)?1:n*fac(n-1);
const unaryPost = (([p, op]) => op(p));
const funApply = ([fun, arg]) => fun(arg);
%}

@lexer lexer

main => null {% d => "" %} # Allow for empty lines
    | AS {% function(d) {return d[0]; } %}


# Addition and subtraction
AS -> AS PLUS MD  {% bin %}  # Prefer this syntax
    | AS MINUS MD {% bin %}
    | MD          {% id %}

# Multiplication and division
MD -> MD MULT E  {% bin %}
    | MD DIV E   {% bin %}
    | E          {% id %}

# Exponents
E -> F EXP E    {% bin %}
    | F         {% id %}

# Factorial 
F ->  P FACTORIAL    {% unaryPost %}
    | P              {% id %} 

# Fixed "bug" sinpi

P -> Q
    | FLOAT     {% id %}
    | SIN  Q    {% funApply %}
    | COS Q     {% funApply %}
    | TAN Q     {% funApply %}
    | ASIN Q    {% funApply %}
    | ACOS Q    {% funApply %}
    | ATAN Q    {% funApply %}
    | PI        {% id %}
    | EULER     {% id %}
    | SQRT Q    {% funApply %}
    | LN Q      {% funApply %}

# Parentheses
Q ->  LP AS RP  {% ([lp, as, rp]) => as %}


##### LEXICAL ANALYSIS #################################################

FLOAT -> %number  
PLUS -> "+"      {% function(d) {return ((a,b) => a+b); } %}
MINUS -> "-"     {% function(d) {return ((a,b) => a-b); } %}
MULT -> "*"      {% function(d) {return ((a,b) => a*b); } %}
DIV -> "/"       {% function(d) {return ((a,b) => a/b); } %}
EXP -> "^"       {% function(d) {return ((a,b) => Math.pow(a,b)); } %}
FACTORIAL -> "!"   {% d => fac %}
LP -> "("         {% Null %}
RP -> ")"         {% Null %}
SIN -> "sin"      {% d => Math.sin %}
COS -> "cos"      {% d => Math.cos %}
TAN -> "tan"      {% d => Math.tan %}
ASIN -> "asin"    {% d => Math.asin %}
ACOS -> "acos"    {% d => Math.acos %}
ATAN -> "atan"    {% d => Math.atan %}
PI -> %pi         {% d => Math.PI %}
EULER -> %e       {% d => Math.E  %}
SQRT -> %sqrt     {% d => Math.sqrt %}
LN -> %ln         {% d => Math.log %}
```

### Lexical Analyzer 

See [examples/calculator/lex.js](https://github.com/ULL-ESIT-PL/learning-nearley/blob/main/examples/calculator/lex.js)

```js
const { makeLexer, moo } = require("moo-ignore");

const Tokens = {
  space: { match: /\s+/, lineBreaks: true },
  number: { match: /\d+(?:\.\d+)?\b/, value: x => Number(x) },
  "+": "+",
  "-": "-",
  "*": "*",
  "/": "/",
  "^": "^",
  "!": "!",
  "(": "(",
  ")": ")",
  id: {
    match: /[a-z_][a-z_0-9]*/,
    type: moo.keywords({
      sin: "sin",
      cos: "cos",
      tan: "tan",
      asin: "asin",
      acos: "acos",
      atan: "atan",
      pi: "pi",
      e: "e",
      sqrt: "sqrt",
      ln: "ln"
    }),
  },
};

let lexer = makeLexer(Tokens, ["space"]);

module.exports = lexer;
```

### main

See [examples/calculator/use-arithmetic.js](https://github.com/ULL-ESIT-PL/learning-nearley/blob/main/examples/calculator/use-arithmetic.js)

```js
const nearley = require("nearley");
const grammar = require("./grammar.js");

let s = process.argv[2] || ` 4 * pi / e`;

try {
  const parser = new nearley.Parser(nearley.Grammar.fromCompiled(grammar));
  parser.feed(s);
  console.log(parser.results[0]) 
} catch (e) {
    console.log(e);
}
```

### Execution 

`nearleyc arithmetic-lexer.ne -o grammar.js && node use-arithmetic.js`


## Performance of NearleyJS

This study by Toby Ho shows a case in which NearleyJS is 200 times slower than a Recursive Descendant Functional parser :disappointed:.

<youtube id="njWmVljrhEE"></youtube> 

## The Earley Algorithm

See the section [The Earley Algorithm Explained](algorithm)

## References

* [Repo ULL-ESIT-PL/learning-nearley](https://github.com/ULL-ESIT-PL/learning-nearley/)

* [Nearley.JS Home Page](https://nearley.js.org/)
* [Nearley.JS GitHub repo](https://github.com/kach/nearley) is a JS parser generator using the Early algorithm
  * [Nearley.js Grammar in Nearley.js](https://github.com/kach/nearley/blob/master/lib/nearley-language-bootstrapped.ne)

* [Toby Ho Course on How to Make a Parser with Nearley.JS](toby-ho-course)
* [Tokenizers for nearley.js](https://nearley.js.org/docs/tokenizers)
* [fun-lang in Nearley.js](https://github.com/airportyh/fun-lang) GitHub Repo by Toby Ho
  * [scripts](https://github.com/airportyh/fun-lang/tree/master/scripts) folder: How to run the compiler
  * Toby Ho has published a more advanced version of the language as a npm package:  [play-lang](https://www.npmjs.com/package/play-lang)
* [Parsing in JavaScript: Tools and Libraries](https://tomassetti.me/parsing-in-javascript/)

### moo

* [moo](https://www.npmjs.com/package/moo/)
* [moo-ignore](https://www.npmjs.com/package/moo-ignore)

