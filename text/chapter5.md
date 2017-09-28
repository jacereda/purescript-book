# Ajuste de patrones (*pattern matching*)

## Objetivos del capítulo

Este capítulo presentará dos nuevos conceptos: tipos de datos algebraicos (*algebraic data types*) y ajuste de patrones. También cubriremos brevemente una interesante característica del sistema de tipos de PureScript: polimorfismo de fila (*row polymorphism*).

El ajuste de patrones es una técnica común en la programación funcional y permite al desarrollador escribir funciones compactas que expresan ideas potencialmente complejas partiendo su implementación en múltiples casos.

Los tipos de datos algebraicos son una característica del sistema de tipos de PureScript que permiten un nivel similar de expresividad en el lenguaje de los tipos; están estrechamente relacionados con el ajuste de patrones.

El objetivo del capítulo será escribir una biblioteca para describir y manipular gráficos vectoriales simples usando tipos de datos algebraicos y ajuste de patrones.

## Preparación del proyecto

El código fuente de este capítulo está definido en el fichero `src/Data/Picture.purs`. 

El proyecto usa algunos paquetes de Bower que ya hemos visto y añade las siguientes dependencias nuevas:

- `purescript-globals`, que proporciona acceso a algunos valores y funciones comunes en JavaScript.
- `purescript-math`, que proporciona acceso al modulo `Math` de JavaScript.

El módulo `Data.Picture` define un tipo de datos `Shape` para formas simples y un tipo `Picture` para colecciones de formas, junto a funciones para trabajar con esos tipos.

El módulo importa el módulo `Data.Foldable`, que proporciona funciones para plegar estructuras de datos:

```haskell
module Data.Picture where

import Prelude
import Data.Foldable (foldl)
```

El módulo `Data.Picture` también importa los módulos `Global` y `Math`, pero esta vez usando la palabra reservada `as`:

```haskell
import Global as Global
import Math as Math
```

Esto deja disponibles los tipos y funciones de esos módulos, pero sólo usando _nombres cualificados_ (*qualified names*), como `Global.infinity` y `Math.max`. Esto puede ser útil para evitar importaciones superpuestas o simplemente para dejar claro de qué modulo se importan ciertas cosas. 

_Nota_: no es necesario usar el mismo nombre de módulo que el original en un import cualificado. Nombres cualificados más cortos como `import Math as M` son posibles y bastante comunes.

## Ajuste de patrones simple

Comencemos viendo un ejemplo. Aquí hay una función que calcula el máximo común divisor de dos enteros usando ajuste de patrones:

```haskell
gcd :: Int -> Int -> Int
gcd n 0 = n
gcd 0 m = m
gcd n m = if n > m
            then gcd (n - m) m
            else gcd n (m - n)
```

Este algoritmo se llama Algoritmo de Euclides. Si buscas su definición, probablemente encontrarás un conjunto que ecuaciones matemáticas que se parecen bastante al código de arriba. Este es uno de los beneficios del ajuste de patrones: te permite definir el código por casos, escribiendo código simple y declarativo que parece una especificación de una función matemática.

Una función escrita usando ajuste de patrones funciona emparejando conjuntos de condiciones con sus resultados. Cada línea se llama _alternativa_ o _caso_. Las expresiones a la izquierda del signo igual se llaman _patrones_ (*patterns*), y cada caso consiste en uno o más patrones separados por espacios. Los casos describen qué condiciones deben satisfacer los argumentos antes de que se evalúe y devuelva la expresión a la derecha del signo igual. Cada caso se intenta por orden y el primer caso cuyos patrones se ajusten a sus entradas determina el valor de retorno.

Por ejemplo, la función `gcd` se evalúa usando los siguientes pasos:

- Se prueba con el primer caso: si el segundo argumento es cero, la función devuelve `n` (el primer argumento).
- Si no, se prueba con el segundo caso: si el primer argumento es cero, la función devuelve `m` (el segundo argumento).
- De otra manera, la función se evalúa y devuelve la expresión de la última línea.

Date cuenta de que los patrones pueden ligar valores a nombres; cada línea en el ejemplo liga uno o dos de los nombres `n` y `m` a sus valores de entrada. Según vayamos aprendiendo distintos tipos de patrones veremos que diferentes tipos de patrones corresponden a diferentes formas de elegir nombres para los argumentos de entrada.

