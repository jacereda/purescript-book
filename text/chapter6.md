# Clases de tipos (*type classes*)

## Objetivos del capítulo

Este capítulo presentará una poderosa forma de abstracción disponible en el sistema de tipos de PureScript: las clases de tipos.

El ejemplo motivador de este capítulo será una biblioteca para resumir (hashing) estructuras de datos. Veremos cómo la maquinaria de las clases de tipos nos permiten resumir estructuras de datos complejas sin tener que pensar directamente en la estructura de los propios datos.

Veremos también una colección de clases de tipos estándar del Prelude de PureScript y de las bibliotecas estándar. El código PureScript se apoya firmemente en la potencia de las clases de tipos para expresar ideas de manera concisa, así que será beneficioso que te familiarices con estas clases.

## Preparación del proyecto

El código fuente de este capítulo está definido en el fichero `src/Data/Hashable.purs`. 

El proyecto tiene las siguientes dependencias Bower:

- `purescript-maybe`, definiendo el tipo de datos `Maybe`, que representa valores opcionales.
- `purescript-tuples`, definiendo el tipo de datos `Tuple`, que representa pares de valores.
- `purescript-either`, definiendo el tipo de datos `Either`, que representa uniones disjuntas.
- `purescript-strings`, que define funciones que operan sobre cadenas.
- `purescript-functions`, que define algunas funciones auxiliares para definir funciones PureScript.

El módulo `Data.Hashable` importa varios módulos proporcionados por estos paquetes Bower.
## ¡Muéstrame!

Nuestro primer ejemplo simple de clase de tipos viene dado por una función que hemos visto varias veces: la función `show`, que toma un valor y lo representa como una cadena.

`show` está definido por una clase de tipos del módulo `Prelude` llamada `Show`, definida como sigue:

```haskell
class Show a where
  show :: a -> String
```

Este código declara una nueva _clase de tipos_ llamada `Show`, que está parametrizada por la variable de tipo `a`.

Una _instancia_ de clase de tipos contiene implementaciones de las funciones definidas en una clase de tipos, especializada para un tipo particular.

Por ejemplo, aquí está la definición de la instancia de clase de tipos `Show` para valores `Boolean`, tomada del Prelude:

```haskell
instance showBoolean :: Show Boolean where
  show true = "true"
  show false = "false"
```

Este código declara una instancia de clase de tipos llamada `showBoolean`; en PureScript, las instancias de clases de tipos tienen nombre para ayudar con la legibilidad del JavaScript generado. Decimos que el tipo `Boolean` _pertenece a la clase de tipos `Show`_.

Podemos probar la clase de tipos `Show` en PSCi mostrando unos cuantos valores de diferentes tipos:

```text
> import Prelude

> show true
"true"

> show 1.0
"1.0"

> show "Hello World"
"\"Hello World\""
```

Estos ejemplos muestran cómo usar `show` para mostrar valores de varios tipos primitivos, pero también podemos usar `show` para mostrar valores de tipos más complejos:

```text
> import Data.Tuple

> show (Tuple 1 true)
"(Tuple 1 true)"

> import Data.Maybe

> show (Just "testing")
"(Just \"testing\")"
```

Si intentamos mostrar un valor del tipo `Data.Either` obtenemos un mensaje de error interesante:

```text
> import Data.Either
> show (Left 10)

The inferred type

    forall a. Show a => String

has type variables which are not mentioned in the body of the type. Consider adding a type annotation.
```

El problema aquí no es que no haya una instancia `Show` para el tipo que tratamos de mostrar, sino que PSCi ha sido incapaz de inferir el tipo. Esto viene indicado por el _tipo desconocido_ `a` en el tipo inferido.

Podemos anotar la expresión con un tipo usando el operador `::`, de manera que PSCi pueda elegir la instancia de clase de tipos correcta:

```text
> show (Left 10 :: Either Int String)
"(Left 10)"
```

Algunos tipos ni siquiera tienen una instancia `Show` definida. Un ejemplo es el tipo función `->`. Si tratamos de mostrar una función de `Int` a `Int`, obtenemos un mensaje de error apropiado del comprobador de tipos:

```text
> import Prelude
> show $ \n -> n + 1

No type class instance was found for

  Data.Show.Show (Int -> Int)
```

X> ## Ejercicios
X>
X> 1. (Fácil) Usa la función `showShape` del capítulo anterior para definir una instancia `Show` para el tipo `Shape`.

## Clases de tipos comunes

En esta sección, vamos a ver algunas clases de tipos estándar definidas en el Prelude y en las bibliotecas estándar. Estas clases de tipos forman la base de muchos patrones comunes de abstracción en el código PureScript idiomático, de manera que una comprensión básica de sus funciones es altamente recomendable.

### Eq

La clase de tipos `Eq` define la función `eq` que comprueba la igualdad entre dos valores. El operador `==` es un sinónimo de `eq`.

