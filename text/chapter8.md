# La mónada Eff

## Objetivos del capítulo

En el último capítulo hemos presentado los funtores aplicativos, una abstracción que usamos para tratar con _efectos secundarios_: valores opcionales, mensajes de error y validación. Este capítulo presentará otra abstracción para tratar con efectos secundarios de una manera más expresiva: _mónadas_.

El objetivo de este capítulo es explicar por qué las mónadas son una abstracción útil, y su conexión con la _notación do_. Extenderemos el ejemplo de la agenda de capítulos anteriores, usando una mónada particular para gestionar los efectos secundarios de construir una interfaz de usuario en el navegador. La mónada que usaremos es una mónada importante en PureScript, la mónada `Eff`, usada para encapsular los llamados efectos _nativos_.

## Preparación del proyecto

El código fuente para este capítulo se basa en el del capítulo anterior. Los módulos del proyecto anterior se incluyen en el directorio `src` de este proyecto.

El proyecto añade las siguientes dependencias Bower:

- `purescript-eff`, que define la mónada `Eff`, el tema de la segunda mitad del capítulo.
- `purescript-react`, un conjunto de vínculos a la biblioteca de interfaz de usuario React, que usaremos para construir una interfaz para nuestra aplicación de agenda.

Además de los módulos del capítulo anterior, este proyecto añade un módulo `Main` que proporciona el punto de entrada para la aplicación, y funciones para representar la interfaz de usuario.
Para ejecutar este proyecto, instala primero React usando `npm install`, y construye y empaqueta el código JavaScript con `pulp browserify --to dist/Main.js`. Para ejecutar el proyecto, abre el fichero `html/index.html` en tu navegador.
## Mónadas (*monads*) y notación do

La notación do se presentó cuando vimos los _arrays por comprensión_. Los arrays por comprensión proporcionan azúcar sintáctico (*syntactic sugar*) para la función `concatMap` del módulo `Data.Array`.

Considera el siguiente ejemplo. Supongamos que lanzamos dos dados y queremos contar el número de formas en que podemos obtener un total de `n`. Podríamos hacerlo usando el siguiente algoritmo no determinista:

- _Elegimos_ el valor `x` del primer lanzamiento.
- _Elegimos_ el valor `y` del segundo lanzamiento.
- Si la suma de `x` e `y` es `n` entonces devolvemos el par `[x, y]`, en caso contrario fallamos.

Los arrays por comprensión nos permiten escribir este algoritmo no determinista de una manera natural:

```haskell
import Prelude

import Control.Plus (empty)
import Data.Array ((..))

countThrows :: Int -> Array (Array Int)
countThrows n = do
  x <- 1 .. 6
  y <- 1 .. 6
  if x + y == n
    then pure [x, y]
    else empty
```

Podemos ver que esta función es correcta en PSCi:

```text
> countThrows 10
[[4,6],[5,5],[6,4]]

> countThrows 12  
[[6,6]]
```

En el último capítulo, formamos una intuición para el funtor aplicativo `Maybe`; las funciones PureScript pueden empotrarse en un lenguaje de programación mayor que soporta _valores opcionales_. De la misma manera, podemos desarrollar la intuición para la _mónada array_; permite empotrar funciones PureScript en un lenguaje de programación mayor que soporta _elección no determinista_.

En general, una _mónada_ para algún constructor de tipo `m` proporciona una manera de usar notación do con valores de tipo `m a`. Date cuenta de que en el array por comprensión anterior, cada línea contiene un cálculo de tipo `Array a` para algún tipo `a`. En general, cada línea de un bloque de notación do contendrá un cálculo de tipo `m a` para algún tipo `a` y nuestra mónada `m`. La mónada `m` debe ser la misma en cada línea (es decir, fijamos los efectos secundarios), pero los tipos `a` pueden ser diferentes (es decir, cálculos individuales pueden tener distintos tipos de resultado).

Aquí hay otro ejemplo de notación do, esta vez aplicado al constructor de tipo `Maybe`. Supongamos que tenemos un tipo `XML` que representa nodos XML y una función

```haskell
child :: XML -> String -> Maybe XML
```

que busca un elemento hijo de un nodo y devuelve `Nothing` si dicho elemento no existe.

En este caso, podemos buscar un elemento profundamente anidado usando notación do. Supongamos que queremos leer la ciudad de un usuario en un perfil de usuario codificado como un documento XML:

```haskell
userCity :: XML -> Maybe XML
userCity root = do
  prof <- child root "profile"
  addr <- child prof "address"
  city <- child addr "city"
  pure city
```

La función `userCity` busca un elemento hijo `profile`, un elemento `address` dentro del elemento `profile`, y finalmente un elemento `city` dentro del elemento `address`. Si cualquiera de estos elementos no existe, el valor de retorno será `Nothing`. En caso contrario, el valor de retorno se construye usando `Just` con el nodo `city`.

Recuerda, la función `pure` de la última línea está definida para todo funtor `Applicative`. Como `pure` está definido como `Just` para el funtor aplicativo `Maybe`, sería igualmente válido cambiar la última línea por `Just city`.

## La clase de tipos mónada

La clase de tipos `Monad` se define como sigue:

```haskell
class Apply m <= Bind m where
  bind :: forall a b. m a -> (a -> m b) -> m b

class (Applicative m, Bind m) <= Monad m
```

La función clave aquí es `bind`, definida en la clase de tipos `Bind`. Al igual que los operadores `<$>` y `<*>` de las clases de tipos `Functor` y `Apply`, el Prelude define un alias infijo `>>=` para la función `bind`.

La clase de tipos `Monad` extiende `Bind` con las operaciones de la clase de tipos `Applicative` que ya hemos visto.

Será útil ver algunos ejemplos de la clase de tipos `Bind`. Una definición sensata para `Bind` sobre arrays puede ser esta:

```haskell
instance bindArray :: Bind Array where
  bind xs f = concatMap f xs
```

Esto explica la conexión entre los arrays por comprensión y la función `concatMap` a la que nos hemos referido antes.

Aquí tenemos una implementación de `Bind` para el constructor de tipo `Maybe`:

```haskell
instance bindMaybe :: Bind Maybe where
  bind Nothing  _ = Nothing
  bind (Just a) f = f a
```

Esta definición confirma la intuición de que los valores ausentes se propagan a traves de un bloque en notación do.

Veamos cómo la clase de tipos `Bind` se relaciona con la notación do. Considera un bloque en notación do simple que comienza ligando un valor resultado de algún cálculo.

```haskell
do value <- someComputation
   whatToDoNext
```

Cada vez que el compilador de PureScript ve este patrón, reemplaza el código por esto:

```haskell
bind someComputation \value -> whatToDoNext
```

