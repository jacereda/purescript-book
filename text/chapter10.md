# Interfaz para funciones externas (*foreign function interface*)

## Objetivos del capítulo

Este capítulo presentará la _interfaz para funciones externas_ (o _FFI_), que permite la comunicación desde el código PureScript al código JavaScript y viceversa. Cubriremos lo siguiente:

- Cómo llamar funciones JavaScript puras desde PureScript
- Cómo crear nuevos tipos de efecto y acciones para usar con la mónada `Eff` basadas en código JavaScript existente
- Cómo llamar código PureScript desde JavaScript
- Cómo entender la representación de valores PureScript en tiempo de ejecución
- Cómo trabajar con datos no tipados usando el paquete `purescript-foreign`

Al final del capítulo retomaremos nuestro ejemplo de la agenda. El objetivo del capítulo será añadir la siguiente funcionalidad nueva a nuestra aplicación usando la FFI:

- Alertar al usuario con una notificación emergente
- Almacenar los datos del formulario serializados en el almacenamiento local del navegador y recargarlos cuando la aplicación reinicia

## Preparación del proyecto

El código fuente para este módulo es una continuación del código fuente de los capítulos 3, 7 y 8. Como tal, el árbol de fuentes incluye los ficheros fuente apropiados de dichos capítulos.

Este capítulo añade dos nuevas dependencias Bower:

1. La biblioteca `purescript-foreign` que proporciona un tipo de datos y funciones para trabajar con _datos no tipados_.
1. La biblioteca `purescript-foreign-generic`, que añade soporte a la biblioteca `purescript-foreign` para _programación sobre tipos de datos genéricos_.

_Nota_: para evitar problemas concretos del navegador con el almacenamiento local al servir la página desde un fichero local, puede ser necesario ejecutar el proyecto de este capítulo por HTTP:

## Una advertencia

PureScript proporciona una interfaz para funciones externas sencilla para hacer que trabajar con JavaScript sea tan simple como sea posible. Sin embargo, hay que hacer notar que la FFI es una capacidad _avanzada_ del lenguaje. Para usarla de manera segura y efectiva, debes tener un conocimiento de la representación en tiempo de ejecución de los datos con los que planeas trabajar. Este capítulo intenta impartir dicho conocimiento.

La FFI de PureScript está diseñado para ser muy flexible. En la práctica esto significa que los desarrolladores pueden elegir entre dar a sus funciones externas tipos muy simples o usar el sistema de tipos como protección frente a usos incorrectos accidentales del código extreno. El código de las bibliotecas estándar tiende a favorecer el segundo enfoque.

Como ejemplo simple, una función JavaScript no garantiza que su valor de retorno no será `null`. De hecho, ¡el código JavaScript idiomático devuelve `null` con bastante frecuencia! Sin embargo, normalmente no hay un valor null en PureScript. Así, es la responsabilidad del desarrolador gestionar estos casos límite de forma apropiada cuando diseñe sus interfaces a código JavaScript usando la FFI.

## Llamando a PureScript desde JavaScript

Llamar una función PureScript desde JavaScript es muy simple, al menos para funciones con tipos simples:

Tomemos el siguiente módulo como ejemplo:

```haskell
module Test where

gcd :: Int -> Int -> Int
gcd 0 m = m
gcd n 0 = n
gcd n m
  | n > m     = gcd (n - m) m
  | otherwise = gcd (m - n) n
```

Esta función encuentra el máximo común divisor de dos números mediante restas sucesivas. Es un buen ejemplo de un caso donde puedes querer usar PureScript para definir la función, pero tenemos el requerimiento de que debe llamarse desde JavaScript: definir esta función en PureScript usando ajuste de patrones y recursividad es simple, y la implementación se puede beneficiar del uso del comprobador de tipos.

Para entender cómo se puede llamar a esta función desde JavaScript, es importante darse cuenta de que las funciones PureScript siempre se convierten en funciones JavaScript de un único argumento, de manera que tenemos que aplicar sus argumentos uno a uno:

```javascript
var Test = require('Test');
Test.gcd(15)(20);
```

Aquí estoy asumiendo que el código ha sido compilado con `pulp build`, que compila los módulos PureScript a módulos CommonJS. Por esa razón, he sido capaz de hacer referencia a la función `gcd` del objeto `Test` tras importar el módulo `Test` usando `require`.

También puedes querer empaquetar el código JavaScript para el navegador usando `pulp build -O --to file.js`. En ese caso, accederías al módulo `Test` del espacio de nombres global de PureScript, que es por defecto `PS`:

```javascript
var Test = PS.Test;
Test.gcd(15)(20);
```

## Entendiendo la generación de nombres

PureScript intenta preservar los nombres durante la generación de código tanto como sea posible. En particular, se puede esperar que se conserven la mayoría de identificadores que no sean palabras reservadas de PureScript o JavaScript, al menos para los nombres de declaraciones de nivel superior.

Si decides usar una palabra clave de JavaScript como identificador, el nombre se escapará con un símbolo dólar doble. Por ejemplo,

```haskell
null = []
```

genera el siguiente JavaScript:

```javascript
var $$null = [];
```

Además, si quieres usar caracteres especiales en los nombres de tus identificadores, serán escapados usando un símbolo de dólar. Por ejemplo,

```haskell
example' = 100
```

genera el siguiente JavaScript:

```javascript
var example$prime = 100;
```