```haskell
class Eq a where
  eq :: a -> a -> Boolean
```

Date cuenta de que en cualquier caso, los dos argumentos deben tener el mismo tipo. No tiene sentido comprobar la igualdad entre dos valores de distinto tipo.

Prueba la clase de tipos `Eq` en PSCi:

```text
> 1 == 2
false

> "Test" == "Test"
true
```

### Ord

La clase de tipos `Ord` define la función `compare`, que se puede usar para comparar dos valores para tipos que soporten ordenación. Los operadores de comparación `<` y `>` junto a sus compañeros no estrictos `<=` y `>=` se pueden definir en términos de `compare`.

```haskell
data Ordering = LT | EQ | GT

class Eq a <= Ord a where
  compare :: a -> a -> Ordering
```

La función `compare` compara dos valores y devuelve un `Ordering` con tres alternativas:

- `LT` - si el primer argumento es menor que el segundo.
- `EQ` - si el primer argumento es igual al segundo.
- `GT` - si el primer argumento es mayor que el segundo.

De nuevo, podemos probar la función `compare` en PSCi: 

```text
> compare 1 2
LT

> compare "A" "Z"
LT
```

### Field

La clase de tipos `Field` identifica los tipos que soportan operadores numéricos como suma, resta, multiplicación y división. Se proporciona para abstraer sobre esos operadores de manera que puedan ser reutilizados donde sea apropiado.

_Nota_: Al igual que las clases de tipos `Eq` y `Ord`, la clase de tipos `Field` tiene soporte especial en el compilador de PureScript, de manera que expresiones simples como `1 + 2 * 3` se traduzcan a JavaScript simple, en contraposición a llamadas de función que se despachan en base a una implementación de clase de tipos.

```haskell
class EuclideanRing a <= Field a
```

La clase de tipos `Field` está compuesta de varias _superclases_ generales más. Esto nos permite hablar de manera abstracta de tipos que soportan algunas, no todas, de las operaciones de `Field`. Por ejemplo, un tipo de números naturales sería cerrado bajo la suma y la multiplicación, pero no necesariamente bajo la resta, de manera que ese tipo puede tener una instancia de la clase `Semiring` (que es una superclase de `Num`), pero no una instancia de `Ring` o `Field`. 

Las superclases se explicarán más tarde en este capítulo, pero la jerarquía completa de tipos numéricos está más allá del ámbito de este capítulo. Animamos al lector interesado a leer la documentación de las superclases de `Field` en `purescript-prelude`. 

### Semigrupos (*semigroups*) y Monoides (*monoids*)

La clase de tipos `Semigroup` identifica aquellos tipos que soportan una operación `append` para combinar dos valores:

```haskell
class Semigroup a where
  append :: a -> a -> a
```

Las cadenas forman un semigrupo bajo la concatenación de cadenas normal, y lo mismo es cierto para los arrays. Muchas otras instancias estándar se proporcionan en el paquete `purescript-monoid`.

El operador de concatenación `<>`, que ya hemos visto, se proporciona como un sinónimo de `append`.

La clase de tipos `Monoid` (proporcionada por el paquete `purescript-monoid`) extiende el tipo `Semigroup` con el concepto de un valor vacío llamado `mempty`:

```haskell
class Semigroup m <= Monoid m where
  mempty :: m
```

De nuevo, las cadenas y los arrays son ejemplos simples de monoides.

Una instancia de la clase de tipos `Monoid` para un tipo describe cómo _acumular_ un resultado con ese tipo, comenzando por un valor "vacío" y combinando nuevos resultados. Por ejemplo, podemos escribir una función que concatena un grupo de valores en algún monoide usando un pliegue. En PSCi:

```haskell
> import Data.Monoid
> import Data.Foldable

> foldl append mempty ["Hello", " ", "World"]  
"Hello World"

> foldl append mempty [[1, 2, 3], [4, 5], [6]]
[1,2,3,4,5,6]
```

El paquete `purescript-monoid` proporciona muchos ejemplos de monoides y semigrupos que usaremos en el resto del libro.

### Foldable

Si la clase de tipos `Monoid` identifica los tipos que pueden actuar como resultado de un pliegue, la clase de tipos `Foldable` identifica los constructores de tipos que pueden ser usados como la fuente de un pliegue.

La clase de tipos `Foldable` se suministra en el paquete `purescript-foldable-traversable`, que también contiene instancias para algunos contenedores estándar como `Array` y `Maybe`.

Las firmas de tipo de las funciones pertenecientes a `Foldable` son un poco más complicadas que las que hemos visto hasta ahora:

```haskell
class Foldable f where
  foldr :: forall a b. (a -> b -> b) -> b -> f a -> b
  foldl :: forall a b. (b -> a -> b) -> b -> f a -> b
  foldMap :: forall a m. Monoid m => (a -> m) -> f a -> m
```