o, escrito de manera infija:

```haskell
someComputation >>= \value -> whatToDoNext
```

El cálculo `whatToDoNext` puede depender de `value`.

Si hay múltiples ligaduras involucradas, esta regla se aplica varias veces, comenzando por arriba. Por ejemplo, el ejemplo `userCity` que vimos antes queda como sigue tras quitarle el azucar:

```haskell
userCity :: XML -> Maybe XML
userCity root =
  child root "profile" >>= \prof ->
    child prof "address" >>= \addr ->
      child addr "city" >>= \city ->
        pure city
```

Merece la pena darse cuenta de que el código expresado usando notación do es a menudo más claro que el código equivalente usando el operador `>>=`. Sin embargo, escribir ligaduras de manera explícita usando `>>=` puede a menudo conducir a oportunidades para escribir código en _forma libre de puntos_, pero hay que tener en cuenta la advertencia habitual sobre la legibilidad. 

## Leyes de la mónada

La clase de tipos `Monad` viene equipada con tres leyes llamadas las _leyes de la mónada_. Estas nos dicen qué podemos esperar de implementaciones sensatas de la clase de tipos `Monad`. 

Es más fácil explicar estas leyes usando notación do.

### Leyes de identidad

La ley de _elemento neutro por la derecha_ (*right-identity*) es la más simple de las tres leyes. Nos dice que podemos eliminar una llamada a `pure` si es la última expresión en un bloque de notación do:

```haskell
do
  x <- expr
  pure x
```

La ley de elemento neutro por la derecha dice que esto es equivalente a `expr`.

La ley de _elemento neutro por la izquierda_ dice que podemos eliminar una llamada a `pure` si es la primera expresión de un bloque en notación do:

```haskell
do
  x <- pure y
  next
```

Esto código es equivalente a `next`, después de que el nombre `x` haya sido reemplazado por la expresión `y`.

La última ley es la _ley de asociatividad_. Nos dice cómo tratar con bloques anidados en notación do. Dice que el siguiente fragmento de código:

```haskell
c1 = do
  y <- do
    x <- m1
    m2
  m3
```

es equivalente a este código:

```haskell  
c2 = do
  x <- m1
  y <- m2
  m3
```

Cada uno de estos cálculos involucra tres expresiones monádicas `m1`, `m2` y `m3`. En cada caso, el resultado de `m1` se liga al nombre `x`, y el resultado de `m2` se asocia al nombre `y`.

En `c1`, las dos expresiones `m1` y `m2` se agrupan en su propio bloque en notación do.

En `c2`, las tres expresiones `m1`, `m2` y `m3` aparecen en el mismo bloque en notación do.

La ley de asociatividad nos dice que es seguro simplificar los bloques anidados en notación do de esta manera.

_Fíjate_ en que por la definición de cómo se quita el azúcar de la notación do convirtiéndola en llamadas a `bind`, tanto `c1` como `c2` son equivalentes a este código:

```haskell  
c3 = do
  x <- m1
  do
    y <- m2
    m3
```

## Plegando con mónadas

Como ejemplo del modo de trabajar con mónadas de manera abstracta, esta sección presentará una función que es válida para cualquier constructor de tipo de la clase de tipos `Monad`. Esto debe servir para solidificar la intuición de que el código monádico corresponde a programar "en un lenguaje mayor" con efectos secundarios, y también ilustra la generalidad que nos proporciona la programación con mónadas.

La función que vamos a escribir se llama `foldM`. Generaliza la función `foldl` que vimos antes a un contexto monádico. Aquí está su firma de tipo:

```haskell
foldM :: forall m a b
       . Monad m
      => (a -> b -> m a)
      -> a
      -> List b
      -> m a
```

Fíjate en que esto es lo mismo que el tipo de `foldl`, excepto por la aparición de la mónada `m`:

```haskell
foldl :: forall a b
       . (a -> b -> a)
      -> a
      -> List b
      -> a
```

De forma intuitiva, `foldM` realiza un pliegue sobre una lista en algún contexto que soporta algún conjunto de efectos secundarios.

Por ejemplo, si `m` fuese `Maybe`, se permitiría a nuestro pliegue fallar devolviendo `Nothing` en cualquier fase; cada paso devuelve un valor opcional y el resultado del pliegue es por lo tanto también opcional.

Si `m` fuese el constructor de tipo `Array`, cada paso del pliegue podría devolver cero o más resultados, y el pliegue continuaría con el siguiente paso independientemente para cada resultado. Al final, el conjunto do resultados consistiría en todos los pliegues sobre todos los caminos posibles. ¡Esto se corresponde al recorrido de un grafo!

Para escribir `foldM` podemos simplemente descomponer la lista de entrada en casos.

Si la lista está vacía, para producir el resultado de tipo `a` sólo tenemos una opción: tenemos que devolver el segundo argumento:

```haskell
foldM _ a Nil = pure a
```

Fíjate en que tenemos que usar `pure` para elevar `a` a la mónada `m`.

¿Qué pasa si la listo no está vacía? En ese caso, tenemos un valor de tipo `a`, un valor de tipo `b`, y una función de tipo `a -> b -> m a`. Si aplicamos la función, obtenemos un resultado monádico de tipo `m a`. Podemos ligar el resultado de este cálculo con la flecha hacia atrás `<-`. 

Sólo queda recurrir sobre la cola de la lista. La implementación es simple:

```haskell
foldM f a (b : bs) = do
  a' <- f a b
  foldM f a' bs
```

Date cuenta de que esta implementación es casi idéntica a la de `foldl` sobre listas, con la excepción de la notación do.

Podemos definir y probar esta función en PSCi. Aquí hay un ejemplo: supongamos que definimos una función de "división segura" sobre enteros, que comprueba la división por cero y usa el constructor de tipo `Maybe` para indicar fallo:

```haskell
safeDivide :: Int -> Int -> Maybe Int
safeDivide _ 0 = Nothing
safeDivide a b = Just (a / b)
```

Podemos entonces usar `foldM` para expresar división segura iterada:

```text
> import Data.List

> foldM safeDivide 100 (fromFoldable [5, 2, 2])
(Just 5)

> foldM safeDivide 100 (fromFoldable [2, 0, 4])
Nothing
```

La función `foldM safeDivide` devuelve `Nothing` si se intenta una división por cero en algún punto. En caso contrario, devuelve el resultado de dividir repetidamente el acumulador, envuelto en el constructor `Just`.

## Mónadas y aplicativos

Toda instancia de la clase de tipos `Monad` es también una instancia de la clase de tipos `Applicative` gracias a la relación de superclase entre ambas.

Sin embargo, hay una implementación de la clase de tipos `Applicative` que viene "gratis" para cualquier instancia de `Monad`, dada por la función `ap`:

```haskell
ap :: forall m a b. Monad m => m (a -> b) -> m a -> m b
ap mf ma = do
  f <- mf
  a <- ma
  pure (f a)
```

Si `m` es un miembro de la clase de tipos `Monad` que respeta las leyes, entonces hay una instancia `Applicative` válida para `m` dada por `ap`.

El lector interesado puede comprobar que `ap` concuerda con `apply` para las mónadas que ya hemos encontrado: `Array`, `Maybe` y `Either e`.

Si toda mónada es también un funtor aplicativo, debemos ser capaces de aplicar nuestra intuición para los funtores aplicativos a todas las mónadas. En particular, podemos esperar razonablemente que una mónada se corresponda, en cierto sentido, a programar "en un lenguaje mayor" aumentado con algún conjunto de efectos secundarios adicional. Debemos ser capaces de elevar funciones de aridad arbitraria, usando `map` y `apply`, a este nuevo lenguaje.

Pero las mónadas nos permiten hacer más de lo que podríamos hacer sólo con funtores aplicativos, y la diferencia clave se pone de relieve con la sintaxis de notación do. Considera de nuevo el ejemplo de `userCity`, en el que buscábamos la ciudad de un usuario en un documento XML que codificaba su perfil de usuario:

```haskell
userCity :: XML -> Maybe XML
userCity root = do
  prof <- child root "profile"
  addr <- child prof "address"
  city <- child addr "city"
  pure city
```

La notación do permite al segundo cálculo depender del resultado `prof` del primero, el tercer cálculo puede depender del resultado `addr` del segundo, y así sucesivamente. Esta dependencia en valores previos no es posible usando sólo la interfaz de la clase de tipos `Applicative`.  

Intenta escribir `userCity` usando sólo `pure` y `apply`: verás que es imposible. Los funtores aplicativos sólo nos permiten elevar argumentos de función que son independientes unos de otros, pero las mónadas nos permiten escribir cálculos que involucran dependencias de datos más interesantes.

En el último capítulo, vimos que la clase de tipos `Applicative` se puede usar para expresar paralelismo. Esto era exactamente porque los argumentos de la función que elevábamos eran independientes unos de otros. Como la clase de tipos `Monad` permite que los cálculos dependan de los resultados de cálculos previos, lo mismo no se aplica; una mónada tiene que combinar sus efectos secundarios en secuencia.

X> ## Ejercicios
X>
X> 1. (Fácil) Busca los tipos de las funciones `head` y `tail` del módulo `Data.array` en el paquete `purescript-arrays`. Usa notación do con la mónada `Maybe` para combinar estas funciones en una función `third` que devuelve el tercer elemento de un array de tres o más elementos. Tu función debe devolver un tipo `Maybe` apropiado.
X> 1. (Medio) Escribe una función `sums` que usa `foldM` para determinar todos los totales posibles que se pueden conseguir usando un conjunto de monedas. Las monedas se especificarán como un array que contiene el valor de cada moneda. Tu función debe tener el siguiente resultado:
X>
X>     ```text
X>     > sums []
X>     [0]
X>
X>     > sums [1, 2, 10]
X>     [0,1,2,3,10,11,12,13]
X>     ```
X>
X>     _Pista_: Esta función se puede escribir en una línea usando `foldM`. Querrás usar las funciones `nub` y `sort` para eliminar duplicados y ordenar el resultado respectivamente.
X> 1. (Medio) Confirma que la función `ap` y el operador `apply` concuerdan para la mónada `Maybe`.
X> 1. (Medio) Verifica que las leyes de la mónada se cumplen para la instancia `Monad` del tipo `Maybe` definida en el paquete `purescript-maybe`.
X> 1. (Medio) Escribe una función `filterM` que generaliza la función `filter` sobre listas. Tu función debe tener la siguiente firma de tipo:
X>
X>     ```haskell
X>     filterM :: forall m a. Monad m => (a -> m Boolean) -> List a -> m (List a)
X>     ```
X>
X>     Prueba tu función en PSCi usando las mónadas `Maybe` y `Array`.
X> 1. (Difícil) Toda mónada tiene una instancia de `Functor` por defecto dada por:
X>
X>     ```haskell
X>     map f a = do
X>       x <- a
X>       pure (f x)
X>     ```
X>
X>     Usa las leyes de la mónada para probar que para cualquier mónada, lo siguiente se cumple:
X>
X>     ```haskell
X>     lift2 f (pure a) (pure b) = pure (f a b)
X>     ```
X>     
X>     Donde la instancia `Applicative` usa la función `ap` definida antes. Recuerda que `lift2` se definió como sigue:
X>    
X>     ```haskell
X>     lift2 :: forall f a b c. Applicative f => (a -> b -> c) -> f a -> f b -> f c
X>     lift2 f a b = f <$> a <*> b
X>     ```

## Efectos nativos (*native effects*)

Veremos una mónada particular que tiene una importancia central en PureScript; la mónada `Eff`.

La mónada `Eff` está definida en el Prelude, en el módulo `Control.Monad.Eff`. Se usa para gestionar los llamados efectos secundarios _nativos_.

¿Qué son los efectos secundarios nativos? Son efectos secundarios que distinguen las expresiones JavaScript de las expresiones idiomáticas PureScript, que normalmente están libres de efectos secundarios. Algunos ejemplos de efectos nativos son: 

- Entrada/salida por consola
- Generación de números aleatorios
- Excepciones
- Lectura/escritura de estado mutable

Y en el navegador:

- Manipulación del DOM
- Llamadas XMLHttpRequest / AJAX
- Interactuar con un websocket
- Escribir/leer de/a almacenamiento local

Hemos visto ya varios ejemplos de efectos secundarios "no nativos":

- Valores opcionales representados por el tipo de datos `Maybe`
- Errores representados por el tipo de datos `Either`
- Multi-funciones, representadas por arrays o listas

Date cuenta de que la distinción es sutil. Es cierto, por ejemplo, que un mensaje de error es un posible efecto secundario de una expresión JavaScript, en forma de excepción. En ese sentido, las excepciones representan efectos secundarios nativos y es posible representarlas usando `Eff`. Sin embargo, los mensajes de error implementados usando `Either` no son un efecto secundario del runtime de JavaScript, de manera que no es apropiado implementar los mensajes de error de ese estilo usando `Eff`. Entonces no es el efecto en sí lo que es nativo, sino la forma como se implementa en tiempo de ejecución.

## Efectos secundarios y pureza

En un lenguaje puro como PureScript, una pregunta que se plantea es: sin efectos secundarios, ¿cómo se puede escribir código útil en el mundo real?