Cuando se pretenda que el código PureScript compilado sea llamado desde JavaScript, se recomienda que los identificadores usen caracteres alfanuméricos y evitar las palabras reservadas de JavaScript. Si se proporcionan operadores definidos por el usuario en código PureScript, es buena práctica proporcionar una función alternativa con un nombre alfanumérico para ser usada desde JavaScript.

## Representación de datos en tiempo de ejecución (*runtime data representation*)

Los tipos nos permiten razonar en tiempo de compilación sobre si nuestros programas son "correctos" en cierto sentido; esto es, que no se van a romper en tiempo de ejecución. Pero ¿que significa eso? En PureScript significa que el tipo de una expresión debe ser compatible con su representación en tiempo de ejecución.

Por esa razón, es importante entender la representación de los datos en tiempo de ejecución para ser capaz de usar código PureScript y JavaScript juntos de manera efectiva. Esto significa que para cualquier expresión PureScript dada, debemos ser capaces de entender el comportamiento del valor a que se evaluará en tiempo de ejecución.

La buena noticia es que las expresiones PureScript tienen una representación particularmente simple en tiempo de ejecución. Debe ser siempre posible entender la representación de los datos en tiempo de ejecución considerando su tipo.

Para tipos simples, la correspondencia es casi trivial. Por ejemplo, si una expresión tiene el tipo `Boolean`, su valor `v` en tiempo de ejecución debe satisfacer que `typeof v === 'boolean'`. Esto es, las expresiones de tipo `Boolean` evalúan a uno de los valores (JavaScript) `true` o `false`. En particular, no hay ninguna expresión PureScript de tipo `Boolean` que se evalúe a `null` o `undefined`.

Una ley similar es aplicable para las expresiones de tipo `Int`, `Number` y `String`. Las expresiones de tipo `Int` o `Number` se evalúan a números JavaScript no nulos, y las expresiones de tipo `String` se evalúan a cadenas JavaScript no nulas. Las expresiones de tipo `Int` se evaluarán a enteros en tiempo de ejecución, aunque no puedan ser distinguidas de valores de tipo `Number` mediante el uso de `typeof`.

¿Qué pasa con los tipos más complejos?

Como ya hemos visto, las funciones de PureScript se corresponden con funciones de JavaScript de un único argumento. De forma más precisa, si una expresión `f` tiene tipo `a -> b` para algunos tipos `a` y `b`, y una expresión `x` se evalúa a un valor con la representación en tiempo de ejecución correcta para el tipo `a`, entonces `f` se evalúa a una función JavaScript que cuando se aplica al resultado de evaluar `x` tiene la representación correcta en tiempo de ejecución para el tipo `b`. Como ejemplo simple, una expresión de tipo `String -> String` se evalúa a una función que toma cadenas JavaScript no nulas y devuelve cadenas JavaScript no nulas.

Como puedes esperar, los arrays de PureScript se corresponden con arrays de JavaScript. Pero recuerda, los arrays de PureScript son homogéneos, de manera que todos los elementos tienen el mismo tipo. Concretamente, si una expresión PureScript `e` tiene tipo `Array a` para algún tipo `a`, entonces `e` se evalúa a un array JavaScript no nulo donde todos sus elementos tienen la representación en tiempo de ejecución correcta para el tipo `a`.

Hemos visto ya que los registros de PureScript se evalúan a objetos JavaScript. Al igual que para las funciones y los arrays, podemos razonar sobre la la representación en tiempo de ejecución de los datos de los campos de un registro considerando los tipos asociados con sus etiquetas. Por supuesto, los campos de un registro no tienen por qué ser del mismo tipo.

## Representando ADTs

Para todo constructor de tipo de datos algebraico, el compilador PureScript crea un nuevo objeto JavaScript definiendo una función. Sus constructores se corresponden con funciones que crean nuevos objetos JavaScript basados en esos prototipos.

Por ejemplo, considera el siguiente ADT simple:

```haskell
data ZeroOrOne a = Zero | One a
```

El compilador PureScript genera el siguiente código:

```javascript
function One(value0) {
    this.value0 = value0;
};

One.create = function (value0) {
    return new One(value0);
};

function Zero() {
};

Zero.value = new Zero();
```

Aquí vemos dos tipos de objeto JavaScript: `Zero` y `One`. Es posible crear valores de cada tipo usando la palabra reservada de JavaScript `new`. Para los constructores con argumentos, el compilador almacena los datos asociados en campos llamados `value0`, `value1`, etc.

El compilador de PureScript también genera funciones auxiliares. Para los constructores sin argumentos, el compilador genera una propiedad `value` que se puede reutilizar en lugar de usar el operador `new` de forma repetida. Para constructores con uno o más argumentos, el compilador genera una función `create` que toma argumentos con la representación apropiada y los aplica a los constructores apropiados.

¿Qué pasa con los constructores con más de un argumento? En ese caso, el compilador PureScript crea también un nuevo tipo de objeto y una función auxiliar. Esta vez, sin embargo, la función auxiliar es una función currificada de dos argumentos. Por ejemplo, este tipo de datos algebraico:

```haskell
data Two a b = Two a b
```

genera este código JavaScript:

```javascript
function Two(value0, value1) {
    this.value0 = value0;
    this.value1 = value1;
};

Two.create = function (value0) {
    return function (value1) {
        return new Two(value0, value1);
    };
};
```

Aquí, los valores del tipo de objeto `Two` se pueden crear usando la palabra reservada `new` o usando la función `Two.create`.

