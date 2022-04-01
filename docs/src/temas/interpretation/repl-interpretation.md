# Construcción de un Repeat Evaluate Print Loop

En la práctica [egg-interpreter](/practicas/egg-interpreter/) deberá hacer  que el ejecutable `egg` funcione como un bucle REPL cuando no se le proporciona un fichero de entrada. 


```js
[~/ull-pl1718-campus-virtual/tema3-analisis-sintactico/src/egg/crguezl-egg(private)]$ bin/egg.js
> def(x, array(1,2,array(3,4))) # x = [1,2,[3,4]]
[ 1, 2, [ 3, 4 ] ]
> <-(x,2) # 3d element
[ 3, 4 ]
> <-(x,0) # 1st element
1
> +(2, # No completamos la expresión: espera
> ..*(3, # Sigue incompleta
> ....4)
> ..) # cerramos: evalúa
14
> # Pulsamos CTRL-D
> goodbye!
```

En este [Vídeo *Programando un bucle REPL para el lenguaje Egg*](https://youtu.be/5gIlt6r29lw) explicamos como hacerlo

<youtube id="5gIlt6r29lw"></youtube>

The function `parBalance(line)` that we introduce in the `parse.js` main file checks for parenthesis balance in `line`. The parameter `line` contains a (potentially incomplete)
egg expression:


```js
'use strict';
const { SPACE } = require("./tokens.js");
const fs = require('fs');
const nearley = require("nearley");
const grammar = require("./grammar.js");
const lex = require('./lex-pl.js');

function parBalance(line) {
  let stack = 0;
  lex.reset(line);
  let tokens = lex.tokens;
  // Increment stack each time you see an LP token, decrement it if you see a RP 
  /* ... fill the code ... */
  return stack;
}
```

Fell free to add and export in file `parse.js`  any convenience parser-related functions you feel you neeed in the `egg-interpreter` module.