La respuesta es que PureScript no pretende eliminar los efectos secundarios. Tiene como objetivo representar los efectos secundarios de tal manera que los cálculos puros se puedan distinguir de los cálculos con efectos secundarios en el sistema de tipos. En este sentido, el lenguaje sigue siendo puro.

Los valores con efectos secundarios tienen tipo diferente al de los valores puros. Así, no es posible pasar un argumento con efectos secundarios a una función, por ejemplo, y tener efectos secundarios que se ejecutan de manera no esperada.

La única forma en que los efectos secundarios gestionados por la mónada `Eff` se presentarán es ejecutar un cálculo de tipo `Eff eff a` desde JavaScript.

La herramienta de construcción Pulp (y otras herramientas) proporciona un atajo, generando JavaScript adicional para invocar el cálculo `main` cuando la aplicación comienza. `main` tiene que ser un cálculo en la mónada `Eff`.

De esta manera, sabemos exactamente qué efectos secundarios esperar: exactamente los usados por `main`. Además, podemos usar la mónada `Eff` para restringir qué tipo de efectos secundarios puede tener `main`, de manera que podemos decir con exactitud, por ejemplo, que nuestra aplicación interactuará con la consola, pero nada más.

## La mónada Eff

El objetivo de la mónada `Eff` es proporcionar un API bien tipado para cálculos con efectos secundarios, al tiempo que genera JavaScript eficiente. También recibe el nombre de mónada de _efectos extensibles_ (*extensible effects*), que explicaremos en breve.

Aquí hay un ejemplo. Usa el paquete `purescript-random` que define funciones para generar números aleatorios:

```haskell
module Main where

import Prelude

import Control.Monad.Eff.Random (random)
import Control.Monad.Eff.Console (logShow)

main = do
  n <- random
  logShow n
```  

Si salvamos este fichero como `src/Main.purs`, podemos compilarlo y ejecutarlo usando Pulp:

```text
$ pulp run
```

Al ejecutar este comando, verás un número aleatorio elegido entre `0` y `1` impreso en la consola.

Este programa usa notación do para combinar dos tipos de efectos nativos proporcionados por el runtime de JavaScript: generación de números aleatorios y entrada/salida por consola.

## Efectos extensibles (*extensible effects*)

Podemos inspeccionar el tipo de `main` abriendo el módulo en PSCi:

```text
> import Main

> :type main
forall eff. Eff (console :: CONSOLE, random :: RANDOM | eff) Unit
```

Este tipo parece bastante complicado, pero es fácil de explicar por analogía con los registros de PureScript.

Considera una función simple que usa un tipo registro:

```haskell
fullName person = person.firstName <> " " <> person.lastName
```

Esta función crea una cadena de nombre completo a partir de un registro que contiene propiedades `firstName` y `lastName`. Si averiguas el tipo de esta función en PSCi como antes, verás esto:

```haskell
forall r. { firstName :: String, lastName :: String | r } -> String
```

Este tipo se lee como sigue: "`fullName` toma un registro con campos `firstName` y `lastName` y _otras propiedades cualesquiera_ y devuelve una `String`".

Esto es, a `fullName` no le importa si pasas un registro con más campos, siempre y cuando las propiedades `firstName` y `lastName` estén presentes:

```text
> firstName { firstName: "Phil", lastName: "Freeman", location: "Los Angeles" }
Phil Freeman
```

De manera similar, el tipo de `main` de arriba se puede interpretar como sigue: "`main` es un _cálculo con efectos secundarios_, que se puede ejecutar en cualquier entorno que soporte generación de números aleatorios y entrada/salida por consola, _y cualquier otro tipo de efectos secundarios_, y que devuelve un valor de tipo `Unit`".

Este es el origen del nombre "efectos extensibles": podemos siempre extender el conjunto de efectos secundarios, siempre y cuando soportemos el conjunto de efectos que necesitamos.

## Intercalando efectos

Esta extensibilidad permite al código en la mónada `Eff` _intercalar_ (*interleave*) distintos tipos de efectos secundarios.

La función `random` que hemos usado tiene el siguiente tipo:

```haskell
forall eff1. Eff (random :: RANDOM | eff1) Number
```

El conjunto de efectos `(random :: RANDOM | eff1)` que vemos aquí _no_ es es el mismo que el que aparece en `main`.

Sin embargo, podemos _instanciar_ el tipo `random` de tal manera que los efectos coinciden. Si elegimos que `eff1` sea `(console :: CONSOLE | eff)`, entonces ambos conjuntos de efectos son iguales, salvo por reordenación.

De manera similar, `logShow` tiene un tipo que se puede especializar para que coincida con los efectos de `main`:

```haskell
forall eff2. Show a => a -> Eff (console :: CONSOLE | eff2) Unit
```
Esta vez, hemos elegido que `eff2` sea `(random :: RANDOM | eff)`.

La cuestión es que los tipos de `random` y `logShow` indican los efectos secundarios que contienen, pero de tal manera que otros efectos secundarios puedan ser _mezclados_ para construir cálculos más grandes con conjuntos de efectos secundarios más grandes.

Fíjate en que no tenemos que dar un tipo para `main`. El compilador encontrará el tipo más general para `main` dados los tipos polimórficos de `random` y `logShow`.

## La familia de Eff

El tipo de `main` no se parece a los otros tipos que hemos visto antes. Para explicarlo, necesitamos considerar la _familia_ (*kind*) de `Eff`. Recuerda que los tipos se clasifican por sus familias de la misma manera que los valores se clasifican por sus tipos. Hasta ahora hemos visto sólo familias construidas a partir de `Type` (la familia de tipos) y `->` (que construye familias para constructores de tipos).

Para averiguar la familia de `Eff`, usa el comando `:kind` en PSCi:

```text
> import Control.Monad.Eff

> :kind Eff
# Control.Monad.Eff.Effect -> Type -> Type
```

Hay dos símbolos que no hemos visto antes.

`Control.Monad.Eff.Effect` es la familia de _efectos_, que representa _etiquetas a nivel de tipo_ (*type-level labels*) para distintos tipos de efectos secundarios. Para entender esto, fíjate en que las dos etiquetas que vimos en `main` tienen ambas familia `Control.Monad.Eff.Effect`:

```text
> import Control.Monad.Eff.Console
> import Control.Monad.Eff.Random

> :kind CONSOLE
Control.Monad.Eff.Effect

> :kind RANDOM
Control.Monad.Eff.Effect
```

El constructor de familia `#` se usa para construir familias para _filas_, es decir, conjuntos etiquetados sin orden.

Así, `Eff` está parametrizada por una fila de efectos y su tipo de retorno. Esto es, el primer argumento a `Eff` es un conjunto etiquetado no ordenado de tipos de efectos, y el segundo parámetro es el tipo de retorno.