El caso de los newtypes es un poco diferente. Recuerda que un newtype es como un tipo algebraico de datos, con la restricción de que tiene un único constructor que toma un único argumento. En este caso, la representación en tiempo de ejecución del newtype es de hecho la misma que la del tipo de su argumento.

Por ejemplo, este newtype que representa números de teléfono:

```haskell
newtype PhoneNumber = PhoneNumber String
```

se representa como una cadena JavaScript en tiempo de ejecución. Esto es útil para diseñar bibliotecas, ya que los newtypes proporcionan una capa adicional de seguridad de tipos, pero sin el sobrecoste en tiempo de ejecución de realizar otra llamada a función.

## Representando tipos cuantificados

Las expresiones con tipos cuantificados (polimórficos) tienen representaciones restrictivas en tiempo de ejecución. En la práctica, esto significa que hay relativamente pocas expresiones con un tipo cuantificado dado, pero que podemos razonar sobre ellas de manera bastante efectiva.

Considera este tipo polimórfico por ejemplo:

```haskell
forall a. a -> a
```

¿Qué clase de funciones tienen este tipo? Bien, hay ciertamente una función con este tipo, a saber, la función identidad `id`, definida en el `Prelude`:

```haskell
id :: forall a. a -> a
id a = a
```

De hecho, ¡la función `id` es la _unica_ función (total) con este tipo! Este parece ser el caso ciertamente (intenta escribir una expresión con este tipo que no sea equivalente a `id`), pero ¿cómo podemos estar seguros? Podemos estar seguros considerando la representación en tiempo de ejecución del tipo.

¿Cuál es la representación en tiempo de ejecución de un tipo cuantificado `forall a. t`? Bien, cualquier expresión con la representación en tiempo de ejecución para este tipo tiene que tener la representación en tiempo de ejecución correcta para el tipo `t` para cualquier elección del tipo `a`. En nuestro ejemplo anterior, una función de tipo `forall a. a -> a` tiene que tener la representación en tiempo de ejecución correcta para los tipos `String -> String`, `Number -> Number`, `Array Boolean -> Array Boolean`, y así sucesivamente. Debe convertir cadenas en cadenas, números en números, etc.

Pero eso no es suficiente, la representación en tiempo de ejecución de un tipo cuantificado es más estricta que esto. Requerimos que cualquier expresión sea _paramétricamente polimórfica_, esto es, no puede usar ninguna información sobre el tipo de su argumento en su implementación. Esta condición adicional impide que implementaciones problemáticas como la siguiente función JavaScript pertenezcan a un tipo polimórfico:

```javascript
function invalid(a) {
    if (typeof a === 'string') {
        return "Argument was a string.";
    } else {
        return a;
    }
}
```

Ciertamente, esta función convierte cadenas en cadenas, números en números, etc. Pero no satisface la condición adicional, ya que inspecciona el tipo (en tiempo de ejecución) de sus argumentos, así que esta función no sería un elemento válido dol tipo `forall a. a -> a`.

Sin ser capaces de inspeccionar el tipo en tiempo de ejecución de nuestro argumento, nuestra única opción es devolver el argumento sin cambios, así que `id` es de hecho la única habitante del tipo `forall a. a -> a`.

Una discusión completa del _polimorfismo paramétrico_ y la _parametricidad_ está fuera del alcance de este libro. Fíjate sin embargo en que como los tipos de PureScript se _borran_ en tiempo de ejecución, una función polimórfica en PureScript _no puede_ inspeccionar la representación en tiempo de ejecución de sus argumentos (sin usar la FFI), de manera que esta representación de datos polimórficos es apropiada.

## Representando tipos restringidos

Las funciones con una restricción de clase de tipos tienen una representación interesante en tiempo de ejecución. Ya que el comportamiento de la función dependerá de la instancia de clase de tipos elegida por el compilador, a la función se le pasa un argumento adicional, llamado _diccionario de clase de tipos_ (*type class dictionary*), que contiene la implementación de la funciones de la clase de tipos proporcionada por la instancia elegida.

Por ejemplo, aquí tenemos una función PureScript simple con un tipo restringido que usa la clase de tipos `Show`:

```haskell
shout :: forall a. Show a => a -> String
shout a = show a <> "!!!"
```

El JavaScript generado tiene esta pinta:

```javascript
var shout = function (dict) {
    return function (a) {
        return show(dict)(a) + "!!!";
    };
};
```

Fíjate en que `shout` se compila a una función (currificada) de dos argumentos, no uno. El primer argumento `dict` es el diccionario de clase de tipos para la restricción `Show`. `dict` contiene la implementación de la función `show` para le tipo `a`.

Podemos llamar a esta función desde JavaScript pasando un diccionario de clase de tipos del Prelude de manera explicita como primer argumento:

```javascript
shout(require('Prelude').showNumber)(42);
```

X> ## Ejercicios
X>
X> 1. (Fácil) ¿Cuales son las representaciones en tiempo de ejecución de estos tipos?
X>
X>     ```haskell
X>     forall a. a
X>     forall a. a -> a -> a
X>     forall a. Ord a => Array a -> Boolean
X>     ```
X>
X>     ¿Qué puedes decir sobre las expresiones que tienen estos tipos?
X> 1. (Medio) Intenta usar las funciones definidas en el paquete `purescript-arrays` llamándolas desde JavaScript, compilando la biblioteca usando `pulp build` e importando los módulos usando la función `require` de NodeJS. _Pista_: puedes necesitar configurar la ruta de salida de manera que los módulos CommonJS generados estén disponibles en la ruta de módulos de NodeJS.