## Patrones simples

El ejemplo de código anterior demuestra dos tipos de patrones:

- Patrones enteros literales, que se ajustan a algo de tipo `Int` sólo si el valor coincide exactamente.
- Patrones variables, que ligan su argumento a un nombre.

Hay otros tipos de patrones simples:

- Literales `Number`, `String`, `Char` y `Boolean`.
- Patrones comodín (*wildcard patterns*), indicados mediante un subrayado (`_`), que van a ajustarse a cualquier argumento y que no ligan ningún nombre.

Aquí tenemos dos ejemplos más que demuestran el uso de estos patrones simples:

```haskell
fromString :: String -> Boolean
fromString "true" = true
fromString _      = false

toString :: Boolean -> String
toString true  = "true"
toString false = "false"
```

Prueba estas funciones en PSCi.

## Guardas

En el ejemplo del Algoritmo de Euclides, hemos usado una expresión `if .. then .. else` para decidir entre las dos alternativas `m > n` y `m <= n`. Otra opción en este caso sería usar una _guarda_.

Una guarda es una expresión de valor booleano que debe ser satisfecha junto a las restricciones impuestas por los patrones. Aquí esta el Algoritmo de Euclides reescrito para usar una guarda:

```haskell
gcd :: Int -> Int -> Int
gcd n 0 = n
gcd 0 n = n
gcd n m | n > m     = gcd (n - m) m
        | otherwise = gcd n (m - n)
```

En este caso, la tercera línea usa una guarda para imponer la condición extra de que el primer argumento es estrictamente mayor que el segundo.

Como este ejemplo muestra, las guardas aparecen a la izquierda del símbolo igual, separadas de la lista de patrones por un carácter barra (`|`).

X> ## Ejercicios
X>
X> 1. (Fácil) Escribe la función factorial usando ajuste de patrones. _Pista_: considera los dos casos donde la entrada es cero y distinta de cero.
X> 1. (Medio) Busca la _Regla de Pascal_ para calcular coeficientes binomiales. Úsala para escribir una función que calcula coeficientes binomiales usando ajuste de patrones.

## Patrones de array (*array patterns*)

Los _patrones de array literales_ proporcionan una forma de ajustarse a arrays de una longitud fija. Por ejemplo, supongamos que queremos escribir una función `isEmpty` que identifica arrays vacíos. Podríamos hacer esto usando un patrón de array vacío (`[]`) en la primera alternativa: 

```haskell
isEmpty :: forall a. Array a -> Boolean
isEmpty [] = true
isEmpty _ = false
```

Aquí tenemos otra función que se ajusta a arrays de longitud 5, ligando cada uno de sus cinco argumentos de distinta manera:

```haskell
takeFive :: Array Int -> Int
takeFive [0, 1, a, b, _] = a * b
takeFive _ = 0
```

El primer patrón sólo se ajusta a arrays de cinco elementos, cuyo primer y segundo elemento son 0 y 1 respectivamente. En ese caso, la función devuelve el producto del tercer y cuarto elemento. En cualquier otro caso, la función devuelve cero. Por ejemplo, en PSCi:

```text
> :paste
… takeFive [0, 1, a, b, _] = a * b
… takeFive _ = 0
… ^D

> takeFive [0, 1, 2, 3, 4]
6

> takeFive [1, 2, 3, 4, 5]
0

> takeFive []
0
```

Los patrones de array literales nos permiten ajustar arrays de una longitud fija, pero PureScript _no_ proporciona ningún medio para ajustar arrays de una longitud no especificada, ya que descomponer arrays inmutables de esta manera resulta en rendimiento pobre. Si necesitas una estructura de datos que soporte este tipo de ajuste, la manera recomendada es usar `Data.List`. Existen otras estructuras de datos que proporcionan rendimiento asintótico mejorado para distintas operaciones.

## Patrones de registro (*record patterns*) y polimorfismo de fila (*row polymorphism*)

Los _patrones de registro_ se usan para ajustar (lo has adivinado) registros.

