# Recursividad, asociaciones (*maps*) y pliegues (*folds*)

## Objetivos del capítulo

En este capítulo, vamos a ver cómo las funciones recursivas pueden ser usadas para estructurar algoritmos. La recursividad es una técnica básica usada en la programación funcional que vamos a usar por todo el libro.

También cubriremos algunas funciones estándar de las bibliotecas estándar de PureScript. Veremos las funciones `map` y `fold`, así como algunos casos especiales útiles, como `filter` y `concatMap`.

El ejemplo motivador para este capítulo es una biblioteca de funciones para trabajar con un sistema de ficheros virtual. Aplicaremos técnicas aprendidas en este capítulo para escribir funciones que calculan propiedades de los ficheros representados por un modelo de un sistema de ficheros.

## Preparación del proyecto

El código fuente para este capítulo está contenido en los dos ficheros `src/Data/Path.purs` y `src/FileOperations.purs`.

El módulo `Data.Path` contiene un modelo de un sistema de ficheros virtual. No necesitas modificar el contenido de este módulo.

El módulo `FileOperations` contiene funciones que usan la API `Data.Path`. Las soluciones a los ejercicios se deben implementar en este fichero.

El proyecto tiene las siguientes dependencias de Bower:

- `purescript-maybe`, que define el constructor de tipo `Maybe`.
- `purescript-arrays`, que define funciones para trabajar con arrays.
- `purescript-strings`, que define funciones para trabajar con cadenas JavaScript.
- `purescript-foldable-traversable`, que define funciones para plegar arrays y otras estructuras de datos.
- `purescript-console`, que define funciones para imprimir en la consola.

## Introducción

La recursividad es una técnica importante en la programación en general, pero es particularmente común en la programación funcional porque, como veremos en este capítulo, la recursividad ayuda a reducir el estado mutable en nuestros programas.

La recursividad está estrechamente vinculada a la estrategia _divide y vencerás_: para resolver un problema para ciertas entradas, podemos descomponer las entradas en partes más pequeñas, resolver el problema sobre esas partes, y ensamblar una solución a partir de las soluciones parciales.

Veamos algunos ejemplos simples de recursividad en PureScript.

Aquí tenemos el habitual ejemplo de _función factorial_:

```haskell
fact :: Int -> Int
fact 0 = 1
fact n = n * fact (n - 1)
```

Aquí, podemos ver cómo la función factorial se calcula reduciendo el problema a un subproblema: el de calcular el factorial de un entero más pequeño. Cuando llegamos a cero, la respuesta es inmediata.

Aquí tenemos otro ejemplo común que calcula la _función de Fibonnacci_:

```haskell
fib :: Int -> Int
fib 0 = 1
fib 1 = 1
fib n = fib (n - 1) + fib (n - 2)
```

De nuevo, este problema se resuelve considerando las soluciones a subproblemas. En este caso, hay dos subproblemas, que corresponden a las expresiones `fib (n - 1)` y `fib (n - 2)`. Cuando estos dos subproblemas están resueltos, ensamblamos el resultado sumando los resultados parciales.

## Recursividad sobre arrays

¡No estamos limitados a definir funciones recursivas sobre el tipo `Int`! Veremos funciones recursivas definidas sobre una amplia variedad de tipos de datos cuando abarquemos el _ajuste de patrones_ (*pattern matching*) más tarde en el libro, pero por ahora, nos ceñiremos a arrays y números.

Igual que podemos ramificar dependiendo de si la entrada es distinta de cero, en el caso de arrays ramificaremos dependiendo de si la entrada es no-vacía. Considera esta función que calcula la longitud de un array usando recursividad:

```haskell
import Prelude

import Data.Array (null)
import Data.Array.Partial (tail)
import Partial.Unsafe (unsafePartial)

length :: forall a. Array a -> Int
length arr =
  if null arr
    then 0
    else 1 + length (unsafePartial tail arr)
```

En esta función, usamos una expresión `if .. then .. else` para ramificar basándonos en si el array está vacío. La función `null` devuelve `true` para un array vacío. Los arrays vacíos tienen una longitud de cero, y cualquier array no vacío tiene una longitud que es uno más que la longitud de su cola.