## Usando código JavaScript desde PureScript

La forma más simple de usar código JavaScript desde PureScript es dar un tipo a un valor JavaScript existente usando una declaración de _importación externa_ (*foreign import*). Las declaraciones de importaciones externas deben tener una declaración JavaScript correspondiente en un _módulo externo JavaScript_ (*foreign JavaScript module*).

Por ejemplo, considera la función `encodeURIComponent` que se puede usar en JavaScript para codificar una componente de un URI escapando caracteres especiales:

```text
$ node

node> encodeURIComponent('Hello World')
'Hello%20World'
```

Esta función tiene la representación correcta en tiempo de ejecución para el tipo de función `String -> String`, ya que transforma cadenas no nulas en cadenas no nulas, y no tiene otros efectos secundarios.

Podemos asignar este tipo a la función con la siguiente declaración de importación externa:

```haskell
module Data.URI where

foreign import encodeURIComponent :: String -> String
```

Necesitamos también escribir un módulo externo JavaScript. Si el módulo de arriba se salva como `src/Data/URI.purs`, el módulo externo debe salvarse como `src/Data/URI.js`:
`src/Data/URI.js`:

```javascript
"use strict";

exports.encodeURIComponent = encodeURIComponent;
```

Pulp encuentra ficheros `.js` en el directorio `src` y los pasa al compilador como módulos externos JavaScript.

Las funciones JavaScript y los valores se exportan de los módulos externos JavaScript asignándolos al objeto `exports` igual que en un módulo CommonJS normal. El compilador `purs` trata este módulo como un módulo CommonJS normal y simplemente lo añade como una dependencia al módulo PureScript compilado. Sin embargo, cuando empaquetamos código para el navegador con `purs bundle` o `pulp build -O --to` es muy importante seguir el patrón de arriba, asignando lo que queremos exportar al objeto `exports` usando una asignación de propiedad. Esto es porque `purs bundle` reconoce este formato, permitiendo eliminar funciones o valores no usados exportados por JavaScript del código que empaqueta.

Con estas dos piezas en su sitio, podemos ahora usar la función `encodeURIComponent` desde PureScript igual que cualquier función escrita en PureScript. Por ejemplo, si esta declaración se salva como un módulo y se carga en PSCi, podemos reproducir el cálculo de arriba:

```text
$ pulp repl

> import Data.URI
> encodeURIComponent "Hello World"
"Hello%20World"
```

Est enfoque funciona bien para valores JavaScript simples, pero tiene un uso limitado para ejemplos más complicados. La razón es que la mayoría del código idiomático JavaScript no cumple los criterios estrictos que impone la representación en tiempo de ejecución de los tipos básicos de PureScript. En esos casos, tenemos otra opción; podemos _envolver_ el código JavaScript de tal manera que lo forzamos a que se ajuste a la representación en tiempo de ejecución correcta.

## Envolviendo valores JavaScript

Podemos querer envolver valores JavaScript y funciones por una serie de razones:

- Una función toma múltiples argumentos pero queremos llamarla como una función currificada
- Podemos querer usar la mónada `Eff` para llevar registro de cualquier efecto secundario de JavaScript
- Puede ser necesario gestionar casos límite como `null` o `undefined` para obtener la representación en tiempo de ejecución correcta

Por ejemplo, supongamos que queremos recrear la función `head` sobre arrays usando una declaración externa. En JavaScript podríamos escribir la función así:

```javascript
function head(arr) {
    return arr[0];
}
```

Pero hay un problema con esta función. Podemos intentar darle el tipo `forall a. Array a -> a`, pero para arrays vacíos esta función devuelve `undefined`. Por lo tanto, esta función no tiene la representación correcta en tiempo de ejecución y debemos usar una función envoltorio para gestionar este caso límite.

Para mantenerlo simple, podemos lanzar una excepción en el caso de un array vacío. Hablando de manera estricta, las funciones puras no deben lanzar excepciones, pero es suficiente para nuestro propósito ilustrativo y podemos indicar la ausencia de seguridad en el nombre de la función:

```haskell
foreign import unsafeHead :: forall a. Array a -> a
```

En nuestro módulo externo JavaScript podemos definir `unsafeHead` como sigue:

```javascript
exports.unsafeHead = function(arr) {
  if (arr.length) {
    return arr[0];
  } else {
    throw new Error('unsafeHead: empty array');
  }
};
```

## Definiendo tipos externos

Lanzar una excepción en caso de fallo es menos que ideal; el código PureScript idiomático usa el sistema de tipos para representar efectos secundarios como valores ausentes. Un ejemplo de este enfoque es el constructor de tipo `Maybe`. En esta sección construiremos otra solución usando la FFI.

Supongamos que queremos definir un nuevo tipo `Undefined a` cuya representación en tiempo de ejecución sea como la del tipo `a` pero que también permita el valor `undefined`.

Podemos definir un _tipo externo_ (*foreign type*) usando la FFI mediante una _declaración de tipo externo_ (*foreign type declaration*). La sintaxis es similar a la de definir una función externa:

```haskell
foreign import data Undefined :: Type -> Type
```

Fíjate en que la palabra reservada `data` indica que estamos definiendo un tipo, no un valor. En lugar de una firma de tipo, damos la _familia_ del nuevo tipo. En este caso, declaramos que la familia de `Undefined` sea `Type -> Type`. En otras palabras, `Undefined` es un constructor de tipo.