Es instructivo especializar para el caso en que `f` es el constructor de tipo array. En este caso, podemos reemplazar `f a` con `Array a` para cualquier `a`, y nos damos cuenta de que los tipos de `foldl` y `foldr` se convierten en los tipos que vimos cuando encontramos por primera vez los pliegues sobre arrays.

¿Qué pasa con `foldMap`? Bien, esa se convierte en `forall a m. Monoid m => (a -> m) -> Array a -> m`. Esta firma de tipo dice que podemos elegir cualquier tipo `m` para nuestro tipo resultado, siempre y cuando ese tipo sea una instancia de la clase de tipos `Monoid`. Si podemos proporcionar una función que convierte nuestros elementos de array en valores en ese monoide, entonces podemos acumular sobre nuestro array usando la estructura del monoide y devolver un único valor.

Probemos `foldMap` en PSCi:

```text
> import Data.Foldable

> foldMap show [1, 2, 3, 4, 5]
"12345"
```

Aquí, elegimos el monoide para cadenas, que concatena cadenas, y la función `show` que representa un `Int` como un `String`. Entonces, pasando un array de enteros, vemos que los resultados de mostrar cada entero han sido concatenados en una única `String`. 

Pero los arrays no son los únicos tipos plegables. `purescript-foldable-traversable` también define instancias de `Foldable` para tipos como `Maybe` y `Tuple`, y otras bibliotecas como `purescript-lists` definen instancias de `Foldable` para sus propios tipos de datos. `Foldable` captura la noción de _contenedor ordenado_. 

### Funtor (*functor*) y leyes de clases de tipos (*type class laws*)

El Prelude también define una colección de clases de tipos que permiten un estilo de programación funcional con efectos secundarios en PureScript: `Functor`, `Applicative` y `Monad`. Veremos estas abstracciones más adelante, pero por ahora veamos la definición de la clase de tipos `Functor`, que ya hemos visto en forma de la función `map`:

```haskell
class Functor f where
  map :: forall a b. (a -> b) -> f a -> f b
```

La función `map` (y su sinónimo `<$>`) permite "elevar" una función a una estructura de datos. La definición precisa de la palabra "elevar" depende de la estructura de datos en cuestión, pero ya hemos visto su comportamiento para algunos tipos simples:

```text
> import Prelude

> map (\n -> n < 3) [1, 2, 3, 4, 5]
[true, true, false, false, false]

> import Data.Maybe
> import Data.String (length)

> map length (Just "testing")
(Just 7)
```

¿Cómo podemos entender el significado de la función `map` cuando actúa sobre muchas estructuras diferentes de una manera diferente cada vez?

Podemos intuir que la función `map` aplica la función que se le da a cada elemento del contenedor y construye un nuevo contenedor a partir de los resultados, con la misma forma que el original. ¿Pero cómo podemos precisar este concepto?

Se espera que las instancias de la clase de tipos `Functor` obedezcan un conjunto de _leyes_ llamadas las _leyes del funtor_ (*functor laws*):

- `map id xs = xs`
- `map g (map f xs) = map (g <<< f) xs`

La primera ley es la _ley de identidad_. Dice que elevar la función identidad (la función que devuelve su argumento sin cambios) sobre una estructura devuelve la estructura original. Esto tiene sentido, ya que la función identidad no modifica su entrada.

La segunda ley es la _ley de composición_. Dice que mapear una función sobre una estructura y mapear una segunda función es lo mismo que mapear la composición de las dos funciones sobre la estructura.

Signifique lo que signifique "elevar" en el sentido general, debe ser cierto que cualquier definición razonable de elevar una función sobre una estructura de datos debe obedecer estas reglas.

Muchas clases de tipos estándar vienen con su propio conjunto de leyes similares. Las leyes dadas a una clase de tipos dan estructura a las funciones de esa clase de tipos y nos permiten estudiar sus instancias en general. El lector interesado puede investigar las leyes atribuidas a las clases de tipos estándar que ya hemos visto.

X> ## Ejercicios
X>
X> 1. (Fácil) El newtype siguiente representa un número complejo:
X>
X>     ```haskell
X>     newtype Complex = Complex
X>       { real :: Number
X>       , imaginary :: Number
X>       }
X>     ```
X>       
X>     Define instancias `Show` y `Eq` para `Complex`.


## Anotaciones de tipo (*type annotations*)

Los tipos de las funciones pueden ser restringidos usando clases de tipos. Aquí tenemos un ejemplo: supongamos que queremos escribir una función que comprueba si tres valores son iguales, usando la igualdad definida por una instancia de clase de tipos `Eq`.

```haskell
threeAreEqual :: forall a. Eq a => a -> a -> a -> Boolean
threeAreEqual a1 a2 a3 = a1 == a2 && a2 == a3
```