Los patrones de registro tienen el mismo aspecto que los registros literales, pero en lugar de especificar valores a la derecha de los dos puntos, especificamos un símbolo a ligar para cada campo.

Por ejemplo: este patrón se ajusta a cualquier registro que contenga campos llamados `first` y `last`, y liga sus valores a los nombres `x` e `y` respectivamente:

```haskell
showPerson :: { first :: String, last :: String } -> String
showPerson { first: x, last: y } = y <> ", " <> x
```

Los patrones de registro proporcionan un buen ejemplo de una característica interesante del sistema de tipos de PureScript: _polimorfismo de fila_. Supongamos que hemos definido `showPerson` sin la firma de tipos de arriba. ¿Cuál sería su tipo inferido? Curiosamente, no es el mismo tipo que le dimos:

```text
> showPerson { first: x, last: y } = y <> ", " <> x

> :type showPerson
forall r. { first :: String, last :: String | r } -> String
```

¿Qué es la variable de tipo `r` aquí? Bien, si probamos `showPerson` en PSCi vemos algo interesante:

```text
> showPerson { first: "Phil", last: "Freeman" }
"Freeman, Phil"

> showPerson { first: "Phil", last: "Freeman", location: "Los Angeles" }
"Freeman, Phil"
```

Podemos añadir campos adicionales al registro y la función `showPerson` sigue funcionando. Siempre y cuando el registro contenga los campos `first` y `last` de tipo `String`, la aplicación de función está bien tipada. Sin embargo, _no_ es válido llamar a `showPerson` con _menos_ campos:

```text
> showPerson { first: "Phil" }

Type of expression lacks required label "last"
```

Podemos leer la nueva firma de tipos de `showPerson` como "toma cualquier registro con campos `first` y `last` que son de tipo `String` _y cualquier otro campo_, y devuelve un `String`".

Esta función es polimórfica en la _fila_ `r` de los campos del registro, de ahí el nombre _polimorfismo de fila_.

Date cuenta de que también podríamos haber escrito: 

```haskell
> showPerson p = p.last <> ", " <> p.first
```

y PSCi habría inferido el mismo tipo.

Veremos el polimorfismo de fila de nuevo más tarde cuando veamos los _efectos extensibles_.

## Patrones anidados (*nested patterns*)

Tanto los patrones de array como los patrones de registro combinan patrones más pequeños para construir patrones más grandes. Mayormente, los ejemplos anteriores sólo han usado patrones simples dentro de patrones array y registro, pero es importante notar que los patrones se pueden _anidar_ arbitrariamente, lo que permite definir funciones usando condiciones en tipos de datos potencialmente complejos. 

Por ejemplo, este código combina dos patrones de registro:

```haskell
type Address = { street :: String, city :: String }

type Person = { name :: String, address :: Address }

livesInLA :: Person -> Boolean
livesInLA { address: { city: "Los Angeles" } } = true
livesInLA _ = false
```

## Patrones nombrados (*named patterns*)

Los patrones pueden ser _nombrados_ para traer nombres adicionales al contexto cuando se usan patrones anidados. Cualquier patrón puede nombrarse usando el símbolo `@`.

Por ejemplo, esta función ordena arrays de dos elementos, nombrando los dos elementos, pero también nombrando el propio array: 

```haskell
sortPair :: Array Int -> Array Int
sortPair arr@[x, y]
  | x <= y = arr
  | otherwise = [y, x]
sortPair arr = arr
```

De esta manera, nos ahorramos reservar un nuevo array si el par está ya ordenado.

X> ## Ejercicios
X>
X> 1. (Fácil) Escribe una función `sameCity` que usa patrones de registro para comprobar si dos registros `Person` pertenecen a la misma ciudad.
X> 1. (Medio) ¿Cuál es el tipo más general de la función `sameCity` teniendo en cuenta el polimorfismo de fila? ¿Y para la función `livesInLA` definida antes?
X> 1. (Medio) Escribe una función `fromSingleton` que usa un patrón de array literal para extraer el único miembro de un array singleton. Si el array no es un singleton, tu función debe devolver un valor por defecto proporcionado. Tu función debe tener el tipo `forall a. a -> Array a -> a`

## Expresiones "case" (*case expressions*)