Podemos ahora simplificar nuestra definición original de `head`:

```javascript
exports.head = function(arr) {
  return arr[0];
};
```

Y en el módulo PureScript:

```haskell
foreign import head :: forall a. Array a -> Undefined a
```

Fíjate en los dos cambios: el cuerpo de la función `head` es ahora mucho más simple y devuelve `arr[0]` incluso si ese valor no está definido, y la firma de tipo se ha cambiado para reflejar el hecho de que nuestra función puede devolver un valor indefinido.

Esta función tiene la representación correcta en tiempo de ejecución para su tipo, pero es bastante inútil porque no tenemos manera de usar un valor de tipo `Undefined a`. Pero ¡podemos arreglar eso escribiendo unas funciones nuevas usando la FFI!

La función más básica que necesitamos nos dirá si un valor está definido o no:

```haskell
foreign import isUndefined :: forall a. Undefined a -> Boolean
```

Esto se define fácilmente en nuestro módulo JavaScript como sigue:

```javascript
exports.isUndefined = function(value) {
  return value === undefined;
};
```

Podemos ahora usar `isUndefined` y `head` juntos desde JavaScript para definir una función útil:

```haskell
isEmpty :: forall a. Array a -> Boolean
isEmpty = isUndefined <<< head
```

Aquí, las funciones externas que hemos definido eran muy simples, lo que significa que nos podíamos beneficiar del uso del comprobador de tipos de PureScript tanto como era posible. Esto es una buena práctica en general: las funciones externas deben mantenerse tan pequeñas como sea posible, y la lógica de la aplicación debe moverse a código PureScript donde sea posible.

## Funciones de múltiples argumentos

El Prelude de PureScript contiene un conjunto interesante de ejemplos de tipos externos. Como ya hemos visto, las funciones PureScript toman un único argumento y se pueden usar para simular funciones de varios argumentos mediante _currificado_. Esto tiene varias ventajas; podemos aplicar funciones parcialmente y dar instancias de clase de tipos para los tipos de función, pero viene con un sobrecoste de rendimiento. Para código de rendimiento crítico es necesario a veces definir funciones JavaScript que aceptan múltiples argumentos. El Prelude define tipos externos que nos ayudan a trabajar de manera segura con dichas funciones.

Por ejemplo, la siguiente declaración externa está sacada del módulo `Data.Function.Uncurried` del Prelude:

```haskell
foreign import data Fn2 :: Type -> Type -> Type -> Type
```

Define el constructor de tipo `Fn2` que toma tres argumentos de tipo. `Fn2 a b c` es un tipo que representa funciones JavaScript de dos argumentos de tipos `a` y `b` con valor de retorno de tipo `c`.

El paquete `purescript-functions` define constructores de tipo similares para funciones de aridades entre 0 y 10.

Podemos crear una función de dos argumentos usando la función `mkFn2` como sigue:

```haskell
import Data.Function.Uncurried

divides :: Fn2 Int Int Boolean
divides = mkFn2 \n m -> m % n == 0
```

y podemos aplicar una función de dos argumentos usando la función `runFn2`:

```haskell
> runFn2 divides 2 10
true

> runFn2 divides 3 10
false
```

La clave aquí es que el compilador _expande in situ_ (*inlines*) las funciones `mkFn2` y `runFn2` cuando se aplican por completo. El resultado es que el código generado es muy compacto:

```javascript
exports.divides = function(n, m) {
    return m % n === 0;
};
```

## Representando efectos secundarios

La mónada `Eff` se define también como un tipo externo en el Prelude. Su representación en tiempo de ejecución es bastante simple; una expresión de tipo `Eff eff a` debe evaluarse a una función JavaScript sin argumentos que realiza cualquier efecto secundario y devuelve un valor con la representación en tiempo de ejecución para el tipo `a`.

La definición del constructor de tipo `Eff` viene dada en el módulo `Control.Monad.Eff` como sigue:

```haskell
foreign import data Eff :: # Effect -> Type -> Type
```

Recuerda que el constructor de tipo `Eff` está parametrizado por una fila de efectos y un tipo de retorno, lo que viene reflejado en su familia.

Como ejemplo simple, considera la función `random` definida en el paquete `purescript-random`. Recuerda que su tipo era:

```haskell
foreign import random :: forall eff. Eff (random :: RANDOM | eff) Number
```

La definición de la función `random` es esta:

```javascript
exports.random = function() {
  return Math.random();
};
```

Fíjate en que la función `random` está representada en tiempo de ejecución como una función sin argumentos. Tiene el efecto secundario de generar un valor aleatorio, lo devuelve, y el valor de retorno coincide con la representación en tiempo de ejecución del tipo `Number`: es un número JavaScript no nulo.

Como ejemplo algo más interesante, considera la función `log` definida por el módulo `Control.Monad.Eff.Console` del paquete `purescript-console`. La función `log` tiene el siguiente tipo:

```haskell
foreign import log :: forall eff. String -> Eff (console :: CONSOLE | eff) Unit
```

Y aquí está su definición:

```javascript
exports.log = function (s) {
  return function () {
    console.log(s);
  };
};
```

La representación de `log` en tiempo de ejecución es una función JavaScript de un único argumento que devuelve una función sin argumentos. La función interna realiza el efecto secundario de escribir un mensaje a la consola.