La declaración de tipo parece un tipo polimórfico ordinario definido usando `forall`. Sin embargo, a continuación hay una restricción de clase de tipos `Eq a` separada del resto del tipo por una flecha doble `=>`.

Este tipo dice que podemos llamar `threeAreEqual` con cualquier elección de tipo `a`, siempre y cuando haya una instancia `Eq` disponible para `a` en uno de los módulos importados.

Los tipos restringidos pueden contener varias instancias de clases de tipos, y los tipos de las instancias no están limitados a simples variables de tipo. Aquí hay otro ejemplo que usa instancias `Ord` y `Show` para comparar dos valores:  

```haskell
showCompare :: forall a. Ord a => Show a => a -> a -> String
showCompare a1 a2 | a1 < a2 =
  show a1 <> " is less than " <> show a2
showCompare a1 a2 | a1 > a2 =
  show a1 <> " is greater than " <> show a2
showCompare a1 a2 =
  show a1 <> " is equal to " <> show a2
```

Date cuenta de que se pueden especificar restricciones múltiples usando el símbolo `=>` múltiples veces, de la misma manera que especificamos funciones currificadas de varios argumentos. Pero recuerda no confundir ambos símbolos:

- `a -> b` denota el tipo de funciones del _tipo_ `a` al _tipo_ `b`, mientras que
- `a => b` aplica la _restricción_ `a` al tipo `b`.

El compilador PureScript intentará inferir los tipos restringidos cuando no se proporcione una anotación de tipo. Esto puede ser util si queremos usar el tipo más general posible para una función.

Para verlo, intenta usar una de las clases de tipos estándar como `Semiring` en PSCi:

```text
> import Prelude

> :type \x -> x + x
forall a. Semiring a => a -> a
```

Aquí, podríamos haber anotado esta función como `Int -> Int`, o `Number -> Number`, pero PSCi nos muestra que el tipo más general funciona para cualquier `Semiring`, permitiéndonos usar nuestra función tanto con `Int`s como con `Number`s.

## Instancias superpuestas (*overlapping instances*)

PureScript tiene otra regla relativa a las instancias de clases de tipos, llamada la _regla de instancias superpuestas_ (*overlapping instances rule*). Cuando una instancia de clase de tipos se necesita en un punto de llamada a función, PureScript usará la información inferida por el comprobador de tipos para elegir la instancia correcta. En ese momento, debe haber exactamente una instancia apropiada para ese tipo. Si hay varias instancias válidas, el compilador emitirá un aviso. 

Para mostrar esto, podemos intentar crear dos instancias de clase de tipos en conflicto para un tipo de ejemplo. En el siguiente código, creamos dos instancias superpuestas de `Show` para le tipo `T`:

```haskell
module Overlapped where

import Prelude

data T = T

instance showT1 :: Show T where
  show _ = "Instance 1"

instance showT2 :: Show T where
  show _ = "Instance 2"
```

Este módulo compilará sin avisos. Sin embargo, si _usamos_ `show` sobre el tipo `T` (requiriendo al compilador que encuentre una instancia `Show`), la regla de instancias superpuestas se aplicará dando un aviso:

```text
Overlapping instances found for Prelude.Show T
```

La regla de instancias superpuestas debe cumplirse para que la selección de instancias de clases de tipos sea un proceso predecible. Si permitiésemos que existiesen dos instancias de clase de tipos para un tipo, cualquiera podría elegirse dependiendo del orden de importación de los módulos, y podría llevar a comportamiento impredecible del programa en tiempo de ejecución, algo no deseable.

Si realmente es cierto que hay dos instancias de clase de tipos válidas para un tipo que satisfacen las leyes apropiadas, una aproximación común es definir newtypes para envolver el tipo existente. Como se permite que los newtypes diferentes tengan diferentes instancias de clases de tipos, para la regla de instancias superpuestas ya no es un problema. Esta aproximación se usa en las bibliotecas estándar de PureScript, por ejemplo en `purescript-maybe`, donde el tipo `Maybe a` tiene múltiples instancias válidas para la clase de tipos `Monoid`.

## Dependencias de instancia (*instance dependencies*)

Al igual que la implementación de funciones puede depender de las instancias de clases de tipos usando tipos restringidos, también puede la implementación de instancias de clases de tipos depender de otras instancias de clases de tipos. Esto proporciona una forma poderosa de inferencia de programa, en la que la implementación de un programa se puede inferir usando sus tipos.

Por ejemplo, considera la clase de tipos `Show`. Podemos escribir una instancia de clase de tipos para mostrar arrays de elementos, siempre y cuando tengamos una manera de mostrar los propios elementos.

```haskell
instance showArray :: Show a => Show (Array a) where
  ...
```

Si una instancia de una clase de tipos depende de varias instancias, esas instancias deben agruparse en paréntesis y separarse por comas a la izquierda del símbolo `=>`:

```haskell
instance showEither :: (Show a, Show b) => Show (Either a b) where
  ...
```

Estas dos instancia de clase de tipos se proporciona el la biblioteca `purescript-prelude`.

Cuando se compila el programa, se elige la instancia correcta de clase de tipos para `Show` basándose en el tipo inferido del argumento pasado a `show`. La instancia seleccionada puede depender de muchas relaciones de instancia, pero esta complejidad no se expone al desarrollador.

X> ## Ejercicios
X>
X> 1. (Fácil) La siguiente declaración define un tipo de arrays no vacios con elementos de tipo `a`:
X>
X>     ```haskell
X>     data NonEmpty a = NonEmpty a (Array a)
X>     ```
X>      
X>     Escribe una instancia `Eq` para el tipo `NonEmpty a` que reutiliza las instancias de `Eq a` y `Eq (Array a)`.
X> 1. (Medio) Escribe una instancia de `Semigroup` para `NonEmpty a` reutilizando la instancia de `Semigroup` para `Array`.
X> 1. (Medio) Escribe una instancia de `Functor` para `NonEmpty`.
X> 1. (Medio) Dado un tipo `a` cualquiera con una instancia de `Ord`, podemos añadir un nuevo valor "infinito" que es mayor que cualquier otro valor: 
X>
X>     ```haskell
X>     data Extended a = Finite a | Infinite
X>     ```
X>         
X>     Escribe una instancia `Ord` para `Extended a` que reutiliza la instancia `Ord` para `a`.
X> 1. (Difícil) Escribe una instancia de `Foldable` para `NonEmpty`. _Pista_: reutiliza la instancia `Foldable` para arrays.
X> 1. (Difícil) Dado un constructor de tipo `f` que define un contenedor ordenado (y por lo tanto tiene una instancia de `Foldable`), podemos crear un nuevo tipo de contenedor que incluye un elemento extra al frente:
X>
X>     ```haskell
X>     data OneMore f a = OneMore a (f a)
X>     ```
X>         
X>     El contenedor `OneMore f` también tiene un orden, donde el nuevo elemento va antes que cualquier elemento de `f`. Escribe una instancia de `Foldable` para `OneMore f`:
X>   
X>     ```haskell
X>     instance foldableOneMore :: Foldable f => Foldable (OneMore f) where
X>       ...
X>     ```

## Clases de tipos de varios parámetros (*multi parameter type classes*)

No es cierto que una clase de tipos pueda tomar un único tipo como argumento. Es el caso más común, pero de hecho, una clase de tipos se puede parametrizar por _cero o más_ argumentos de tipo.

Veamos un ejemplo de una clase de tipos con dos argumentos de tipo.

```haskell
module Stream where

import Data.Array as Array
import Data.Maybe (Maybe)
import Data.String as String

class Stream stream element where
  uncons :: stream -> Maybe { head :: element, tail :: stream }

instance streamArray :: Stream (Array a) a where
  uncons = Array.uncons

instance streamString :: Stream String Char where
  uncons = String.uncons
```

El módulo `Stream` define una clase `Stream` que identifica tipos que se pueden ver como flujos de elementos, donde los elementos pueden ser extraídos del frente del flujo usando la función `uncons`.

Date cuenta de que la clase de tipos `Stream` está parametrizada no sólo por el tipo del flujo, sino también por sus elementos. Esto permite definir instancias de clase de tipos para el mismo tipo de flujo pero con diferentes tipos de elementos.

El módulo define dos instancias de clase de tipos: una instancia para arrays, donde `uncons` quita el elemento de cabeza del array usando ajuste de patrones, y una instancia para `String` que quita el primer carácter de una `String`.

Podemos escribir funciones que trabajan sobre flujos arbitrarios. Por ejemplo, aquí hay una función que acumula un resultado en algún `Monoid` basándose en los elementos de un flujo:

```haskell
import Prelude
import Data.Monoid (class Monoid, mempty)

foldStream :: forall l e m. Stream l e => Monoid m => (e -> m) -> l -> m
foldStream f list =
  case uncons list of
    Nothing -> mempty
    Just cons -> f cons.head <> foldStream f cons.tail
```

Intenta usar `foldStream` en PSCi para diferentes tipos de `Stream` y diferentes tipos de `Monoid`.

## Dependencias funcionales (*functional dependencies*)

Las clases de tipos multiparamétricas pueden ser muy útiles, pero pueden llevar fácilmente a tipos confusos e incluso problemas con la inferencia de tipos. Como ejemplo simple, supongamos que tenemos que escribir una función `tail` genérica sobre flujos usando la clase `Stream` dada arriba:

```haskell
genericTail xs = map _.tail (uncons xs)
```

Esto nos da un mensaje de error algo confuso:

```text
The inferred type

  forall stream a. Stream stream a => stream -> Maybe stream

has type variables which are not mentioned in the body of the type. Consider adding a type annotation.
```

El problema es que la función `genericTail` no usa el tipo `element` mencionado en la definición de la clase de tipos `Stream`, de manera que el tipo queda sin resolver.

Peor aún, ni siquiera podemos usar `genericTail` aplicándolo a un tipo específico de flujo:

```text
> map _.tail (uncons "testing")

The inferred type

  forall a. Stream String a => Maybe String

has type variables which are not mentioned in the body of the type. Consider adding a type annotation.
```

Aquí, podemos esperar que el compilador elija la instancia `streamString`. Después de todo, una `String` es un flujo de `Char`s, y no puede ser un flujo de ningún otro tipo de elementos.

El compilador no es capaz de hacer esa deducción automáticamente, y no puede confiar en la instancia `streamString`. Sin embargo, podemos ayudar al compilador añadiendo una pista en la definición de la clase de tipo:

```haskell
class Stream stream element | stream -> element where
  uncons :: stream -> Maybe { head :: element, tail :: stream }
```

Aquí, `stream -> element` se llama _dependencia funcional_. Una dependencia funcional asegura una relación funcional entre los argumentos de tipo de una clase de tipo multiparamétrica. Esa dependencia funcional dice al compilador que hay una función de tipos de flujo a tipos de elemento (únicos), de manera que si el compilador sabe el tipo del flujo, puede conocer el tipo del elemento.

Esta pista es suficiente para que el compilador infiera el tipo correcto para nuestra función tail genérica de arriba:

```text
> :type genericTail
forall stream element. Stream stream element => stream -> Maybe stream

> genericTail "testing"
(Just "esting")
```

Las dependencias funcionales pueden ser bastante útiles cuando se usan clases de tipo multiparamétricas para diseñar ciertas APIs.

## Clases de tipos nularias (*nullary type classes*)

¡Podemos incluso definir clases de tipos sin argumentos de tipo! Se corresponden a aseveraciones (*asserts*) en tiempo de compilación sobre nuestras funciones, permitiéndonos seguir la pista a propiedades globales de nuestro código en el sistema de tipos.

Un ejemplo importante es la clase `Partial` que vimos anteriormente cuando hablábamos de las funciones parciales. Hemos visto ya las funciones parciales `head` y `tail` definidas en `Data.Array.Partial`:

```haskell
head :: forall a. Partial => Array a -> a

tail :: forall a. Partial => Array a -> Array a
```

¡Date cuenta de que no hay instancias definidas para la clase de tipos `Partial`! Hacerlo anularía su propósito: intentar usar la función `head` directamente resultará en un error de tipo:

```text
> head [1, 2, 3]

No type class instance was found for

  Prim.Partial
```

En su lugar, podemos volver a publicar la restricción `Partial` para cualquier función que haga uso de funciones parciales:

```haskell
secondElement :: forall a. Partial => Array a -> a
secondElement xs = head (tail xs)
```

Ya hemos visto la función `unsafePartial` que nos permite tratar una función parcial como una función normal (de manera insegura). Esta función está definida en el módulo `Partial.Unsafe`:

```haskell
unsafePartial :: forall a. (Partial => a) -> a
```

Date cuenta de que la restricción `Partial` aparece _entre paréntesis_ a la izquierda de la flecha de función, pero no en el `forall` de fuera. Esto es, `unsafePartial` es una función de valores parciales a valores normales.

## Superclases (*superclasses*)

Al igual que podemos expresar relaciones entre instancias de clases de tipos haciendo que una instancia dependa de otra, podemos expresar relaciones entre las propias clases de tipos usando las llamadas _superclases_.

Decimos que una clase de tipos es una superclase de otra si se requiere que toda instancia de la segunda clase sea una instancia de la primera, e indicamos una relación de superclase en la definición de clase usando una doble flecha hacia atrás. 

Hemos visto ya algunos ejemplos de relaciones de superclase: La clase `Eq` es una superclase de `Ord`, y la clase `Semigroup` es una superclase de `Monoid`. Para todas las instancias de clase de tipos de la clase `Ord` tiene que haber una instancia correspondiente de `Eq` para el mismo tipo. Esto tiene sentido, ya que en muchos casos, cuando la función `compare` informa de que dos valores son incomparables, a menudo queremos usar la clase `Eq` para determinar si de hecho son iguales. 

En general, tiene sentido definir una relación de superclase cuando las leyes de la subclase mencionan los miembros de la superclase. Por ejemplo, es razonable asumir, para cualquier par de instancias `Ord` y `Eq`, que si dos valores son iguales bajo la instancia `Eq`, entonces la función `compare` debe devolver `EQ`. En otras palabras, `a == b` debe ser ciero exactamente cuando `compare a b` se evalúa a `EQ`. Esta relación a nivel de leyes justifica la relación de superclase entre `Eq` y `Ord`.