Este ejemplo es obviamente una manera muy poco práctica de encontrar la longitud de un array en JavaScript, pero debe proporcionar suficiente ayuda para permitirte completar los siguientes ejercicios:

X> ## Ejercicios
X>
X> 1. (Fácil) Escribe una función recursiva que devuelve `true` si y sólo si su entrada es un entero par.
X> 2. (Medio) Escribe una función recursiva que cuenta el número de enteros pares en un array. _Pista_: la función `unsafePartial head` (donde `head` también se importa de `Data.Array.Partial`) se puede usar para encontrar el primer elemento de un array no vacío.

## Asociaciones (*maps*)

La función `map` es un ejemplo de una función recursiva sobre arrays. Se usa para transformar los elementos de un array aplicando una función a cada uno de sus elementos. Así, cambian el _contenido_ del array, pero preserva su _forma_ (es decir, su longitud).

Cuando veamos las _clases de tipo_ más adelante en el libro, veremos que la función `map` es un ejemplo de un patron más general de funciones que preservan la forma y que transforman una clase de constructores de tipo llamados _funtores_ (*functors*).

Probemos la función `map` en PSCi:

```text
$ pulp repl

> import Prelude
> map (\n -> n + 1) [1, 2, 3, 4, 5]
[2, 3, 4, 5, 6]
```

Date cuenta de cómo se usa `map`: proporcionamos como primer argumento una función que debe ser "mapeada sobre" el array, y proporcionamos el array como segundo argumento.

## Operadores infijos

La función `map` también se puede escribir entre la función de mapeo y el array rodeándola de comillas inversas:

```text
> (\n -> n + 1) `map` [1, 2, 3, 4, 5]
[2, 3, 4, 5, 6]
```

Esta sintaxis se llama _aplicación de función infija_, y cualquier función se puede hacer infija de esta manera. Normalmente es más apropiada para funciones de dos argumentos.

Hay un operador que es equivalente a la función `map` cuando se usa con arrays, llamado `<$>`. Este operador se puede usar de manera infija como cualquier otro operador binario:

```text
> (\n -> n + 1) <$> [1, 2, 3, 4, 5]
[2, 3, 4, 5, 6]
```

Veamos el tipo de `map`:

```text
> :type map
forall a b f. Functor f => (a -> b) -> f a -> f b
```

El tipo de `map` es de hecho más general de lo que necesitamos en este capítulo. Para nuestros propósitos, podemos tratar `map` como si tuviese el siguiente tipo menos general:

```text
forall a b. (a -> b) -> Array a -> Array b
```

Este tipo dice que podemos elegir dos tipos cualesquiera, `a` y `b`, con los que aplicar la función `map`. `a` es el tipo de elementos del array original y `b` es el tipo de elementos del array resultante. En particular, no hay ninguna razón por la que `map` tenga que preservar el tipo de los elementos del array. Podemos usar `map` o `<$>` para transformar enteros a cadenas, por ejemplo:

```text
> show <$> [1, 2, 3, 4, 5]

["1","2","3","4","5"]
```

Aunque el operador infijo `<$>` parece una sintaxis especial, es de hecho un simple apodo (*alias*) para una función PureScript normal. La función es simplemente _aplicada_ usando notación infija. De hecho, la función se puede usar como una función normal poniendo el nombre entre paréntesis. Esto significa que podemos usar el nombre entre paréntesis `(<$>)` en lugar de `map` sobre arrays:

```text
> (<$>) show [1, 2, 3, 4, 5]
["1","2","3","4","5"]
```

Los nombres de función infijos se definen como _apodos_ para nombres de función existentes. Por ejemplo, el módulo `Data.Array` define un operador infijo `(..)` como sinónimo de la funcion `range` como sigue:

```haskell
infix 8 range as ..
```

Podemos usar este operador como sigue:

```text
> import Data.Array

> 1 .. 5
[1, 2, 3, 4, 5]

> show <$> (1 .. 5)
["1","2","3","4","5"]
```