Los efectos `RANDOM` y `CONSOLE` se definen también como tipos externos. Sus familias están definidas como `Effect`, la familia de los efectos. Por ejemplo:

```haskell
foreign import data RANDOM :: Effect
```

De hecho, es posible definir nuevos efectos de esta manera como veremos pronto.

Las expresiones de tipo `Eff eff a` se pueden invocar desde JavaScript como métodos JavaScript normales. Por ejemplo, como la función `main` tiene que tener el tipo `Eff eff a` para algún conjunto de efectos `eff` y algún tipo `a`, se puede invocar como sigue:

```javascript
require('Main').main();
```

Cuando usamos `pulp build -O --to` o `pulp run`, esta llamada a `main` se genera automáticamente si el módulo `Main` está definido.

## Definiendo nuevos efectos

El código fuente para este capítulo define dos nuevos efectos. El más simple es el efecto `ALERT` definido en el módulo `Control.Monad.Eff.Alert`. Se usa para indicar que un cálculo puede alertar al usuario usando una ventana emergente.

El efecto se define primero usando una declaración de tipo externo:

```haskell
foreign import data ALERT :: Effect
```

Se da a `ALERT` la familia `Effect`, indicando que representa un efecto y no un tipo.

A continuación, se define la acción `alert`, que muestra un aviso emergente y añade el efecto `ALERT` a la fila de efectos:

```haskell
foreign import alert :: forall eff. String -> Eff (alert :: ALERT | eff) Unit
```

El módulo JavaScript externo es sencillo, definiendo la función `alert` mediante asignación a la variable `exports`:

```javascript
"use strict";

exports.alert = function(msg) {
    return function() {
        window.alert(msg);
    };
};

```

La acción `alert` es muy similar a la acción `log` del módulo `Control.Monad.Eff.Console`. La única diferencia es que la acción `alert` usa el método `window.alert` y la acción `log` usa el método `console.log`. Como tal, `alert` sólo se puede usar en entornos donde `window.alert` esté definido, como en un navegador web.

Date cuenta de que al igual que en el caso de `log`, la función `alert` usa una función sin argumentos para representar el cálculo de tipo `Eff (alert :: ALERT | eff) Unit`.

El segundo efecto definido en este capítulo es el efecto `STORAGE` definido en el módulo `Control.Monad.Eff.Storage`. Se usa para indicar que un cálculo puede leer o escribir valores usando la API Web Storage.

El efecto se define de la misma manera:

```haskell
foreign import data STORAGE :: Effect
```

El módulo `Control.Monad.Eff.Storage` define dos acciones: `getItem` que extrae un valor del almacenamiento local, y `setItem` que inserta o actualiza un valor en el almacenamiento local. Las dos funciones tienen los tipos siguientes:

```haskell
foreign import getItem
  :: forall eff
   . String
  -> Eff (storage :: STORAGE | eff) Foreign

foreign import setItem
  :: forall eff
   . String
  -> String
  -> Eff (storage :: STORAGE | eff) Unit
```

El lector interesado puede inspeccionar el código fuente de este módulo para ver las definiciones de estas acciones.

`setItem` toma una clave y un valor (ambos cadenas), y devuelve un cálculo que almacena en el almacenamiento local el valor en la clave especificada.

El tipo de `getItem` es más interesante. Toma una clave y trata de extraer el valor asociado del almacenamiento local. Sin embargo, como el método `getItem` de `window.localStorage` puede devolver `null`, el tipo de retorno no es `String`, sino `Foreign` que está definido en el paquete `purescript-foreign` en el módulo `Data.Foreign`.

`Data.Foreign` proporciona una forma de trabajar con _datos no tipados_, o de manera más general, datos cuya representación en tiempo de ejecución es incierta.

X> ## Ejercicios
X>
X> 1. (Medio) Escribe un envoltorio para el método `confirm` del objeto JavaScript `Window` y añade tu función externa al módulo `Control.Monad.Eff.Alert`.
X> 1. (Medio) Escribe un envoltorio para el método `removeItem` del objeto JavaScript `localStorage` y añade tu función externa al módulo `Control.Monad.Eff.Storage`.

## Trabajando con datos no tipados

En esta sección veremos cómo podemos usar la biblioteca `Data.Foreign` para convertir datos no tipados en datos tipados con la representación en tiempo de ejecución correcta para su tipo.

El código para este capítulo se basa en el ejemplo de la agenda del capítulo 8, añadiendo un botón de salvar en la parte inferior del formulario. Cuando se pulsa el botón de salvar, el estado del formulario se serializa a JSON y se almacena en el almacenamiento local. Cuando la página se recarga, el documento JSON se saca del almacenamiento local y se analiza.

El módulo `Main` define un tipo para los datos del formulario salvados:

```haskell
newtype FormData = FormData
  { firstName  :: String
  , lastName   :: String
  , street     :: String
  , city       :: String
  , state      :: String
  , homePhone  :: String
  , cellPhone  :: String
  }
```

El problema es que no tenemos garantías de que el JSON tendrá la forma correcta. Planteándolo de otra manera, no sabemos que el JSON representa el tipo de datos correcto en tiempo de ejecución. Esta es la clase de problema que resuelve la biblioteca `purescript-foreign`. Aquí hay otros ejemplos:

- Una respuesta JSON de un servicio web
- Un valor pasado a una función desde código JavaScript

Probemos la biblioteca `purescript-foreign` y `purescript-foreign-generic` en PSCi. 