Otra razón para definir una relación de superclase es en el caso donde hay una clara relación "es-un" entre las dos clases. Esto es, todos los miembros de la subclase _son_ miembros de la superclase también.

X> ## Ejercicios
X>
X> 1. (Medio) Define una función parcial que encuentra el máximo de un array no vacío de enteros. Tu función debe tener el tipo `Partial => Array Int -> Int`. Prueba tu función en PSCi usando `unsafePartial`. _Pista_: Usa la función `maximum` de `Data.Foldable`.
X> 1. (Medio) La clase `Action` es una clase de tipos de varios parámetros que define una acción de un tipo sobre otro:
X>
X>     ```haskell
X>     class Monoid m <= Action m a where
X>       act :: m -> a -> a
X>     ```
X>           
X>     Una _acción_ es una función que describe cómo se puede usar un monoide para modificar un valor de otro tipo. Hay dos leyes para la clase de tipos `Action`:
X>
X>     - `act mempty a = a`
X>     - `act (m1 <> m2) a = act m1 (act m2 a)`
X>
X>     Esto es, la acción respeta las operaciones definidas en la clase `Monoid`.
X>        
X>     Por ejemplo, los números naturales forman un monoide bajo la multiplicación:
X>
X>     ```haskell
X>     newtype Multiply = Multiply Int
X>
X>     instance semigroupMultiply :: Semigroup Multiply where
X>       append (Multiply n) (Multiply m) = Multiply (n * m)
X>
X>     instance monoidMultiply :: Monoid Multiply where
X>       mempty = Multiply 1
X>     ```
X>
X>     Este monoide actúa sobre cadenas repitiendo la cadena de entrada un número determinado de veces. Escribe una instancia que implementa esta acción:
X>
X>     ```haskell
X>     instance repeatAction :: Action Multiply String
X>     ```
X>
X>     ¿Satisface esta instancia las leyes listadas arriba?
X> 1. (Medio) Escribe una instancia `Action m a => Action m (Array a)`, donde la acción sobre arrays está definida por la acción sobre los elementos de manera independiente.
X> 1. (Difícil) Dado el siguiente newtype, escribe una instancia para `Action m (Self m)`, donde el monoide `m` actúa sobre sí mismo usando `append`:
X>
X>     ```haskell
X>     newtype Self m = Self m
X>     ```
X>
X> 1. (Difícil) ¿Deben los argumentos de la clase de tipos multiparamétrica `Action` estar relacionados por alguna dependencia funcional? ¿Por qué?

## Una clase de tipos para funciones resumen

En la última sección de este capítulo vamos a usar las lecciones del resto del capítulo para crear una biblioteca para resumir estructuras de datos.

Date cuenta de que esta biblioteca es para propósitos de demostración solamente y no pretende proporcionar un mecanismo de resumen robusto.

¿Qué propiedades podemos esperar de una función resumen?

- Una función resumen debe ser determinista y mapear valores iguales a códigos resumen iguales.
- Una función resumen debe distribuir sus resultados de manera aproximadamente uniforme sobre el conjunto de códigos resumen.

La primera propiedad se parece bastante a una ley para una clase de tipos, mientras que la segunda propiedad es más bien un contrato informal y ciertamente no sería realizable con el sistema de tipos de PureScript. Sin embargo, esto debe proporcionar la intuición para la siguiente clase de tipos:

```haskell
newtype HashCode = HashCode Int

hashCode :: Int -> HashCode
hashCode h = HashCode (h `mod` 65535)

class Eq a <= Hashable a where
  hash :: a -> HashCode
```

Con la ley asociada de que `a == b` implica `hash a == hash b`.

Pasaremos el resto de esta sección construyendo una biblioteca de instancias y funciones asociadas a la clase de tipos `Hashable`.

Necesitaremos una forma de combinar códigos hash de una forma determinista:

```haskell
combineHashes :: HashCode -> HashCode -> HashCode
combineHashes (HashCode h1) (HashCode h2) = hashCode (73 * h1 + 51 * h2)
```

La función `combineHashes` mezclará dos códigos resumen y redistribuirá el resultado sobre el intervalo 0-65535.

Escribamos una función que usa la restricción `Hashable` para restringir los tipos de sus entradas. Una tarea común que requiere una función resumen es determinar si dos valores se mapean al mismo código resumen. La relación `hashEqual` proporciona dicha capacidad:

```Haskell
hashEqual :: forall a. Hashable a => a -> a -> Boolean
hashEqual = eq `on` hash
```

Esta función usa la función `on` de `Data.Function` para definir la igualdad de resumen en términos de igualdad de códigos resumen, y debe leerse como una definición declarativa de la igualdad de resumen: dos valores son iguales a nivel de resumen si son iguales después de que cada valor haya pasado a través de la función `hash`.