_Nota_: Los operadores infijos pueden ser una gran herramienta para definir lenguajes específicos del dominio con una sintaxis natural. Sin embargo, si se usan excesivamente, pueden volver el código ilegible para principiantes, de manera que es una buena cosa tener precaución al definir cualquier operador nuevo.

En el ejemplo anterior, hemos puesto la expresión `1 .. 5` entre paréntesis, pero de hecho no era necesario, porque el módulo `Data.Array` asigna un nivel de precedencia mayor al operador `..` que el asignado al operador `<$>`. En el ejemplo anterior, la precedencia del operador `..` se definía como `8`, el número tras la palabra clave `infix`. Este valor es más alto que el nivel de precedencia de `<$>`, lo que significa que no necesitamos añadir paréntesis:

```text
> show <$> 1 .. 5
["1","2","3","4","5"]
```

Si quisiésemos asignar una _asociatividad_ (a izquierda o derecha) a un operador infijo, podríamos hacerlo con las palabras clave `infixl` o `infixr`.

## Filtrando arrays

El módulo `Data.Array` proporciona otra función `filter` que se usa a menudo junto a `map`. Proporciona la capacidad de crear un nuevo array, a partir de uno existente, manteniendo únicamente los elementos que coinciden con una función predicado.

Por ejemplo, supongamos que queremos calcular un array de todos los números pares entre 1 y 10. Lo podríamos hacer como sigue:

```text
> import Data.Array

> filter (\n -> n `mod` 2 == 0) (1 .. 10)
[2,4,6,8,10]
```

X> ## Ejercicios
X>
X> 1. (Fácil) Usa la función `map` o `<$>` para escribir una función que calcula los cuadrados de un array de números.
X> 1. (Fácil) Usa la función `filter` para escribir una función que quita los números negativos de un array de números.
X> 1. (Medio) Define un sinónimo infijo `<$?>` para `filter`. Reescribe tu respuesta al ejercicio anterior usando el nuevo operador. Experimenta con el nivel de precedencia y asociatividad de tu operador en PSCi.

## Aplanando arrays

Otra función estándar sobre arrays es la función `concat`, definida en `Data.Array`. `concat` aplana un array de arrays en un único array:

```text
> import Data.Array

> :type concat
forall a. Array (Array a) -> Array a

> concat [[1, 2, 3], [4, 5], [6]]
[1, 2, 3, 4, 5, 6]
```

Hay una función relacionada llamada `concatMap` que es como una combinación de las funciones `concat` y `map`. Si `map` toma una función de valores a valores (posiblemente de un tipo diferente), `concatMap` toma una función de valores a arrays de valores.

Veámosla en acción:

```text
> import Data.Array

> :type concatMap
forall a b. (a -> Array b) -> Array a -> Array b

> concatMap (\n -> [n, n * n]) (1 .. 5)
[1,1,2,4,3,9,4,16,5,25]
```

Aquí llamamos `concatMap` con la función `\n -> [n, n * n]` que envía un entero al array de dos elementos consistente en el propio entero y su cuadrado. El resultado es un array de diez enteros: los enteros de 1 a 5 junto a sus cuadrados.

Date cuenta que `concatMap` concatena sus resultados. Llama a la función proporcionada una vez para cada elemento del array original, generando un array para cada uno. Finalmente, colapsa todos estos arrays en un único array que será su resultado.

`map`, `filter` y `concatMap` forman la base de todo un rango de funciones sobre arrays llamadas `arrays por comprensión`.

## Arrays por comprensión (*array comprehensions*)

Supongamos que queremos encontrar los factores de un número `n`. Una forma simple de hacerlo sería por fuerza bruta: podemos generar todos los pares de números entre 1 y `n`, y tratar de multiplicarlos. Si el producto fuese `n`, habríamos encontrado un par de factores de `n`.

Podemos realizar este cálculo usando un array por comprensión. Lo haremos por pasos, usando PSCi como nuestro entorno de desarrollo interactivo.