Comienza importando algunos módulos:

```text
> import Data.Foreign
> import Data.Foreign.Generic
> import Data.Foreign.JSON
```

Una buena forma de obtener un valor `Foreign` es analizar un documento JSON. `purescript-foreign-generic` define las siguientes funciones:

```haskell
parseJSON :: String -> F Foreign
decodeJSON :: forall a. Decode a => String -> F a
```

El constructor de tipo `F` es de hecho un sinónimo de tipo definido en `Data.Foreign`:

```haskell
type F = Except (NonEmptyList ForeignError)
```

Aquí, `Except` es una mónada para gestionar excepciones en código puro, similar a `Either`. Podemos convertir un valor en la mónada `F` en un valor en la mónada `Either` usando la función `runExcept`.

La mayoría de las funciones de la biblioteca `purescript-foreign` y `purescript-foreign-generic` devuelven un valor en la mónada `F`, lo que significa que podemos usar notación do y los combinadores de funtor aplicativo para construir valores tipados.

La clase de tipos `Decode` representa aquellos tipos que se pueden obtener a partir de datos no tipados. Hay instancias de la clase de tipos definidas para los tipos primitivos y arrays, y podemos también definir nuestras propias instancias.

Intentemos analizar algún documento JSON simple usando `readJSON` en PSCI (recordando usar `runExcept` para desenvolver los resultados):

```text
> import Control.Monad.Except

> runExcept (decodeJSON "\"Testing\"" :: F String)
Right "Testing"

> runExcept (decodeJSON "true" :: F Boolean)
Right true

> runExcept (decodeJSON "[1, 2, 3]" :: F (Array Int))
Right [1, 2, 3]
```

Recuerda que en la mónada `Either`, el constructor de datos `Right` indica éxito. Fíjate sin embargo en que un JSON inválido o un tipo incorrecto acaba en error:

```text
> runExcept (decodeJSON "[1, 2, true]" :: F (Array Int))
(Left (NonEmptyList (NonEmpty (ErrorAtIndex 2 (TypeMismatch "Int" "Boolean")) Nil)))
```

La biblioteca `purescript-foreign-generic` nos dice en qué parte del documento JSON ha ocurrido el error.

## Gestionando valores nulos e indefinidos

Los documentos JSON del mundo real contienen valores nulos e indefinidos, así que necesitamos ser capaces de gestionar estos también.

`purescript-foreign-generic` define un constructor de tipo para resolver este problema: `NullOrUndefined`. Sirve un propósito similar al constructor de tipo `Undefined` que definimos previamente, pero usa el constructor de tipo `Maybe` internamente para representar valores ausentes.

El módulo también proporciona una función `unNullOrUndefined` para desenvolver el valor interno. Podemos elevar la función apropiada sobre la acción `readJSON` para analizar documentos JSON que permiten valores nulos:

```text
> import Prelude
> import Data.Foreign.NullOrUndefined

> runExcept (unNullOrUndefined <$> decodeJSON "42" :: F (NullOrUndefined Int))
(Right (Just 42))

> runExcept (unNullOrUndefined <$> decodeJSON "null" :: F (NullOrUndefined Int))
(Right Nothing)
```

En cada caso, la anotación de tipo se aplica al término a la derecha del operador `<$>`. Por ejemplo, `decodeJSON "42"` tiene el tipo `F (NullOrUndefined Int)`. La función `unNullOrUndefined` se eleva entonces sobre `F` para dar el tipo final `F (Maybe Int)`.

El tipo `NullOrUndefined Int` representa valores que son o bien enteros o null. ¿Qué pasa si queremos analizar valores más interesantes, como arrays de enteros, donde cada elemento puede ser `null`? En ese caso, podemos elevar la función `map unNullOrUndefined` sobre la acción `decodeJSON` como sigue:

```text
> runExcept (map unNullOrUndefined <$> decodeJSON "[1, 2, null]" :: F (Array (NullOrUndefined Int)))
(Right [(Just 1),(Just 2),Nothing])
```

En general, usar newtypes para envolver un tipo existente es una buena forma de proporcionar estrategias de serialización diferentes para el mismo tipo. El tipo `NullOrUndefined` se define como un newtype en torno al constructor de tipos `Maybe`.

## Serialización genérica de JSON

De hecho, ráramente necesitamos escribir instancias de la clase `Decode`, ya que la biblioteca `purescript-foreign-generic` nos permite _derivar_ instancias usando una técnica llamada `programación sobre tipos de datos genéricos` (`datatype-generic programming`). Una explicación de esta técnica está más allá del ámbito de este libro, pero nos permite escribir una funcion una vez y reusarla sobre muchos tipos de datos diferentes, basándose en la estructura de los propios tipos.

Para derivar una instancia `Decode` para nuestro tipo `FormData` (de manera que podamos deserializarlo a partir de su representación JSON), primero usamos la palabra clave `derive` para derivar una instancia de la clase de tipos `Generic`, de esta forma:

```haskell
derive instance genericFormData :: Generic FormData _
```

A continuación, símplemente definimos la función `decode` usando la función `genericDecode` como sigue:

```haskell
instance decodeFormData :: Decode FormData where
  decode = genericDecode (defaultOptions { unwrapSingleConstructors = true })
```

De hecho, podemos también derivar un _codificador_ de la misma forma:

```haskell
instance encodeFormData :: Encode FormData where
  encode = genericEncode (defaultOptions { unwrapSingleConstructors = true })
```