Los patrones no sólo aparecen en las declaraciones de función de nivel superior. Es posible usar patrones para ajustarse a un valor intermedio de un cálculo usando una expresión `case`. Las expresiones case proporcionan una utilidad similar a las funciones anónimas: no siempre es deseable dar un nombre a una función, y una expresión `case` nos permite evitar nombrar una función sólo porque queremos usar un patrón.

Aquí tenemos un ejemplo. Esta función calcula el "sufijo cero más largo" de un array (el sufijo más largo que suma cero):

```haskell
import Data.Array.Partial (tail)
import Partial.Unsafe (unsafePartial)

lzs :: Array Int -> Array Int
lzs [] = []
lzs xs = case sum xs of
           0 -> xs
           _ -> lzs (unsafePartial tail xs)
```

Por ejemplo:

```text
> lzs [1, 2, 3, 4]
[]

> lzs [1, -1, -2, 3]
[-1, -2, 3]
```

Esta función trabaja por análisis de casos. Si el array está vacío, nuestra única opción es devolver un array vacío. Si el array no está vacío, usamos una expresión `case` para partir en dos casos. Si la suma del array es cero, devolvemos el array completo. Si no, recurrimos sobre la cola del array.

## Fallos de ajuste de patrones (*pattern match failures*) y funciones parciales (*partial functions*)

Si los patrones de una expresión case se prueban por orden, ¿qué pasa en el caso en que ninguno de los patrones de las alternativas se ajustan a sus entradas? En este caso, la expresión case fallará en tiempo de ejecución con un _fallo de ajuste de patrones_.

Podemos ver este comportamiento con un ejemplo simple:

```haskell
import Partial.Unsafe (unsafePartial)

partialFunction :: Boolean -> Boolean
partialFunction = unsafePartial \true -> true
```

Esta función contiene un único caso, que sólo se ajusta a una única entrada, `true`. Si compilamos este fichero y probamos en PSCi con cualquier otro argumento, veremos un error en tiempo de ejecución:

```text
> partialFunction false

Failed pattern match
```

Las funciones que devuelven un valor para cualquier combinación de entradas se llaman funciones _totales_, y las funciones que no se llaman _parciales_.

Generalmente se considera mejor definir funciones totales donde sea posible. Si se sabe que una función no devuelve un resultado para algún conjunto válido de entradas, normalmente es mejor devolver un valor de tipo `Maybe a` para algún `a`, usando `Nothing` para indicar fallo. De esta manera, la presencia o ausencia de un valor se puede indicar de una forma segura a nivel de tipos.

El compilador de PureScript generará un error si puede detectar que tu función no es total debido a un ajuste de patrones incompleto. La función `unsafePartial` se puede usar para silenciar estos errores (¡si estas seguro de que tu función parcial es segura!). Si quitamos la llamada a la función `unsafePartial` en la función de antes, el compilador generará el siguiente error:

```text
A case expression could not be determined to cover all inputs.
The following additional cases are required to cover all inputs:

  false
```

Esto nos dice que el valor `false` no coincide con ningún patrón. En general, estos avisos pueden incluir múltiples casos sin ajuste.

Si también omitimos la firma de tipos de antes:

```purescript
partialFunction true = true
```

PSCi infiere un tipo curioso:

```text
> :type partialFunction

Partial => Boolean -> Boolean
```

Veremos más tipos que involucran el símbolo `=>` más tarde en el libro (están relacionados con las _clases de tipos_), pero por ahora, basta observar que PureScript lleva el registro de las funciones parciales usando el sistema de tipos, y que debemos decir explícitamente al comprobador de tipos cuándo son seguras.

El compilador generará también un aviso en ciertos casos cuando puede detectar casos _redundantes_ (esto es, un caso sólo se ajusta a valores que habrían sido ajustados por un caso anterior):

```purescript
redundantCase :: Boolean -> Boolean
redundantCase true = true
redundantCase false = false
redundantCase false = false
```

En este caso, el último caso se identifica correctamente como redundante:

```text
Redundant cases have been detected.
The definition has the following redundant cases:

  false
```

_Nota_: PSCi no muestra avisos, de manera que para reproducir este ejemplo tendrás que salvar esta función a un fichero y compilarla usando `pulp build`.