El primer paso es generar un array de pares de números inferiores a `n`, cosa que podemos hacer usando `concatMap`.

Empecemos mapeando cada número al array `1 .. n`:

```text
> pairs n = concatMap (\i -> 1 .. n) (1 .. n)
```

Podemos probar nuestra función:

```text
> pairs 3
[1,2,3,1,2,3,1,2,3]
```

Esto no es exactamente lo que queremos. En lugar de simplemente devolver el segundo elemento de cada par, necesitamos mapear una función sobre la copia interna de `1 .. n` que nos permitirá mantener el par completo:

```text
> :paste
… pairs' n =
…   concatMap (\i ->
…     map (\j -> [i, j]) (1 .. n)
…   ) (1 .. n)
… ^D

> pairs' 3
[[1,1],[1,2],[1,3],[2,1],[2,2],[2,3],[3,1],[3,2],[3,3]]
```

Esto tiene mejor pinta. Sin embargo, estamos generando demasiados pares: tenemos [1, 2] y [2, 1] por ejemplo. Podemos excluir el segundo caso asegurándonos de que `j` sólo va de `i` a `n`:

```text
> :paste
… pairs'' n =
…   concatMap (\i ->
…     map (\j -> [i, j]) (i .. n)
…   ) (1 .. n)
… ^D
> pairs'' 3
[[1,1],[1,2],[1,3],[2,2],[2,3],[3,3]]
```

¡Estupendo! Ahora que tenemos todos los pares de factores potenciales, podemos usar `filter` para elegir los pares cuya multiplicación da `n`:

```text
> import Data.Foldable

> factors n = filter (\pair -> product pair == n) (pairs'' n)

> factors 10
[[1,10],[2,5]]
```

Este código usa la función `product` del módulo `Data.Foldable` en la biblioteca `purescript-foldable-traversable`.

¡Excelente! Hemos conseguido encontrar el conjunto correcto de pares de factores sin duplicados.

## Notación 'do' (*do notation*)

Sin embargo, podemos mejorar la legibilidad de nuestro código considerablemente. `map` y `concatMap` son tan fundamentales que forman la base (o más bien, sus generalizaciones `map` y `bind` forman la base) de una sintaxis especial llamada _notación do_.

_Nota_: Igual que `map` y `concatMap` nos permitían escribir _arrays por comprensión_, los operadores más generales `map` y `bind` nos permiten escribir las llamadas _mónadas por comprensión_ (*monad comprehensions*). Veremos muchos mas ejemplos de _mónadas_ mas adelante, pero en este capítulo vamos a considerar únicamente arrays.

Podemos reescribir nuestra función `factors` usando notación do como sigue:

```haskell
factors :: Int -> Array (Array Int)
factors n = filter (\xs -> product xs == n) $ do
  i <- 1 .. n
  j <- i .. n
  pure [i, j]
```

La palabra clave `do` comienza un bloque de código que usa notación do. El bloque consiste de expresiones de estos tipos:

- Expresiones que ligan elementos de un array a un nombre. Estas se indican con la flecha apuntando hacia atrás `<-`, con un nombre a la izquierda y una expresión a la derecha de tipo array.
- Expresiones que no ligan elementos del array a nombres. La última línea `pure [i, j]` es un ejemplo de este tipo de expresión.
- Expresiones que dan nombre a expresiones, usando la palabra clave `let`.

Con suerte, esta nueva notación hará la estructura del algoritmo más clara. Si reemplazas mentalmente la flecha `<-` con la palabra "elige", puedes leerlo como sigue: "elige un elemento `i` entre 1 y `n`, luego elige un elemento `j` entre `i` y `n`, y finalmente devuelve `[i, j]`".

En la última línea usamos la función `pure`. Esta función puede ser evaluada en PSCi, pero tenemos que proporcionar un tipo:

```text
> pure [1, 2] :: Array (Array Int)
[[1, 2]]
```

En el caso de arrays, `pure` simplemente construye un array de un único elemento. De hecho, podemos modificar nuestra función `factors` para que use esta forma en lugar de usar `pure`:

```haskell
factors :: Int -> Array (Array Int)
factors n = filter (\xs -> product xs == n) $ do
  i <- 1 .. n
  j <- i .. n
  [[i, j]]
```

El resultado será el mismo.

## Guardas (*guards*)

Otra mejora que podemos hacer a la función `factors` es poner el filtro dentro del array por comprensión. Esto es posible usando la función `guard` del módulo `Control.MonadZero` (del paquete `purescript-control`):

```haskell
import Control.MonadZero (guard)

factors :: Int -> Array (Array Int)
factors n = do
  i <- 1 .. n
  j <- i .. n
  guard $ i * j == n
  pure [i, j]
```

Al igual que `pure`, podemos aplicar la función `guard` en PSCi para entender cómo funciona. El tipo de la función `guard` es más general de lo que necesitamos aquí:

```text
> import Control.MonadZero

> :type guard
forall m. MonadZero m => Boolean -> m Unit
```

En nuestro caso, podemos asumir que PSCi reportó el siguiente tipo:

```haskell
Boolean -> Array Unit
```

Para nuestros propósitos, los siguientes cálculos nos dicen todo lo que necesitamos saber de la función `guard` sobre arrays:

```text
> import Data.Array

> length $ guard true
1

> length $ guard false
0
```

Esto es, si a `guard` se le pasa una expresión que evalúa a `true`, devuelve un array con un único elemento. Si la expresión devuelve `false`, su resultado está vacío.

Esto significa que si la guarda falla, la rama actual del array por comprensión terminará de manera temprana sin resultados. Lo que significa que una llamada a `guard` es equivalente a usar `filter` en el array intermedio. Dependiendo de la aplicación, puede que prefieras usar `guard` en lugar de `filter`. Prueba las dos definiciones de `factors` para verificar que dan el mismo resultado.

X> ## Ejercicios
X>
X> 1. (Fácil) Usa la función `factors` para definir una función `isPrime` que comprueba si su argumento entero es primo o no.
X> 1. (Medio) Escribe una función usando notación do para encontrar el _producto cartesiano_ de dos arrays, es decir, el conjunto de pares de elementos `a`, `b`, donde `a` es un elemento del primer array y `b` es un elemento del segundo.
X> 1. (Medio) Una _terna pitagórica_ es un array de números `[a, b, c]` tales que `a² + b² = c²`. Usa la función `guard` en un array por comprensión para escribir una función `triples` que toma un número `n` y calcula todas las ternas pitagóricas cuyos componentes sean inferiores a `n`. Tu función debe tener el tipo `Int -> Array (Array Int)`.
X> 1. (Difícil) Escribe una función `factorizations` que produce todas las _factorizaciones_ de un entero `n`, es decir, arrays de enteros cuyo producto es `n`. _Pista_: para un entero mayor que 1, parte el problema en dos subproblemas: encontrar el primer factor y encontrar los factores restantes.

## Pliegues (folds)

Los pliegues por la izquierda y por la derecha sobre arrays proporcionan otra clase de funciones interesantes que se pueden implementar usando recursividad.

Comienza importando el módulo `Data.Foldable` e inspecciona los tipos de las funciones `foldl` y `foldr` usando PSCi

```text
> import Data.Foldable

> :type foldl
forall a b f. Foldable f => (b -> a -> b) -> b -> f a -> b

> :type foldr
forall a b f. Foldable f => (a -> b -> b) -> b -> f a -> b
```

Estos tipos son más necesarios de lo que nos interesa por ahora. Para los propósitos de este capítulo, podemos asumir que PSCi nos da la siguiente respuesta (más específica):

```text
> :type foldl
forall a b. (b -> a -> b) -> b -> Array a -> b

> :type foldr
forall a b. (a -> b -> b) -> b -> Array a -> b
```

En ambos casos, el tipo `a` corresponde a los tipos de los elementos de nuestro array. El tipo `b` se puede considerar como el tipo de un "acumulador", que acumulará un resultado según recorremos el array.