Podemos ya leer el tipo de `main` expuesto antes:

```text
forall eff. Eff (console :: CONSOLE, random :: RANDOM | eff) Unit
```

El primer argumento a `Eff` es `(console :: CONSOLE, random :: RANDOM | eff)`. Esto es una fila que contiene el efecto `CONSOLE` y el efecto `RANDOM`. El símbolo barra `|` separa los efectos etiquetados de la _variable de fila_ (*row variable*) `eff` que representa _cualquier otro efecto secundario_ que queramos mezclar.

El segundo argumento a `Eff` es `Unit`, que es el tipo del valor de retorno del cálculo.

## Objetos y filas

Considerar la familia de `Eff` nos permite establecer una conexión más profunda entre efectos extensibles y registros.

Toma la función que definimos antes:

```haskell
fullName :: forall r. { firstName :: String, lastName :: String | r } -> String
fullName person = person.firstName <> " " <> person.lastName
```

La familia del tipo a la izquierda de la flecha de función ha de ser `Type`, porque sólo los tipos de familia `Type` tienen valores.

Las llaves son de hecho azúcar sintáctico, y el tipo completo que el compilador PureScript entiende es como sigue:

```haskell
fullName :: forall r. Record (firstName :: String, lastName :: String | r) -> String
```

Date cuenta de que las llaves han sido eliminadas y hay un constructor extra `Record`. `Record` es un constructor de tipo incorporado definido en el módulo `Prim`. Si buscamos su familia vemos lo siguiente:

```text
> :kind Record
# Type -> Type
```

Esto es, `Record` es un constructor de tipo que toma una _fila de tipos_ y construye un tipo. Esto es lo que nos permite escribir funciones polimórficas por fila sobre registros.

El sistema de tipos usa la misma maquinaria para gestionar los efectos extensibles y se usa para registros polimórficos por fila (o _registros extensibles_). La única diferencia es la _familia_ de los tipos que aparecen en las etiquetas. Los registros se parametrizan por una fila de tipos, y `Eff` se parametriza por una fila de efectos.

La misma característica del sistema de tipos se podría usar para construir otros tipos parametrizados por filas de constructores de tipos, ¡o incluso filas de filas!

## Efectos de grano fino (*fine-grained effects*)

Las anotaciones de tipo no suelen ser necesarias cuando usamos `Eff`, ya que las filas de efectos se pueden inferir, pero se pueden usar para indicar al compilador qué efectos se esperan de un cálculo.

Si anotamos el ejemplo previo con una fila de efectos _cerrada_

``` haskell
main :: Eff (console :: CONSOLE, random :: RANDOM) Unit
main = do
  n <- random
  print n
```

(fíjate en que no hay variable de fila `eff` aquí), entonces no podemos incluir accidentalmente un subcálculo que hace uso de un tipo de efectos diferente. De esta manera, podemos controlar los efectos secundarios que permitimos a nuestro código.

## Gestores (*handlers*) y acciones (*actions*)

Las funciones como `print` y `random` se llaman _acciones_. Las acciones tienen el tipo `Eff` a la parte derecha de sus funciones, y su propósito es _intoducir_ nuevos efectos.

Esto contrasta con los _gestores_, en los que el tipo `Eff` aparece como tipo de un argumento de la función. Mientras que las acciones _suman_ al conjunto de efectos requeridos, un gestor normalmente _resta_ efectos del conjunto.

Como ejemplo, considera el paquete `purescript-exceptions`. Define dos funciones, `throwException` y `catchException`:

```haskell
throwException :: forall a eff
                . Error
               -> Eff (exception :: EXCEPTION | eff) a

catchException :: forall a eff
                . (Error -> Eff eff a)
               -> Eff (exception :: EXCEPTION | eff) a
               -> Eff eff a
```

`throwException` es una acción. `Eff` aparece en la parte derecha e introduce el nuevo efecto `EXCEPTION`.

`catchException` es un gestor. `Eff` aparece como tipo del segundo argumento de la función, y el efecto neto es _eliminar_ el efecto `EXCEPTION`.

Esto es útil, porque el sistema de tipos se puede usar para delimitar porciones de código que requieren un efecto concreto. Ese código se puede envolver en un gestor, permitiéndole ser empotrado dentro de un bloque de código que no permite ese efecto.

Por ejemplo, podemos escribir un fragmento de código que arroja excepciones usando el efecto `Exception`, y luego envolver ese código usando `catchException` para empotrar el cálculo en un fragmento de código que no permite excepciones.

Supongamos que queremos leer la configuración de nuestra aplicación de un documento JSON. El proceso de analizar el documento puede resultar en una excepción. El proceso de leer y analizar la configuración se puede escribir como una función con esta firma de tipo:

``` haskell
readConfig :: forall eff. Eff (exception :: EXCEPTION | eff) Config
```

Entonces, en la función `main`, podemos usar `catchException` para gestionar el efecto `EXCEPTION` anotando el error y devolviendo una configuración por defecto:

```haskell
main = do
    config <- catchException printException readConfig
    runApplication config
  where
    printException e = do
      log (message e)
      pure defaultConfig
```

El paquete `purescript-eff` también define el gestor `runPure`, que toma un cálculo _sin_ efectos secundarios y lo evalúa de manera segura como un valor puro:

```haskell
type Pure a = Eff () a

runPure :: forall a. Pure a -> a
```

## Estado mutable

Hay otro efecto definido en las bibliotecas base: el efecto `ST`.

El efecto `ST` se usa para manipular estado mutable. Como programadores funcionales puros, sabemos que el estado mutable compartido puede ser problemático. Sin embargo, el efecto `ST` usa el sistema de tipos para restringir el uso compartido de tal manera que sólo se permita mutación _local_ segura.

El efecto `ST` se define en el módulo `Control.Monad.ST`. Para ver cómo funciona, necesitamos mirar los tipos de sus acciones:

```haskell
newSTRef :: forall a h eff. a -> Eff (st :: ST h | eff) (STRef h a)

readSTRef :: forall a h eff. STRef h a -> Eff (st :: ST h | eff) a

writeSTRef :: forall a h eff. STRef h a -> a -> Eff (st :: ST h | eff) a

modifySTRef :: forall a h eff. STRef h a -> (a -> a) -> Eff (st :: ST h | eff) a
```

`newSTRef` se usa para crear una nueva referencia a una celda mutable de tipo `STRef h a`, que se puede leer usando la acción `readSTRef` y se puede modificar usando las acciones `writeSTRef` y `modifySTRef`. El tipo `a` es el tipo del valor almacenado en la celda, y el tipo `h` se usa para indicar una _región de memoria_ en el sistema de tipos.