Escribamos algunas instancias de `Hashable` para algunos tipos primitivos. Comencemos con una instancia para enteros. Ya que `HashCode` es en realidad un entero envuelto, esto es fácil, podemos usar la función auxiliar `hashCode`:

```haskell
instance hashInt :: Hashable Int where
  hash = hashCode
```

Podemos también definir una instancia simple para valores `Boolean` usando ajuste de patrones:

```haskell
instance hashBoolean :: Hashable Boolean where
  hash false = hashCode 0
  hash true  = hashCode 1
```

Con una instancia para resumir enteros, podemos crear una instancia para resumir `Char`s usando la función `toCharCode` de `Data.Char`:

```haskell
instance hashChar :: Hashable Char where
  hash = hash <<< toCharCode
```

Para definir una instancia para arrays, podemos mapear la función `hash` sobre los elementos del array (si el tipo elemental es también una instancia de `Hashable`) y entonces realizar un pliegue por la izquierda sobre los códigos resultantes usando la función `combineHashes`:

```haskell
instance hashArray :: Hashable a => Hashable (Array a) where
  hash = foldl combineHashes (hashCode 0) <<< map hash
```

Date cuenta de cómo construimos instancias usando las instancias más simples que ya hemos escrito. Usemos nuestra nueva instancia para `Array` para definir una instancia para `String`s, convirtiendo una `String` en un array de `Char`s:

```haskell
instance hashString :: Hashable String where
  hash = hash <<< toCharArray
```

¿Cómo podemos probar que estas instancias de `Hashable` satisfacen las leyes de clase de tipos que hemos enunciado antes? Necesitamos asegurarnos de que valores iguales tienen códigos resumen iguales. En casos como `Int`, `Char`, `String` y `Boolean`, esto es simple porque no hay valores de esos tipos que sean iguales en el sentido de `Eq` pero no sean idénticamente iguales. 

¿Y en el caso de los tipos más interesantes? Para probar la ley de clase para la instancia de `Array` podemos usar inducción sobre la longitud del array. El único array con longitud cero es `[]`. Cualquier par de arrays no vacíos son iguales sólo si tienen elementos iguales a la cabeza y sus colas son iguales, por la definición de `Eq` sobre arrays. Por hipótesis inductiva, las colas tienen códigos resumen iguales, y sabemos que los elementos de la cabeza tienen códigos resumen iguales si la instancia de `Hashable a` tiene que satisfacer la ley. Por lo tanto, los dos arrays tienen códigos resumen iguales, de manera que el `Hashable (Array a)` obedece la ley de clase de tipos también.

El código fuente para este capítulo incluye varios ejemplos de instancias `Hashable`, como instancias para los tipos `Maybe` y `Tuple`.  

X> ## Ejercicios
X>
X> 1. (Fácil) Usa PSCi para probar las funciones resumen para cada una de las instancias definidas.
X> 1. (Medio) Usa la función `hashEqual` para probar si un array tiene algún elemento duplicado, usando la igualdad de código resumen como una aproximación a la igualdad de valor. Recuerda comprobar la igualdad de valor usando `==` si se encuentra un duplicado. _Pista_: la función `nubBy` de `Data.Array` debe hacer esta tarea mucho más simple.
X> 1. (Medio) Escribe una instancia `Hashable` para el siguiente newtype que satisfaga la ley de clase de tipos:
X>
X>     ```haskell
X>     newtype Hour = Hour Int
X>     
X>     instance eqHour :: Eq Hour where
X>       eq (Hour n) (Hour m) = mod n 12 == mod m 12
X>     ```
X>     
X>     El newtype `Hour` y su instancia `Eq` representan el tipo de enteros módulo 12, de manera que 1 y 13 se identifican como iguales, por ejemplo. Prueba que la ley de clase de tipos se cumple para tu instancia.
X> 1. (Difícil) Prueba las leyes de clase de tipos para las instancias `Hashable` de `Maybe`, `Either` y `Tuple`.

## Conclusión

En este capítulo hemos visto las _clases de tipos_, una forma de abstracción orientada a tipos que permite formas potentes de reutilización de código. Hemos visto una colección de clases de tipos estándar de las bibliotecas estándar de PureScript, y hemos definido nuestra propia biblioteca basada en una clase de tipos para calcular códigos de función resumen.

Este capítulo también ha presentado la noción de leyes de clases de tipos, una técnica para probar propiedades acerca del código que usa clases de tipos como forma de abstracción. Las leyes de clases de tipos son parte de un tema más amplio llamado _razonamiento ecuacional_ (*equational reasoning*), en el que las propiedades de un lenguaje de programación y su sistema de tipos se usan para permitir razonamiento lógico acerca de sus programas. Esta es una idea importante y es un tema al que vamos a volver durante el resto del libro.