La diferencia entre las funciones `foldl` y `foldr` es la dirección del recorrido. `foldl` pliega el array "desde la izquierda", mientras que `foldr` lo pliega "desde la derecha".

Veamos estas funciones en acción. Usemos `foldl` para sumar un array de enteros. El tipo `a` será `Int`, y podemos también elegir que el tipo resultante `b` sea `Int`. Necesitamos proporcionar tres argumentos: una función `Int -> Int -> Int` que sumará el siguiente elemento al acumulador, un valor inicial para el acumulador de tipo `Int`, y un array de `Int` a sumar. Para el primer argumento, podemos simplemente usar el operador de suma, y el valor inicial del acumulador será cero:

```text
> foldl (+) 0 (1 .. 5)
15
```

En este caso, no importa si usamos `foldl` o `foldr` ya que el resultado es el mismo, no importa el orden en que sucedan las sumas:

```text
> foldr (+) 0 (1 .. 5)
15
```

Escribamos un ejemplo donde la elección de la función de pliegue importa para ilustrar la diferencia. En lugar de la función de suma, usemos la concatenación de cadenas para construir una cadena:

```text
> foldl (\acc n -> acc <> show n) "" [1,2,3,4,5]
"12345"

> foldr (\n acc -> acc <> show n) "" [1,2,3,4,5]
"54321"
```

Esto ilustra la diferencia entre ambas funciones. La expresión de pliegue por la izquierda es equivalente a la siguiente aplicación:

```text
((((("" <> show 1) <> show 2) <> show 3) <> show 4) <> show 5)
```

Mientras que el pliegue por la derecha es equivalente a esto:

```text
((((("" <> show 5) <> show 4) <> show 3) <> show 2) <> show 1)
```

## Recursividad final (*tail recursion*)

La recursividad es una técnica potente para especificar algoritmos, pero tiene un problema: evaluar funciones recursivas en JavaScript puede llevar a errores de desbordamiento de pila si las entradas son demasiado grandes.

Es fácil verificar el problema con el siguiente código en PSCi:

```text
> f 0 = 0
> f n = 1 + f (n - 1)

> f 10
10

> f 100000
RangeError: Maximum call stack size exceeded
```

Esto es un problema. Si vamos a adoptar la recursividad como una técnica estándar de la programación funcional, necesitamos una forma de tratar con la recursividad posiblemente infinita.

PureScript proporciona una solución parcial a este problema en la forma de _optimización de recursividad final_ (*tail recursion optimization*).

_Nota_: se pueden implementar soluciones al problema más completas en bibliotecas usando el llamado _trampolining_, pero eso está fuera del ámbito de este capítulo. El lector interesado puede consultar la documentación de los paquetes `purescript-free` y `purescript-tailrec`.

La observación clave que permite la optimización de la recursividad final es la siguiente: una llamada recursiva en una _posición de cola_ (*tail position*) a una función se puede reemplazar por un _salto_, que no reserva una trama de pila. Una llamada está en _posición de cola_ cuando es la última llamada hecha antes de que la función retorne. Esta es la razón por la que hemos observado un desbordamiento de pila en el ejemplo - la llamada recursiva a `f` _no_ estaba en posición de cola.

En la práctica, el compilador de PureScript no sustituye la llamada recursiva por un salto, en su lugar sustituye la función recursiva por un _bucle while_.

Aquí hay un ejemplo de una función recursiva con todas las llamadas recursivas en posición de cola:

```haskell
fact :: Int -> Int -> Int
fact 0 acc = acc
fact n acc = fact (n - 1) (acc * n)
```

Date cuenta de que la llamada recursiva a `fact` es lo último que sucede en esta función - está en posición de cola.

## Acumuladores (*accumulators*)

Una forma común de convertir una función que no es recursiva final en una función recursiva final es usar un _parámetro acumulador_. Un parámetro acumulador es un parámetro adicional que se añade a una función para _acumular_ un valor de retorno, en contraposición a usar el valor de retorno para acumular el resultado.

Por ejemplo, considera esta recursividad de array que invierte el array de entrada añadiendo elementos de la cabeza del array de entrada al final del resultado:

```haskell
reverse :: forall a. Array a -> Array a
reverse [] = []
reverse xs = snoc (reverse (unsafePartial tail xs))
                  (unsafePartial head xs)
```

Esta implementación no es recursiva final, de manera que el JavaScript generado provocará un desbordamiento de pila cuando se ejecute sobre un array de entrada grande. Sin embargo, podemos hacerla recursiva final introduciendo un segundo argumento para acumular el resultado:

```haskell
reverse :: forall a. Array a -> Array a
reverse = reverse' []
  where
    reverse' acc [] = acc
    reverse' acc xs = reverse' (unsafePartial head xs : acc)
                               (unsafePartial tail xs)
```

En este caso, delegamos a la función auxiliar `reverse'`, que realiza la tarea pesada de invertir el array. Date cuenta de que la función `reverse'` es recursiva final. Su única llamada recursiva está en el último caso y está en posición de cola. Esto significa que el código generado será un _bucle while_ y no desbordará la pila para entradas grandes.

Para entender la segunda implementación de `reverse`, date cuenta de que la función auxiliar `reverse'` usa esencialmente el parámetro acumulador para mantener un trozo adicional de estado, el resultado parcialmente construido. El resultado comienza vacío y crece en un elemento para cada elemento del array de entrada. Sin embargo, dado que los elementos posteriores se añaden al principio del array, el resultado es el array original invertido.

Date cuenta también de que mientras que podemos considerar el acumulador como "estado", no sucede una mutación directa. El acumulador es un array inmutable y simplemente usamos argumentos de función para hilar el estado durante el cálculo.

## Prefiere pliegues a recursividad explícita

Si podemos escribir nuestras funciones recursivas usando recursividad final podemos beneficiarnos de las optimizaciones de recursividad final, de manera que parece tentador intentar escribir todas nuestras funciones de esta forma. Sin embargo, es fácil olvidar que muchas funciones se puedes escribir directamente como un pliegue sobre un array o una estructura de datos similar. Escribir algoritmos directamente en términos de combinadores como `map` y `fold` tiene la ventaja de la simplicidad del código. Estos combinadores se comprenden bien y, de esta manera, comunican la _intención_ del algoritmo mucho mejor que la recursividad explícita.

Por ejemplo, el ejemplo `reverse` se puede escribir como un pliegue al menos de dos maneras. Aquí hay una versión que usa `foldr`:

```text
> import Data.Foldable

> :paste
… reverse :: forall a. Array a -> Array a
… reverse = foldr (\x xs -> xs <> [x]) []
… ^D

> reverse [1, 2, 3]
[3,2,1]
```

Escribir `reverse` en términos de `foldl` se deja como ejercicio para el lector.

X> ## Ejercicios
X>
X> 1. (Fácil) Usa `foldl` para comprobar si todos los elementos de un array de valores booleanos son true.
X> 2. (Medio) Identifica los arrays `xs` para los que la función `foldl (==) false xs` devuelve true.
X> 3. (Medio) Reescribe la siguiente función en forma recursiva final usando un parámetro acumulador:
X>
X>     ```haskell
X>     import Prelude
X>     import Data.Array.Partial (head, tail)
X>     
X>     count :: forall a. (a -> Boolean) -> Array a -> Int
X>     count _ [] = 0
X>     count p xs = if p (unsafePartial head xs)
X>                    then count p (unsafePartial tail xs) + 1
X>                    else count p (unsafePartial tail xs)
X>     ```
X>
X> 4. (Medio) Escribe `reverse` en términos de `foldl`.

## Un sistema de ficheros virtual

En esta sección, vamos a aplicar lo que hemos aprendido escribiendo funciones para trabajar sobre un modelo de un sistema de ficheros. Usaremos asociaciones, pliegues y filtros para trabajar con un API predefinido.

El módulo `Data.Path` define un API para un sistema de ficheros virtual como sigue:

- Hay un tipo `Path` que representa rutas en el sistema de ficheros.
- Hay una ruta `root` que representa el directorio raíz.
- La función `ls` enumera los ficheros de un directorio.
- La función `filename` devuelve el nombre de fichero para un `Path`.
- La función `size` devuelve el tamaño de fichero para un `Path` que representa un fichero.
- La función `isDirectory` comprueba si un `Path` es un fichero o un directorio.

En términos de tipos, tenemos las siguientes definiciones de tipos:

```haskell
root :: Path

ls :: Path -> Array Path

filename :: Path -> String

size :: Path -> Maybe Number

isDirectory :: Path -> Boolean
```

Podemos probar la API en PSCi:

```text
$ pulp repl

> import Data.Path

> root
/

> isDirectory root
true

> ls root
[/bin/,/etc/,/home/]
```

El módulo `FileOperations` define funciones que usan la API de `Data.Path`. No necesitas modificar el módulo `Data.Path` o entender su implementación. Trabajaremos en el módulo `FileOperations`.

## Listando todos los ficheros

Escribamos una función que realiza una enumeración profunda de todos los ficheros dentro de un directorio. La función tendrá el siguiente tipo:

```haskell
allFiles :: Path -> Array Path
```

Podemos definir esta función mediante recursividad. Primero, podemos usar `ls` para enumerar los hijos inmediatos de un directorio. Para cada hijo, podemos aplicar recursivamente `allFiles`, que devolverá un array de rutas. `concatMap` nos permitirá aplicar `allFiles` y aplanar los resultados al mismo tiempo.

Finalmente, podemos usar el operador cons `:` para incluir el fichero actual:

```haskell
allFiles file = file : concatMap allFiles (ls file)
```

_Nota_: El operador cons `:` tiene de hecho un rendimiento pobre en arrays inmutables, de manera que en general no se recomienda. El rendimiento se puede mejorar usando otras estructuras de datos como listas enlazadas y secuencias.

Probemos esta función en PSCi:

```text
> import FileOperations
> import Data.Path

> allFiles root

[/,/bin/,/bin/cp,/bin/ls,/bin/mv,/etc/,/etc/hosts, ...]
```

¡Estupendo! Ahora veamos si podemos escribir esta función usando un array por comprensión usando notación do.

Recuerda que una flecha hacia atrás corresponde a elegir un elemento de un array. El primer paso es elegir el elemento de los hijos inmediatos del argumento. A continuación, llamamos a la función recursivamente para ese fichero. Como estamos usando notación do, hay una llamada implícita a `concatMap` que concatena todos los resultados recursivos.

Aquí está la nueva versión:

```haskell
allFiles' :: Path -> Array Path
allFiles' file = file : do
  child <- ls file
  allFiles' child
```

Prueba la nueva versión en PSCi. Debes obtener el mismo resultado. Te dejo decidir qué versión ves más clara.

X> ## Ejercicios
X>
X> 1. (Fácil) Escribe una función `onlyFiles` que devuelve todos los _ficheros_ (no directorios) en todos los subdirectorios de un directorio.
X> 1. (Medio) Escribe un pliege para determinar el fichero más grande y el más pequeño del sistema de ficheros.
X> 1. (Difícil) Escribe una función `whereIs` para buscar un fichero por nombre. La función debe devolver un valor de tipo `Maybe Path`, indicando el directorio que contiene el fichero si existe. Debe comportarse como sigue:
X>
X>     ```text
X>     > whereIs "/bin/ls"
X>     Just (/bin/)
X>     
X>     > whereIs "/bin/cat"
X>     Nothing
X>     ```
X>
X>     _Pista_: Intenta escribir esta función como un array por comprensión usando notación do.

## Conclusión

En este capítulo hemos cubierto las bases de la recursividad en PureScript como una manera de expresar algoritmos de manera concisa. También hemos introducido los operadores infijos definidos por el usuario, funciones estándar sobre arrays como asociaciones, filtros y pliegues, y los arrays por comprensión que combinan estas ideas. Finalmente, hemos mostrado la importancia de la recursividad final para evitar errores de desbordamiento de pila, y como usar parámetros acumuladores para convertir funciones a forma recursiva final.