Aquí tenemos un ejemplo. Supongamos que queremos simular el movimiento de una partícula cayendo por la gravedad mediante la iteración de una función de actualización sobre un gran número de pequeños pasos de tiempo.

Podemos hacer esto creando una referencia a una celda mutable que contendrá la posición y velocidad de la partícula, y mediante un bucle for (usando la acción `forE` de `Control.Monad.Eff`) actualizar el valor almacenado en esa celda:

```haskell
import Prelude

import Control.Monad.Eff (Eff, forE)
import Control.Monad.ST (ST, newSTRef, readSTRef, modifySTRef)

simulate :: forall eff h. Number -> Number -> Int -> Eff (st :: ST h | eff) Number
simulate x0 v0 time = do
  ref <- newSTRef { x: x0, v: v0 }
  forE 0 (time * 1000) \_ -> do
    modifySTRef ref \o ->
      { v: o.v - 9.81 * 0.001
      , x: o.x + o.v * 0.001
      }
    pure unit
  final <- readSTRef ref
  pure final.x
```

Al final del cálculo, leemos el valor final de la referencia a celda y devolvemos la posición de la partícula.

Fíjate en que aunque esta función usa estado mutable, sigue siendo una función pura siempre y cuando la referencia a celda `ref` no se use en otras partes del programa. Veremos que esto es exactamente lo que el efecto `ST` no permite.

Para ejecutar un cálculo con el efecto `ST` tenemos que usar la función `runST`:

```haskell
runST :: forall a eff. (forall h. Eff (st :: ST h | eff) a) -> Eff eff a
```

Lo que tenemos que observar aquí es que el tipo de la region `h` está cuantificado _dentro de los paréntesis_ a la izquierda de la flecha de función. Significa que cualquier acción que pasemos a `runST` tiene que funcionar con _cualquier region_ `h`.

Sin embargo, una vez que una referencia a celda ha sido creada por `newSTRef`, su tipo de región ya se ha fijado, de manera que sería un error de tipos usar la referencia fuera del código delimitado por `runST`. Esto es lo que permite a `runST` eliminar el efecto `ST` de manera segura.

De hecho, ya que `ST` es el único efecto de nuestro ejemplo, podemos usar `runST` junto a `runPure` para convertir `simulate` en una función pura:

```haskell
simulate' :: Number -> Number -> Number -> Number
simulate' x0 v0 time = runPure (runST (simulate x0 v0 time))
```

Puedes incluso intentar ejecutar la función en PSCi:

```text
> import Main

> simulate' 100.0 0.0 0.0
100.00

> simulate' 100.0 0.0 1.0
95.10

> simulate' 100.0 0.0 2.0
80.39

> simulate' 100.0 0.0 3.0
55.87

> simulate' 100.0 0.0 4.0
21.54
```

De hecho, si expandimos la definición de `simulate` en la llamada a `runST` como sigue:

```haskell
simulate :: Number -> Number -> Int -> Number
simulate x0 v0 time = runPure $ runST do
  ref <- newSTRef { x: x0, v: v0 }
  forE 0 (time * 1000) \_ -> do
    modifySTRef ref \o ->  
      { v: o.v - 9.81 * 0.001
      , x: o.x + o.v * 0.001  
      }
    pure unit  
  final <- readSTRef ref
  pure final.x
```

el compilador se dará cuenta de que la referencia a celda no puede escapar de su ámbito y puede convertirla de manera segura en una `var`. Aquí está el JavaScript generado para el cuerpo de la llamada a `runST`:

```javascript
var ref = { x: x0, v: v0 };

Control_Monad_Eff.forE(0)(time * 1000 | 0)(function (i) {
  return function __do() {
    ref = (function (o) {
      return {
        v: o.v - 9.81 * 1.0e-3,
        x: o.x + o.v * 1.0e-3
      };
    })(ref);
    return Prelude.unit;
  };
})();

return ref.x;
```

El efecto `ST` es una buena forma de generar JavaScript corto cuando trabajamos con estado mutable en ámbito local, especialmente cuando se usa junto a acciones como `forE`, `foreachE`, `whileE` y `untilE` que generan bucles eficientes en la mónada `Eff`.

X> ## Ejercicios
X>
X> 1. (Medio) Reescribe la función `safeDivide` para lanzar una excepción usando `throwException` si el denominador es cero.
X> 1. (Difícil) El siguiente es un método simple para estimar pi: elige de manera aleatoria un gran número `N` de puntos en el cuadrado unidad y cuenta el número `n` que cae en el círculo inscrito. Una estimación para pi es `4n/N`. Usa los efectos `RANDOM` y `ST` con la función `forE` para escribir una función que estima pi de esta forma.

## Efectos DOM

En las secciones finales de este capítulo, aplicaremos lo que hemos aprendido sobre efectos en la mónada `Eff` al problema de trabajar con el DOM.

Hay un número de paquetes PureScript para trabajar directamente con el DOM o con bibliotecas DOM de código abierto. Por ejemplo:

- [`purescript-dom`](http://github.com/purescript-contrib/purescript-dom) es un conjunto amplio de vínculos de bajo nivel al API DOM del navegador.
- [`purescript-jquery`](http://github.com/paf31/purescript-jquery) es un conjunto de vínculos a la biblioteca [jQuery](http://jquery.org).

Hay también bibliotecas PureScript que construyen abstracciones sobre estas bibliotecas, como

- [`purescript-thermite`](http://github.com/paf31/purescript-thermite), que se basa en `purescript-react`, y 
- [`purescript-halogen`](http://github.com/slamdata/purescript-halogen) que proporciona un conjunto de abstracciones seguras a nivel de tipos sobre una biblioteca de DOM virtual propia.

En este capítulo, usaremos la biblioteca `purescript-react` para añadir una interfaz de usuario a nuestra agenda, pero animamos al lector interesado a explorar enfoques alternativos. 

## Una interfaz de usuario para la agenda

Usando la biblioteca `purescript-react`, definiremos nuestra aplicación como una _componente_ React. Las componentes React describen elementos HTML en código como estructuras de datos puras, que son presentadas de manera eficiente al DOM. Además, las componentes pueden responder a eventos como pulsaciones de botón. La biblioteca `purescript-react` usa la mónada `Eff` para describir cómo gestionar estos eventos.

Un tutorial completo de la biblioteca React está bastante fuera del alcance de este capítulo, pero animamos al lector a consultar su documentación cuando sea necesario. Para nuestros propósitos, React proporciona un ejemplo práctico de la mónada `Eff`.

Vamos a construir un formulario que permita a un usuario añadir una nueva entrada a nuestra agenda. El formulario contendrá cajas de texto para varios campos (nombre, apellido, ciudad, estado, etc.), y un área en la que mostraremos los errores de validación. Según vaya escribiendo texto el usuario en las cajas de texto, los errores de validación se actualizarán.

Para mantener las cosas simples, el formulario tendrá una forma fija: los diferentes tipos de número de teléfono (casa, móvil, trabajo, otro) se pedirán en cajas de texto separadas.

El fichero HTML está básicamente vacío, excepto por la siguiente línea:

```html
<script type="text/javascript" src="../dist/Main.js"></script>
```

Esta línea incluye el código JavaScript generado por Pulp. La ponemos al final del fichero para asegurarnos de que los elementos relevantes están en la página antes de que tratemos de accederlos. Para reconstruir el fichero `Main.js` se puede usar Pulp con el comando `browserify`. Asegúrate primero de que el directorio `dist` existe y de que has instalado React como una dependencia NPM:

```text
$ npm install # Install React
$ mkdir dist/
$ pulp browserify --to dist/Main.js
```

El módulo `Main` define la función `main`, que crea la componente agenda y la representa en pantalla. La función `main` usa sólo los efectos `CONSOLE` y `DOM` como indica su firma de tipo:

```haskell
main :: Eff (console :: CONSOLE, dom :: DOM) Unit
```

Primero, `main` registra un mensaje de estado en la consola:

```haskell
main = void do
  log "Rendering address book component"
```

Después, `main` usa la API DOM para obtener una referencia (`doc`) al cuerpo del documento:

```haskell
  doc <- window >>= document
```

Fíjate en que esto proporciona un ejemplo de efectos intercalados: la función `log` usa el efecto `CONSOLE`, y las funciones `window` y `document` usan ambas el efecto `DOM`. El tipo de `main` indica que usa ambos efectos.

`main` usa la acción `window` para obtener una referencia al objeto ventana y pasa el resultado a la función `document` usando `>>=`. `document` toma un objeto ventana y devuelve una referencia a su documento.

Date cuenta de que, por la definición de la notación do, podríamos haber escrito esto como sigue:

```haskell
  w <- window
  doc <- document w
```

Si esto es más o menos legible es un problema de preferencia personal. La primera versión es un ejemplo de estilo _libre de puntos_, ya que no hay argumentos a función con nombre, al contrario que la segundo versión que usa el nombre `w` para el objeto ventana.

El módulo `Main` define una _componente_ agenda, llamada `addressBook`. Para entender su definición, necesitaremos primero entender algunos conceptos.

Para crear una componente React, debemos primero crear una _clase_ React, que actúa como una plantilla para una componente. En `purescript-react` podemos crear clases usando la función `createClass`. `createClass` require una _especificación_ de nuestra clase, que es esencialmente una colección de acciones `Eff` que se usan para gestionar varias partes de ciclo de vida de la componente. La acción que nos interesa es la acción `Render`.

Aquí están los tipos de algunas funciones relevantes proporcionadas por la biblioteca React:

```haskell
createClass
  :: forall props state eff
   . ReactSpec props state eff
  -> ReactClass props

type Render props state eff
   = ReactThis props state
  -> Eff ( props :: ReactProps
         , refs :: ReactRefs Disallowed
         , state :: ReactState ReadOnly
         | eff
         ) ReactElement

spec
  :: forall props state eff
   . state
  -> Render props state eff
  -> ReactSpec props state eff
```

Hay unas cuantas cosas interesantes en las que fijarse aquí:

- El sinónimo de tipo `Render` se porporciona para simplificar algunas firmas de tipo, y denota la función representadora de una componente.
- Una acción `Render` toma una referencia a la componente (de tipo `ReactThis`), y devuelve un `ReactElement` en la mónada `Eff`. Un `ReactElement` es una estructura de datos que describe el estado deseado del DOM tras la representación.
- Cada componente React define algún tipo de estado. El estado puede cambiar en respuesta a eventos como pulsación de botones. En `purescript-react`, el valor inicial del estado se proporciona en la función `spec`.
- La fila de efectos del tipo `Render` usa algunos efectos interesantes para restringir el acceso al estado de la componente React en ciertas funciones. Por ejemplo, durante la representación, el acceso al objeto "refs" no está permitido (`Disallowed`), y el acceso al estado de la componente es de sólo lectura (`ReadOnly`).

El módulo `Main` define un tipo de estados para la componente agenda y un estado inicial:

```haskell
newtype AppState = AppState
  { person :: Person
  , errors :: Errors
  }

initialState :: AppState
initialState = AppState
  { person: examplePerson
  , errors: []
  }
```

El estado contiene un registro `Person` (que haremos editable usando componentes formulario) y una colección de errores (que se rellenará usando nuestro código de validación existente).

Veamos ahora la definición de nuestra componente:

```haskell
addressBook :: forall props. ReactClass props
```

Como ya hemos indicado, `addressBook` usará `createClass` y `spec` para crear una clase React. Para hacerlo, proporcionará nuestro valor de estado inicial y una acción `Render`. Sin embargo, ¿que podemos hacer en la acción `Render`? Para responder a eso, `purescript-react` proporciona algunas acciones simples que podemos usar:

```haskell
readState
  :: forall props state access eff
   . ReactThis props state
  -> Eff ( state :: ReactState ( read :: Read
                               | access
                               )
         | eff
         ) state

writeState
  :: forall props state access eff
   . ReactThis props state
  -> state
  -> Eff ( state :: ReactState ( write :: Write
                               | access
                               )
         | eff
         ) state
```

Las funciones `readState` y `writeState` usan efectos extensibles para asegurarse de que tenemos acceso al estado de React (a través del efecto `ReactState`), pero fíjate en que los permisos de lectura y escritura están separados, parametrizando el efecto `ReactState` mediante _otra_ fila.

Esto ilustra un punto interesante acerca de los efectos basados en fila de PureScript: los efectos que aparecen dentro de las filas no tienen por qué ser singletons simples, sino que pueden tener estructura interesante, y esta flexibilidad permite algunas restricciones útiles en tiempo de compilación. Si la biblioteca `purescript-react` no usase esta restricción sería posible obtener excepciones en tiempo de ejecución si por ejemplo intentásemos escribir el estado en la acción `Render`. En su lugar, dichos errores son ahora detectados en tiempo de compilación.

Ahora podemos leer la definición de nuestra componente `addressBook`. Comienza leyendo el estado actual de la componente:

```haskell
addressBook = createClass $ spec initialState \ctx -> do
  AppState { person: Person person@{ homeAddress: Address address }
           , errors
           } <- readState ctx
```

Fíjate en que:

- El nombre `ctx` se refiere a la referencia a `ReactThis`, y se puede usar para leer y escribir el estado donde sea apropiado.
- El registro dentro de `AppState` se ajusta usando una ligatura de registro, incluyendo un doble sentido de registro (*record pun*) para el campo _errors_. Nombramos explícitamente varias partes de la estructura de estado por conveniencia.

Recuerda que `Render` debe devolver una estructura `ReactElement`, representando el estado deseado del DOM. La acción `Render` se define en términos de unas funciones auxiliares. Una de dichas funciones auxiliares es `renderValidationErrors`, que convierte la estructura `Errors` en un array de `ReactElement`s.

```haskell
renderValidationError :: String -> ReactElement
renderValidationError err = D.li' [ D.text err ]

renderValidationErrors :: Errors -> Array ReactElement
renderValidationErrors [] = []
renderValidationErrors xs =
  [ D.div [ P.className "alert alert-danger" ]
          [ D.ul' (map renderValidationError xs) ]
  ]
```

En `purescript-react`' los `ReactElement`s se crean típicamente aplicando funciones como `div` que crean elementos HTML. Estas funciones normalmente toman un array de atributos y un array de elementos hijos como argumentos. Sin embargo, los nombres que acaban con una comilla (como `ul'` aquí) omiten el array de atributos y usan los atributos por defecto en su lugar.

Fíjate en que como estamos simplemente manipulando estructuras de datos normales, podemos usar funciones como `map` para construir elementos más interesantes.

Una segunda función auxiliar es `formField`, que crea un `ReactElement` conteniendo una entrada de texto para un único campo del formulario:

```haskell
formField
  :: String
  -> String
  -> String
  -> (String -> Person)
  -> ReactElement
formField name hint value update =
  D.div [ P.className "form-group" ]
        [ D.label [ P.className "col-sm-2 control-label" ]
                  [ D.text name ]
        , D.div [ P.className "col-sm-3" ]
                [ D.input [ P._type "text"
                          , P.className "form-control"
                          , P.placeholder hint
                          , P.value value
                          , P.onChange (updateAppState ctx update)
                          ] []
                ]
        ]
```

De nuevo, date cuenta de que estamos componiendo elementos más interesantes a partir de elementos más simples, aplicando atributos a cada elemento sobre la marcha. Un atributo en el que debemos fijarnos aquí es el atributo `onChange` aplicado al elemento `input`. Esto es un _gestor de eventos_ (*event handler*), y se usa para actualizar el estado de la componente cuando el usuario edita el texto de nuestra caja de texto. Nuestro gestor de eventos se define usando una tercera función auxiliar, `updateAppState`:

```haskell
updateAppState
  :: forall props eff
   . ReactThis props AppState
  -> (String -> Person)
  -> Event
  -> Eff ( console :: CONSOLE
         , state :: ReactState ReadWrite
         | eff
         ) Unit
```

`updateAppState` toma una referencia a la componente del formulario de nuestro valor `ReactThis`, una función para actualizar el registro `Person`, y el registro `Event` al que respondemos. Primero, extrae el nuevo valor de la caja de texto del evento `change` (usando la función auxiliar `valueOf`), y lo usa para crear un nuevo estado `Person`:

```haskell
  for_ (valueOf e) \s -> do
    let newPerson = update s
```

Entonces ejecuta la función de validación y actualiza el estado de la componente (usando `writeState`) en consecuencia:

```haskell
    log "Running validators"
    case validatePerson' newPerson of
      Left errors ->
        writeState ctx (AppState { person: newPerson
                                 , errors: errors
                                 })
      Right _ ->
        writeState ctx (AppState { person: newPerson
                                 , errors: []
                                 })
```

Eso cubre lo esencial de la implementación de nuestra componente. Sin embargo, debes leer el código fuente que acompaña a este capítulo para obtener una comprensión completa de la forma en que funciona la componente.

Prueba también la interfaz de usuario ejecutando `pulp browserify --to dist/Main.js` y abriendo el fichero `html/index.html` en tu navegador. Debes ser capaz de introducir algunos valores en los campos del formulario y ver los errores de validación impresos en la página.

Obviamente, esta interfaz de usuario se puede mejorar de varias maneras. Los ejercicios explorarán varias formas en que podemos hacer la aplicación más usable.

X> ## Ejercicios
X>
X> 1. (Fácil) Modifica la aplicación para que incluya una caja de texto para un número de teléfono del trabajo. 
X> 1. (Medio) En lugar de usar un elemento `ul` para mostrar los errores de validación en una lista, modifica el código para crear un `div` con el estilo `alert` para cada error.
X> 1. (Difícil, Extendido) Un problema con esta interfaz de usuario es que los errores de validación no se muestran junto al campo del formulario en que se originan. Modifica el código para solucionar este problema.
X>   
X>   _Pista_: el tipo de error devuelto por el validador debe extenderse para que indique qué campo causó el error. Puedes usar el siguiente tipo `Errors` modificado:
X>   
X>   ```haskell
X>   data Field = FirstNameField
X>              | LastNameField
X>              | StreetField
X>              | CityField
X>              | StateField
X>              | PhoneField PhoneType
X>   
X>   data ValidationError = ValidationError String Field
X>   
X>   type Errors = Array ValidationError
X>   ```
X>
X>   Necesitarás escribir una función que extrae el error de validación de un `Field` particular de la estructura `Errors`.

## Conclusión

Este capítulo ha cubierto un montón de ideas sobre gestión de efectos secundarios en PureScript:

- Hemos conocido la clase de tipos `Monad` y su conexión con la notación do.
- Hemos presentado las leyes de la mónada, y hemos visto que nos permiten transformar código escrito usando notación do.
- Hemos visto cómo las mónadas se pueden usar de manera abstracta para escribir código que funciona con diferentes tipos de efectos secundarios.
- Hemos visto cómo las mónadas son ejemplos de funtores aplicativos, cómo ambos nos permiten calcular con efectos secundarios, y las diferencias entre ambos enfoques.
- Se ha definido el concepto de efectos nativos, y hemos conocido la mónada `Eff`, que se usa para gestionar efectos secundarios nativos. 
- Hemos visto cómo la mónada `Eff` soporta efectos extensibles, y cómo múltiples tipos de efectos nativos se pueden intercalar en el mismo cálculo.
- Hemos visto como los efectos y los registros se gestionan en el sistema de familias, y la conexión entre registros extensibles y efectos extensibles.
- Hemos usado la mónada `Eff` para gestionar una variedad de efectos: generación de números aleatorios, excepciones, entrada/salida por consola, estado mutable, y manipulación del DOM usando React.

La mónada `Eff` es una herramienta fundamental en el código PureScript del mundo real. Se usará en el resto del libro para gestionar efectos secundarios y en otros casos de uso.
