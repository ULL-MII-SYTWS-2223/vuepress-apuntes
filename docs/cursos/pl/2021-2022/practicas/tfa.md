---
title: "TFA: Final Project PL"
categories: ["temas", "practicas"]
sidebar: auto
key: tfa
published: true
date: 2022/05/06
delivery: "2022/05/26"
order: 15
layout: Practica
prev: /practicas/extended-egg-interpreter.md
template: 
---

## Introduction

* Any proposal related to what is seen in the course is welcome. Check with the teacher.
* The ideas proposed here are to extend the [Egg language](https://github.com/ULL-ESIT-PL-1819/egg)
but you can propose a TFA with another topic related to PL.
* An idea may be to extend Egg with a DSL with functionalities to facilitate the resolution of problems in a specific context that is of interest to the student. See the section [Strategy Pattern](#strategy-pattern-use) 


<!--
### Making Egg to wait for Async Operations

Una manera de simplificar todo el manejo de la asincronía en Egg
es modificar la forma en la que todo se evalúa: Cambiar  los métodos
`evaluate` para que sean funciones `async` y se haga un `await` en todas 
las llamadas a las evaluaciones (como una versión `asyncAwaitEvaluate` del `evaluate` que ya hemos visto). 

Vea como reescribimos nuestro anterior ejemplo de `fetch`:

```
[~/.../src/egg(async)]$ pwd -P
/Users/casiano/local/src/javascript/PLgrado/eloquentjsegg
[~/.../PLgrado/eloquentjsegg(async)]$ cat examples/fetch-2.egg
```
```ruby
do{
  :=(res, fetch("https://api.github.com/users/github")),
  :=(json, res.json()),
  print(json)
}
```

Veamos el resultado de una ejecución:

```
[~/.../PLgrado/eloquentjsegg(async)]$ bin/egg.js examples/fetch.egg
```
```js
{
  login: 'github',
  id: 9919,
  node_id: 'MDEyOk9yZ2FuaXphdGlvbjk5MTk=',
  avatar_url: 'https://avatars1.githubusercontent.com/u/9919?v=4',
  gravatar_id: '',
  url: 'https://api.github.com/users/github',
  ...
  created_at: '2008-05-11T04:37:31Z',
  updated_at: '2020-02-07T13:08:07Z'
}
```
-->
  
## Async and Await en Egg

### Promises in Egg

At this point the Egg machine can handle promises as
that it is possible in Egg to call the methods of JavaScript objects
and promises are nothing but JS Objects.

Suppose we extend Egg with a `fetch` object that implements the JS fetch API:

```js
topEnv['fetch'] = require('node-fetch');
```

In versions higher than 17 `fetch` is part of the Node.js core as an ESM module.

We can immediately write Egg programs like this:

```
[~/.../egg/crguezl-egg(private2019)]$ cat examples/fetch.egg
```
```js
➜  egg-oop-parser-solution git:(master) ✗ cat examples/fetch.egg 
(
  def(user, ||(process.argv[3], "crguezl")),
  print(user),
  def(userUrl, +("https://api.github.com/users/", user)),
  print(userUrl),
  fetch(userUrl)
    .then(->(res, res.json()))
    .then(->( json, 
      print(JSON.stringify(json,null,2))
    ))
)
```

Al ejecutarlo obtenemos:

```json
➜  egg-oop-parser-solution git:(master) ✗ bin/eggc.js examples/fetch.egg      
➜  egg-oop-parser-solution git:(master) ✗ npx evm examples/fetch.json esegredo
esegredo
https://api.github.com/users/esegredo
{
  "login": "esegredo",
  "id": 2370777,
  "node_id": "MDQ6VXNlcjIzNzA3Nzc=",
  "avatar_url": "https://avatars.githubusercontent.com/u/2370777?v=4",
  "gravatar_id": "",
  "url": "https://api.github.com/users/esegredo",
  "html_url": "https://github.com/esegredo",
  "followers_url": "https://api.github.com/users/esegredo/followers",
  "following_url": "https://api.github.com/users/esegredo/following{/other_user}",
  "gists_url": "https://api.github.com/users/esegredo/gists{/gist_id}",
  "starred_url": "https://api.github.com/users/esegredo/starred{/owner}{/repo}",
  "subscriptions_url": "https://api.github.com/users/esegredo/subscriptions",
  "organizations_url": "https://api.github.com/users/esegredo/orgs",
  "repos_url": "https://api.github.com/users/esegredo/repos",
  "events_url": "https://api.github.com/users/esegredo/events{/privacy}",
  "received_events_url": "https://api.github.com/users/esegredo/received_events",
  "type": "User",
  "site_admin": false,
  "name": "Eduardo Segredo",
  "company": "Universidad de La Laguna",
  "blog": "",
  "location": "San Cristóbal de La Laguna, Canary Islands, Spain",
  "email": null,
  "hireable": null,
  "bio": null,
  "twitter_username": null,
  "public_repos": 27,
  "public_gists": 0,
  "followers": 30,
  "following": 10,
  "created_at": "2012-09-18T14:31:50Z",
  "updated_at": "2022-04-26T10:57:58Z"
}
```

### Callbacks en Egg

Veamos un ejemplo de asíncronía en Egg con callbacks.
Extendamos Egg con un objeto que provee acceso al sistema de
archivos:

```js
topEnv['fs'] = require('fs');
```

Me he encontrado con algunos problemas cuando probé a escribir este programa:

```ruby
➜  egg-oop-parser-solution git:(master) ✗ cat examples/fs.egg 
(
  def(cb, fun(err, data, 
      if(==(err, null), print(data), print(err))
  )),
  fs.readFile("examples/no-existe.egg", "utf8", cb ),
  fs.readFile("examples/fs.egg", "utf8", cb) 
)
```

::: danger JS calls the callback with different number of arguments
The problem is that JS calls the callback
with a single argument `err` when an error occurs and with two
`(err, data)` when the operation succeeds.

This JS behavior causes that in some versions of the Egg compiler, the virtual machine  protests saying that *it expects the number of arguments to match the number of declared parameters*. Unfortunately, when there is an error JS calls the Egg-callback with a different number of arguments than it was declared with.
:::

The thing has several solutions, but I have opted for the fastest one, which has been that Egg does not protest against calls with a number of arguments less than those that were declared.

Another issue in this example is that in some versions, the Egg interpreter lacks the JS `null` object and
the convention is that JS calls the callback with `cb(null, data)` to indicate the absence of an error. 
Again there are numerous ways to approach this issue, but a simple one is to warn the Egg virtual machine of the existence of `null` so that it doesn't protest:

```js{1}
topEnv['null'] = null;
topEnv['true'] = true;
...
```

Here is an execution:

```js
➜  egg-oop-parser-solution git:(master) ✗ bin/egg examples/fs
{"errno":-2,"code":"ENOENT","syscall":"open","path":"examples/no-existe.egg"}
(
  def(cb, fun(err, data, 
      if(==(err, null), print(data), print(err))
  )),
  fs.readFile("examples/no-existe.egg", "utf8", cb ),
  fs.readFile("examples/fs.egg", "utf8", cb) 
)
```

### The Goals

To have **async** and **await** availables in Egg or to implement some alternative mechanisms.

An exploration I did in 2018 to achieve this goal can be found in repo [ULL-ESIT-PL-1819/private-egg branch async2019](https://github.com/ULL-ESIT-PL-1819/private-egg/tree/async2019) (private). The idea was to make async `evaluate`s (see [lib/ast.js](https://github.com/ULL-ESIT-PL-1819/private-egg/blob/async2019/lib/ast.js)) and produce an implicit `await` after each evaluation.

For instance, see [examples/fetch.egg](https://github.com/ULL-ESIT-PL-1819/private-egg/blob/async2019/examples/fetch.egg):

```ruby{3,5}
➜  eloquentjsegg git:(async2019) ✗ cat examples/fetch.egg 
do(
  def(userUrl, +("https://api.github.com/users/", process.argv[3])),
  print(userUrl),
  def(res, fetch(userUrl)),
  def(json, res.json()),
  print(json.name), 
  print(json.blog), 
  print(json.bio)
)
``` 

When executed produces:

```
➜  eloquentjsegg git:(async2019) ✗ bin/egg.js examples/fetch.egg crguezl
https://api.github.com/users/crguezl
Casiano Rodriguez-Leon
https://crguezl.github.io/
My Control Version Bio: Started with Unix RCS, then moved to  CVS, then Subversion and finally git 
```

However, this wasn't a satisfactory solution. I believe it will be much easier to do in a Egg to JS translator.

### Resources and References 

* [ULL-ESIT-PL-1819/private-egg branch async2019](https://github.com/ULL-ESIT-PL-1819/private-egg/tree/async2019) (private)
  * [lib/ast.js](https://github.com/ULL-ESIT-PL-1819/private-egg/blob/async2019/lib/ast.js)
  * [examples/fetch.egg](https://github.com/ULL-ESIT-PL-1819/private-egg/blob/async2019/examples/fetch.egg)
* [Book *The Modern Javascript Tutorial*. Chapter Promises, async/await](https://javascript.info/async)
* Vídeo [Cómo funciona Async/Await en menos de 15 minutos](https://youtu.be/u2axmPnxUoo) YouTube Vídeo por Appdelante

## Scope Analysis

Aunque el lenguaje  Egg dispone de ámbitos, los errores de ámbito (variables no declaradas) solo se detectan en tiempo de ejecución:

```ruby
[.../TFA-04-16-2020-03-22-00/davafons(casiano)]$ cat examples/set-error-compile.egg
set(x, 4)
```

Si lo ejecutamos nos da un  run-time error:

```
[.../TFA-04-16-2020-03-22-00/davafons(casiano)]$ bin/egg.js examples/set-error-compile.egg
ReferenceError: Tried setting an undefined variable: x
```

The point here is to detect errors as early as possible, 
before the program is executed.
This is done by walking the AST and looking for usage nodes of *words* that have not been defined in a higher scope:

```
[.../TFA-04-16-2020-03-22-00/davafons(casiano)]$ bin/egg.js -c examples/set-error-compile.egg
ReferenceError: Trying to use the undefined symbol x
```

In this variant of Egg the `-c` option used compiles the program but does not run it.

Some other types of usage errors can also be checked in this scope analysis phase.
We can extend Egg with declarations of the form `const(a,4)` for constants, and then we can walk the AST checking that no attempt is made to modify (`set(a, ...)`) that variable in its declaration scope.

### Referencias

* [Scope Analysis Notes]()
* Vea el Capítulo [Análisis del Contexto](/temas/scope-analysis/analisis-dependiente-del-contexto) en estos apuntes y lea el capítulo *Symbol Table Structure* y el Chapter 3 del libro de Muchnik:

## Translation: Egg to JS Compiler

The goal of this proposal is to implement a translator from Egg to JavaScript as is described in the section
[Scope Analysis and Translation: Translating From Egg to JavaScript](/temas/translation/egg-2-js) of this book/site/subject. This task implies the study and building of the  Scope Analysis stage. Make it as complete as possible.


## Lua Compiler

Using an interpreter for a large enough subset of the [Lua Grammar](https://github.com/kach/nearley/blob/master/examples/lua.ne) in Nearley.js by translating to Egg trees or directly to JS.

## Add Object Inheritance to Egg

It could be via a `child` method like this:

```ruby
➜  eloquentjsegg git:(develop2122) ✗ cat examples/child.egg 
do(
  def(x, object ( 
    "c", 0,
    "d", 4,
    "gc", ->( c ),
    "sc", ->(value, =(self.c, value)), # = never creates a new entry!
    "inc", ->(=(self.c, +(self.c, 1)))
  )),
  def(y, child(x)),
  y.sc(5),  
  y.inc(),
  print(y),   # {c: 6}
  print(y.d), # 4
  print(x)    # {c:0, d:4}
)
```

The `def(y, child(x))` declaration causes the `y` object to inherit the properties and methods of the `x` object.

Execution:

```
➜  eloquentjsegg git:(develop2122) ✗ bin/egg.js examples/child.egg 
{"c":6}
4
{"c":0,"d":4}
```

Notice how `self` references `y` when the code makes the calls `y.sc(5), y.inc()` setting its `c` to 6, while the  attribute `d` of `y` hast the value `4` since it is inherited from `x`.

Here is another example:

```ruby
➜  eloquentjsegg git:(develop2122) ✗ cat examples/child2.egg
do(
  def(x, object ( 
    "c", 0,
    "gc", ->( self.c ),
    "sc", ->(value, =(self.c, value)),
    "inc", ->(=(self.c, +(self.c, 1)))
  )),
  def(y, child(x)),
  y.sc(3),
  =(y.d, 4), # The attribute d only exists in y not in x
  y.inc(), y.inc(),
  console.log(y),
  console.log(x),
)
``` 

Execution:

```js
➜  eloquentjsegg git:(develop2122) ✗ bin/egg.js examples/child2.egg
{
  gc: [Function: jsFun] {
    numParams: 0,
    ast: Apply { type: 'apply', operator: [Object], args: [Array] }
  },
  sc: [Function: jsFun] {
    numParams: 1,
    ast: Apply { type: 'apply', operator: [Object], args: [Array] }
  },
  inc: [Function: jsFun] {
    numParams: 0,
    ast: Apply { type: 'apply', operator: [Object], args: [Array] }
  },
  c: 5,
  d: 4
}
{
  c: 0,
  gc: [Function: jsFun] {
    numParams: 0,
    ast: Apply { type: 'apply', operator: [Object], args: [Array] }
  },
  sc: [Function: jsFun] {
    numParams: 1,
    ast: Apply { type: 'apply', operator: [Object], args: [Array] }
  },
  inc: [Function: jsFun] {
    numParams: 0,
    ast: Apply { type: 'apply', operator: [Object], args: [Array] }
  }
}
```

In this case the assignment `=(y.d, 4)` creates the property `d` in the object `y` which does not exists in the parent object `x`

## Add Classes

Here is a proposal of syntax. Feel free to change it:

```ruby
do(
  class(Math, 
    {
      constructor: fun(x, y, 
        do(
          =(self.x, x),
          =(self.y, y)
        )
      ),
      methods: {
        sum: fun(+(self.x, self.y))
      },
      properties: { # class properties
        PI: 3.14159
      },
      extends: null # Optional. Reference to the parent class
    }
  ),
  do( /* main */
    def(a, Math.new(2,3)),
    print(a.sum()), # 5
    print(Math.PI)
  )
)
```

`Class` receives as arguments the **name** of the class and an object  describing the class:
 constructor and properties.

## Default values of function parameters

This extension consists of adding the possibility that the
last parameters of a function have default values and can be omitted in the call:

```js
do (
  def(f, fun(x, default(y, 3)), default(z, 2),
    do (
      print(x+y+z)
    )
  ),
  f(3),      # 8
  f(3, 5),   # 10
  f(3, 1, 9) # 13
)
```

You may find it helpful to read this tutorial [JavaScript Default Parameters](https://www.javascripttutorial.net/es6/javascript-default-parameters/)
if you decide to tackle this extension.

## Spread operator

Add a `spread` operator to Egg that works like the one in JS.
`spread(array)`  can be used in function calls where multiple elements are expected and vice versa: 
multiple arguments to a function can be placed in an array within the function body.

Follows an example:

```ruby
do (
  def(f1, fun(x, y, # f1 espera dos argumentos
    do (
      +(x,y)
    )
  )),
  def(z, array(1,4)),
  print(f1(spread(z))), # Lo llamamos con un array. Resultado: 5
  def(g, fun(a, spread(x), # g espera uno o mas argumentos
    do (
      +(x[0], x[1])
    )
  )),
  print(g(1, 4, 5)) # a es 1 y x es [4, 5]. Resultado: 9
)
```

## Improve Location Information and Run Time Errors

Compute the location information of the tokens (line, offset, starting point, etc.) for each node of the AST tree. Ideally, you can have stored  on each node where the associated code begins and where it ends. 

For example, given an AST like:

```
APPLY(op: W[n:if], args:ARRAY(W[n: true], V[v:4], V[V:5]]) # `if(true,4,5)
```

we compute a `loc` attribute for `APPLY` nodes and `PROPERTY` nodes with information about the start and end line and column of the `APPLY`. Take advantage of such information to improve runtime errors.


## AST Optimizations

### Folding Constants

It is about adding an optimization phase to the Egg compiler that does constant folding.

For example, when given a program like this as input:

```
[.../TFA-04-16-2020-03-22-00/davafons(casiano)]$ cat examples/optimize.egg
```
```ruby
do (
  :=(x, +(*(2, 3), -(5, 1))) # 2 * 3 + (5 - 1) == 10
)
```

Si se compila con la opción `--optimize` de lugar a un plegado de constantes (o en inglés [constant folding](https://en.wikipedia.org/wiki/Constant_folding))

```
[.../TFA-04-16-2020-03-22-00/davafons(casiano)]$ bin/egg.js --optimize -c examples/optimize.egg
```

El código resultante produce un programa equivalente a `:= (x, 10)`:

```
[.../TFA-04-16-2020-03-22-00/davafons(casiano)]$ cat examples/optimize.egg.evm
```
```js
{
  "type": "apply",
  "operator": {
    "type": "word",
    "name": "do"
  },
  "args": [
    {
      "type": "apply",
      "operator": {
        "type": "word",
        "name": ":="
      },
      "args": [
        {
          "type": "word",
          "name": "x"
        },
        {
          "type": "value",
          "value": 10
        }
      ]
    }
  ]
}
```

* [constant folding](https://en.wikipedia.org/wiki/Constant_folding) en la Wikipedia
* You can use [estraverse](https://github.com/estools/estraverse) to traverse the AST looking for constant trees
* 
See the book [Advanced Compiler Design Implementation](https://books.google.es/books/about/Advanced_Compiler_Design_Implementation.html?id=Pq7pHwG1_OkC&redir_esc=y) by Muchnick.


### Other Machine Independent Optimizations

Other possible optimizations are:

- [Loop invariant code motion](https://www.tuhh.de/es/esd/research/wcc/optimizations/loop-invariant-code-motion.html)
- [Constant Propagation](https://cran.r-project.org/web/packages/rco/vignettes/opt-constant-propagation.html)
- Dead code elimination

## Strategy Pattern: use 

The idea is to introduce a `use` function which is similar to `require`
but with the difference that it extends the `Egg-aluXX` language
using a library written in JavaScript.

That is, someone from the world wide, a programmer named Y excited about your language `Egg-aluXX`
extends it with a library called `egg-aluXX-tutu` which she publishes at [npm](http://npmjs.com).

She has done it by adding new features to `specialForms` and `topEnv`. 

She was able to do so because her module imports from the module in which you export the `specialForms` and `topEnv` hashes.

A statement like `use('tutu')` should cause the `egg` interpreter to do a `require` of `egg-aluXX-tutu` (which is assumed to have been previously installed in `node_modules/`) and the features exported by `egg-aluXX-tutu` are available to the Egg program.

Here is an skeleton of a plugin called `github.js`:

```js
let specialForms_ = new Map;
let topEnv_ = new Map;
const readFileSync = require('fs').readFileSync;
const path = require('path');
let accessToken;
/* ... load other libraries ... */

topEnv_['setToken'] = function (token_) {
  try {
    debugger;
    if (token_) {
        return accessToken = readFileSync(token_);
    }
    else if (accessToken = process.env["egggithubtoken"]) {
        return accessToken;
    }
    let eggConfig = require(
      path.join(
        require('os').homedir(),'.egg','config.json'
      )
    );
    return accessToken = eggConfig.github.token;
  } catch(e) {
        console.error(`
        Go to https://github.com/settings/tokens and get a token for
        GitHub Egg. Save it in the Egg config file 
        '~/.egg/config.json' as the entry 'github.token' 
        or in a file and provide its name calling 'setToken(fileName)'
        or save it in a environment variable with name 'egggithubtoken'
        `)
        process.exit(1);
    }
    return accessToken;
}

topEnv_['org'] = function (org_) {
  ... // request to GitHub API asking for org info 
}

topEnv_['whoami'] = function () {
  ...
}

topEnv_['members'] = function (org_) {
  ... // request to GitHub API
}

topEnv_['collaborators'] = function (org_) {
  ...
}

topEnv_['names'] = function (array) {
  ...
}

try {
  topEnv_['setToken']();
} catch(e) {
   // nothing
}
module.exports = {topEnv_, specialForms_};
```

Here is an example of use of `use` of the former lib  `github.js`

```ruby{15}
➜  egg-oop-parser-solution git:(master) ✗ cat examples/github.egg 
do(
    def(lib, 
      path.join(
        process.cwd(), # Execute from the root of this project
        "node_modules", 
        "@ull-esit-pl-2122,
        "eloquentjsegg",
        "lib",
        "github.js"
      )
    ),
    #print(lib),

    use(lib),   # Carga el módulo para trabajar con la Api de GitHub
   
    setToken("examples/.eggtoken"),  # Token Obtenido en la web de GitHub https://github.com/settings/tokens

    def(me, whoami()),
    print("Teacher: ",me.name),
    print("Teacher's blog:",me.blog),

    def(pl, org("ULL-ESIT-PL-2122")), 
    # print(pl),
    print("Total number of repos in ULL-ESIT-PL: ",pl.total_private_repos),
    print("Number of collaborators in ULL-ESIT-PL: ",pl.collaborators),

    def(membersPL, members(pl)),
    print("Total members in PL: ",membersPL.length),

    def(collaboratorsPL, collaborators(pl)), 
    print("Total collaborators in PL: ",collaboratorsPL.length),
    
    def(inside, membersPL.map(fun(cv, i, a, 
          [cv.login, cv.url]
        ) # end function
      ) # end map
    ), # end def
    print("First and last Members: ", inside[0], inside[-1])
)
```
## Types


Here is a proposal of syntax for introducing types in Egg:

```ruby
do(
  TYPE(STUDENT, OBJECT(name: STRING,id: STRING, mark: NUMBER)),
  TYPE(NUMLIST, ARRAY(NUMBER)),
  TYPE(NUMFUN, FUN(NUMBER, NUMBER)),
  NUMLIST(x, [4,7,9]),
  STUDENT(juan, {name: "Juan", id: "aluXX", mark: 10}),
  NUMFUN(f, fun(x, "hi!")), # error
  =(juan.name, 4),          # error
  =(juan.id, "alu123"),     # correct
  =(x[0], "not allowed"),   # error
)
``` 

* `TYPE`s and type names will be in uppercase
* The call to type `TYPE(X, ...) ` will produce a sort of *constructor of the type* that can be referred later in the program as `X(leftvalue, expression)`
* The special form `def` dissapears and is substituted by these type constructors
* Now we have to compute the type attribute of every AST node and check for type compatibility
* Ideally structural compatibility

To attempt this phase you must first have completed the scope analysis.

Scope analysis is the computation of the bindings between variable uses and variable declarations:

![](/images/type-analysis-binding-from-use-to-declaration.png)

Now both the symbol table and the ASTs will be decorated with the `type` attribute.
Then we traverse the AST computing the types of the nodes, their correct use and making amendments wherever it is necessary.

For instance, for each operator we know that the operands must have certain types, and that the result has a certain type. 

![](/images/type-analysis-computing-node-types.png)

## Translator from Egg AST to AST Term 

!!!include(includes/evmtoterm.md)!!!


## JSCodeshift, Putout and others

You can propose for the TFA tree transformations to help in code factorization using any of these tools:

* [JSCodeshift](/temas/tree-transformations/jscodeshift)
* [Putout](/temas/tree-transformations/putout)
* [ast-types](/temas/tree-transformations/ast-types)
* [codemods](/temas/tree-transformations/codemods)

## Experimental Language

Do the compiler explained in
* [Experimental Language: EXPL](https://silcnitc.github.io/) Write your own compiler!. An educational platform for compiler construction ([silcnitc](https://github.com/silcnitc))