## Tipos de datos algebraicos (*algebraic data types*)

Esta sección introducirá una característica del sistema de tipos de PureScript llamada _tipos de datos algebraicos_ (o _ADTs_), que se relacionan fundamentalmente con el ajuste de patrones.

Sin embargo, vamos primero a considerar un ejemplo motivador que proporcionará la base para una solución al problema de este capítulo de implementar una biblioteca de gráficos vectoriales simple.

Supongamos que queremos definir un tipo para representar algunos tipos simples de formas: líneas, rectángulos, círculos, texto, etc. En un lenguaje orientado a objetos, probablemente definiríamos una interfaz o clase abstracta `Shape`, y una subclase concreta para cada tipo de forma con la que queramos trabajar.

Sin embargo, esta aproximación tiene un inconveniente importante: para trabajar con `Shape`s de manera abstracta, es necesario identificar todas las operaciones que uno quiera realizar y definirlas en la interfaz `Shape`. Se vuelve difícil añadir nuevas operaciones sin romper la modularidad.

Los tipos de datos algebraicos proporcionan una forma segura a nivel de tipos de resolver este tipo de problemas si el conjunto de formas se conoce por anticipado. Es posible definir nuevas operaciones sobre `Shape` de una forma modular manteniendo la seguridad a nivel de tipos.

Así es como `Shape` se puede representar como un tipo de datos algebraico:

```haskell
data Shape
  = Circle Point Number
  | Rectangle Point Number Number
  | Line Point Point
  | Text Point String
```

El tipo `Point` se puede definir también como un tipo de datos algebraico como sigue:

```haskell
data Point = Point
  { x :: Number
  , y :: Number
  }
```

El tipo de datos `Point` ilustra algunos puntos interesantes:

- Los datos portados por un constructor de ADT no están restringidos a tipos primitivos: los constructores pueden incluir registros, arrays, o incluso otros ADTs.
- Aunque los ADTs son útiles para describir datos con múltiples constructores, también pueden ser útiles cuando hay un único constructor.
- Los constructores de un tipo de datos algebraico pueden tener el mismo nombre que el propio ADT. Esto es bastante común y es importante no confundir el _constructor de tipo_ `Point` con el _constructor de datos_ `Point`; viven en espacios de nombres distintos.

Esta declaración define `Shape` como una suma de diferentes constructores, y para cada constructor identifica los datos que se incluyen. Una `Shape` es o bien un `Circle` que contiene un `Point` para el centro y un radio (un número), o un `Rectangle`, o una `Line`, o `Text`. No hay otras formas de construir un valor de tipo `Shape`.

Los tipos de datos algebraicos se presentan usando la palabra reservada `data`, seguida del nombre del tipo nuevo y cualquier argumento de tipo. Los constructores de tipo se definen tras el signo igual y se separan con barras (`|`).

Veamos otro ejemplo de las bibliotecas estándar de PureScript. Vimos antes el tipo `Maybe` que se usa para definir valores opcionales. Aquí está su definición del paquete `purescript-maybe`:

```haskell
data Maybe a = Nothing | Just a
```

Este ejemplo muestra el uso del parámetro de tipo `a`. Leyendo la barra como la palabra "o", su definición es bastante legible: "un valor de tipo `Maybe a` es o bien `Nothing` o `Just` un valor de tipo `a`".

Los constructores de datos se pueden usar también para definir estructuras de datos recursivas. Aquí hay otro ejemplo definiendo un tipo de datos para listas enlazadas de elementos de tipo `a`:

```haskell
data List a = Nil | Cons a (List a)
```

Este ejemplo está sacado del paquete `purescript-lists`. Aquí, el constructor `Nil` representa una lista vacía, y `Cons` se usa para crear listas no vacías a partir de un elemento de cabeza y una cola. Date cuenta como la cola se define usando el tipo de datos `List a`, convirtiéndose en un tipo de datos recursivo.

## Usando ADTs

Es bastante simple usar los constructores de un tipo de datos algebraico para construir un valor: simplemente los aplicamos como funciones, proporcionando argumentos correspondientes a los datos incluidos en el constructor apropiado.