Es importante que usemos las mismas opciones en el codificador y en el decodificador, de otra manera nuestros documentos JSON pueden no decodificarse correctamente.

Cuando se pulsa el botón de salvar, un valor de tipo `FormData` se pasa a la función `encode`, serializándose como un documento JSON. El tipo `FormData` es un newtype para un registro, así que un valor de tipo `FormData` pasado a `encode` será serializado como un _objeto_ JSON. Esto es porque hemos usado la opción `unwrapSingleConstructors` cuendo hemos definido nuestro codificador JSON.

Nuestra instancia de la clase de tipos `Decode` se usa con `decodeJSON` para analizar el documento JSON cuando se lee de almacenamiento local como sigue:

```haskell
loadSavedData = do
  item <- getItem "person"

  let
    savedData :: Either (NonEmptyList ForeignError) (Maybe FormData)
    savedData = runExcept do
      jsonOrNull <- traverse readString =<< readNullOrUndefined item
      traverse decodeJSON jsonOrNull
```

La acción `savedData` lee la estructura `FormData` en dos pasos: primero analiza el valor `Foreign` obtenido de `getItem`. El compilador infiere el tipo de `jsonOrNull` como `Maybe String` (ejercicio para el lector, ¿cómo se infiere este tipo?). La función `traverse` se usa entonces para aplicar `decodeJSON` al (posiblemente ausente) elemento del resultado de tipo `Maybe String`. La instancia de clase de tipos inferida para `decodeJSON` es la que acabamos de escribir, resultando en un valor de tipo `F (Maybe FormData)`.

Necesitamos usar la estructura monádica de `F`, ya que el argumento a `traverse` usa el resultado `jsonOrNull` obtenido en la primera línea.

Hay tres posibilidades para el resultado de `FormData`:

- Si el constructor externo es `Left`, hubo un error analizando la cadena JSON o representaba un valor del tipo erróneo. En este caso, la aplicación muestra un error usando la acción `alert` que escribimos antes.
- Si el constructor externo es `Right` pero el interno es `Nothing`, entonces `getItem` devolvió también `Nothing`, lo que significa que la clave no existía en el almacenamiento local. En este caso, la aplicación continúa silenciosamente.
- Finalmente, un valor que se ajuste al patrón `Right (Just _)` indica un documento JSON analizado con éxito. En este caso, la aplicación actualiza los campos del formulario con los valores apropiados.

Prueba el código ejecutando `pulp build -O --to dist/Main.js`, y abriendo `html/index.html` en el navegador. Debes poder salvar el contenido de los campos del formulario a almacenamiento local pulsando el botón de salvar. Cuando refresques la página los campos deben rellenarse de nuevo.

_Nota_: Puede que sea necesario servir los ficheros HTML y JavaScript a través de un servidor HTTP local para no tener problemas con algunos navegadores.

X> ## Ejercicios
X>
X> 1. (Fácil) Usa `decodeJSON` para analizar un documento JSON que representa un array JavaScript bidimensional de enteros, como `[[1, 2, 3], [4, 5], [6]]`. ¿Qué pasa si se permite que los elementos sean null? ¿Y si se permite que los propios arrays sean null?
X> 1. (Medio) Convéncete de que la implementación de `savedData` debe pasar la comprobación de tipos y escribe los tipos inferidos para cada subexpresión del cálculo.
X> 1. (Medio) El siguiente tipo de datos representa un árbol binario con valores en la hojas:
X>
X>     ```haskell
X>     data Tree a = Leaf a | Branch (Tree a) (Tree a)
X>     ```
X>
X>     Deriva instancias `Encode` y `Decode` para este tipo usando `purescript-foreign-generic`, y verifica que los valores codificados se pueden decodificar correctamente en PSCi.
X> 1. (Difícil) El siguiente tipo `data` debe representarse directamente en JSON como un entero o como una cadena:
X>
X>     ```haskell
X>     data IntOrString
X>       = IntOrString_Int Int
X>       | IntOrString_String String
X>     ```
X>
X>     Escribe instancias `Encode` y `Decode` para el tipo de datos `IntOrString` que implementen este comportamiento, y verifica que los valores codificados pueden decodificarse corréctamente desde PSCi.

## Conclusión

En este capítulo hemos aprendido cómo trabajar con código JavaScript externo desde PureScript y viceversa, y hemos visto los problemas asociados con escribir código fiable usando la FFI:

- Hemos visto la importancia de la _representación en tiempo de ejecución_ de los datos, y de asegurarse de que las funciones externas tienen la representación correcta.
- Hemos aprendido cómo tratar con casos límite como valores null y otros tipos de datos JavaScript, usando tipos externos o el tipo de datos `Foreign`.
- Hemos visto algunos tipos externos comunes definidos en el Prelude y cómo se pueden usar para interoperar con código JavaScript idiomático. En particular, hemos presentado la representación de efectos secundarios en la mónada `Eff` y vimos cómo usar la mónada `Eff` para capturar nuevos efectos secundarios.
- Vimos cómo deserializar datos JSON usando la clase de tipos `Decode`.

Para ver más ejemplos, las organizaciones `purescript` y `purescript-contrib` en GitHub proporcionan muchos ejemplos de bibliotecas que usan la FFI. En los capítulos restantes usaremos algunas de estas bibliotecas para resolver problemas del mundo real de una manera segura a nivel de tipos.