Por ejemplo, el constructor `Line` definido arriba requería dos `Point`s, así que para construir un `Shape` usando el constructor `Line` tenemos que proporcionar dos argumentos de tipo `Point`:

```haskell
exampleLine :: Shape
exampleLine = Line p1 p2
  where
    p1 :: Point
    p1 = Point { x: 0.0, y: 0.0 }

    p2 :: Point
    p2 = Point { x: 100.0, y: 50.0 }
```

Para construir los puntos `p1` y `p2`, aplicamos el constructor `Point` a su único argumento, que es un registro.

Así, construir valores de tipos de datos algebraicos es simple, pero ¿cómo los usamos? Aquí es donde aparece la conexión importante con el ajuste de patrones: la única forma de consumir valores de un tipo algebraico de datos es usar ajuste de patrones para ajustarse a su constructor.

Veamos un ejemplo. Supongamos que queremos convertir una `Shape` en `String`. Tenemos que usar ajuste de patrones para descubrir qué constructor se usó para construir la `Shape`. Lo podemos hacer como sigue:

```haskell
showPoint :: Point -> String
showPoint (Point { x: x, y: y }) =
  "(" <> show x <> ", " <> show y <> ")"

showShape :: Shape -> String
showShape (Circle c r)      = ...
showShape (Rectangle c w h) = ...
showShape (Line start end)  = ...
showShape (Text p text) = ...
```

Cada constructor se puede usar como un patrón, y los argumentos al constructor se pueden ligar usando sus propios patrones. Considera el primer caso de `showShape`: si la `Shape` coincide con el constructor `Circle`, metemos en contexto los argumentos de `Circle` (centro y radio) usando dos patrones variables `c` y `r`. Los otros casos son similares.

`showPoint` es otro ejemplo de ajuste de patrones. En este caso, sólo hay un único caso, pero usamos un patrón anidado para ajustarnos a los nombres del registro contenido dentro del constructor `Point`.

## Doble sentido en registros (*record puns*)

La función `showPoint` se ajusta a un registro dentro de su argumento, ligando las propiedades `x` e `y` a valores con los mismos nombres. En PureScript podemos simplificar este tipo de ajuste de patrones como sigue:

```haskell
showPoint :: Point -> String
showPoint (Point { x, y }) = ...
```

Aquí, únicamente especificamos los nombres de las propiedades y no necesitamos especificar los nombres de los valores que queremos ligar. Es lo que se llama un _doble sentido en registro_.

También es posible usar doble sentido en registro para _construir_ registros. Por ejemplo, si tenemos valores llamados `x` e `y` en el ámbito, podemos construir un `Point` usando Point{ x, y}`:

```haskell
origin :: Point
origin = Point { x, y }
  where
    x = 0.0
    y = 0.0
```

Esto puede ser útil para mejorar la legibilidad del código en algunas circunstancias.

X> ## Ejercicios
X>
X> 1. (Fácil) Construye un valor de tipo `Shape` que representa un círculo centrado en el origen con radio `10.0`.
X> 1. (Medio) Escribe una función de `Shape`s a `Shape`s que escala sus argumentos por un factor de `2.0` con centro en el origen.
X> 1. (Medio) Escribe una función que extraiga el texto de un `Shape`. Debe devolver `Maybe String` y usar el constructor `Nothing` si la entrada no está construida usando `Text`.

## Newtypes

Hay un caso especial importante de tipos de datos algebraicos llamados _newtypes_. Los newtypes se presentan usando la palabra reservada `newtype` en lugar de `data`.

Los newtypes deben definir _exactamente un_ constructor, y ese constructor debe tomar _exactamente un_ argumento. Esto es, un newtype da un nuevo nombre a un tipo existente. De hecho, los valores de un newtype tienen la misma representación en tiempo de ejecución que el tipo subyacente. Son, sin embargo, distintos desde el punto de vista del sistema de tipos. Esto da un nivel adicional de seguridad de tipos.

Como ejemplo, podemos querer definir newtypes como sinónimos de `Number` para atribuir unidades como píxeles y pulgadas:

```haskell
newtype Pixels = Pixels Number
newtype Inches = Inches Number
```

De esta forma, es imposible pasar un valor de tipo `Pixels` a una función que espera `Inches`, pero no hay sobrecoste de rendimiento en tiempo de ejecución. 

Los newtypes cobrarán importancia cuando veamos las _clases de tipos_ en el siguiente capítulo, ya que nos permiten adjuntar comportamiento diferente a un tipo sin cambiar su representación en tiempo de ejecución.

## Una biblioteca para gráficos vectoriales

Usemos los tipos de datos que hemos definido antes para crear una biblioteca simple para usar gráficos vectoriales.

Definamos un sinónimo de tipo para una `Picture`, un simple array de `Shape`s:

```haskell
type Picture = Array Shape
```

Para depurar, querremos ser capaces de convertir una `Picture` en algo legible. La función `showPicture` nos permite hacerlo:

```haskell
showPicture :: Picture -> Array String
showPicture = map showShape
```

Probémosla. Compila tu módulo con `pulp build` y abre PSCi con `pulp repl`:

```text
$ pulp build
$ pulp repl

> import Data.Picture

> :paste
… showPicture
…   [ Line (Point { x: 0.0, y: 0.0 })
…          (Point { x: 1.0, y: 1.0 })
…   ]
… ^D

["Line [start: (0.0, 0.0), end: (1.0, 1.0)]"]
```

## Calculando rectángulos de delimitación

El código de ejemplo para este módulo contiene una función `bounds` que calcula el rectángulo de delimitación mínimo para una `Picture`.

El tipo de datos `Bounds` define un rectángulo de delimitación. También está definido como un tipo algebraico de datos con un único constructor:

```haskell
data Bounds = Bounds
  { top    :: Number
  , left   :: Number
  , bottom :: Number
  , right  :: Number
  }
```

`bounds` usa la función `foldl` de `Data.Foldable` para recorrer el array de `Shape`s en una `Picture`, y acumular el rectángulo de delimitación mínimo:

```haskell
bounds :: Picture -> Bounds
bounds = foldl combine emptyBounds
  where
    combine :: Bounds -> Shape -> Bounds
    combine b shape = union (shapeBounds shape) b
```

En el caso base, necesitamos encontrar el rectángulo de delimitación mínimo de una `Picture` vacía, y el rectángulo de delimitación mínimo vacío definido por `emptyBounds` es suficiente.

La función de acumulación `combine` se define en un bloque `where`. `combine` toma un rectángulo de delimitación calculado por la llamada recursiva `foldl` y la siguiente `Shape` del array, y usa la función `union` para calcular la unión de dos rectángulos de delimitación. La función `shapeBounds` calcula la delimitación de un único shape usando ajuste de patrones. 

X> ## Ejercicios
X>
X> 1. (Medio) Extiende la biblioteca de gráficos vectoriales con una nueva operación `area` que calcula el área de una `Shape`. Para los propósitos de este ejercicio, el área de un texto se asume que es cero.
X> 1. (Difícil) Extiende el tipo `Shape` con un constructor de datos nuevo `Clipped`, que recorta otra `Picture` con un rectángulo. Extiende la función `shapeBounds` para calcular los límites de una imagen recortada. Date cuenta de que esto convierte `Shape` en un tipo de datos recursivo.

## Conclusión

En este capítulo, hemos visto el ajuste de patrones, una técnica básica pero potente de la programación funcional. Hemos visto cómo usar patrones simples así como patrones de array y de registro para ajustar partes de estructuras de datos profundas.

Este capítulo también ha presentado los tipos de datos algebraicos, que están estrechamente relacionados con el ajuste de patrones. Hemos visto cómo los tipos de datos algebraicos permiten descripciones concisas de estructuras de datos y proporcionan una forma modular de extender tipos de datos con nuevas operaciones.

Finalmente, hemos visto el _polimorfismo de línea_, un potente tipo de abstracción que permite dar un tipo a muchas funciones idiomáticas de JavaScript. Veremos esta idea de nuevo más adelante.

En el resto del libro, usaremos ADTs y ajuste de patrones extensamente, de manera que va a resultar rentable familiarizarse con ellos ya. Intenta crear tus propios tipos de datos algebraicos y escribir funciones que los consuman usando ajuste de patrones.
