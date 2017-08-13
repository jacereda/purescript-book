# Funciones y registros (*records*)

## Objetivos del capítulo

Este capítulo presenta dos elementos esenciales de los programas PureScript: funciones y registros. Además veremos cómo estructurar programas PureScript y cómo usar los tipos como una ayuda en el desarrollo de programas.

Construiremos una aplicación de agenda simple para manejar una lista de contactos. Este código presentará algunas nuevas ideas de la sintaxis de PureScript.

La interfaz de nuestra aplicación será el modo interactivo PSCi, pero sería posible tomar como base este código para construir una interfaz en JavaScript. De hecho, haremos exactamente eso en capítulos posteriores, añadiendo validación de formulario y funcionalidad de salvado/carga.

## Preparación del proyecto

El código fuente de este capítulo estará contenido en el fichero `src/Data/AddressBook.purs`. Este fichero comienza con una declaración de módulo y su lista de importación:

```haskell
module Data.AddressBook where

import Prelude

import Control.Plus (empty)
import Data.List (List(..), filter, head)
import Data.Maybe (Maybe)
```

Aquí importamos varios módulos:

- El módulo `Control.Plus` que define el valor `empty`.
- El módulo `Data.List` proporcionado por el paquete `purescript-lists` que puede instalarse usando Bower. Contiene unas cuantas funciones que necesitaremos para trabajar con listas enlazadas.
- El módulo `Data.Maybe` que define tipos de datos y funciones para trabajar con valores opcionales.

Date cuenta de que lo que importamos de estos módulos está listado explícitamente entre paréntesis. Esto es generalmente una buena práctica, ya que ayuda a evitar conflictos entre los símbolos importados.

Asumiendo que has clonado el repositorio con el código fuente del libro, el proyecto para este capítulo puede construirse usando Pulp mediante los siguientes comandos:

```text
$ cd chapter3
$ bower update
$ pulp build
```

## Tipos simples

PureScript define tres tipos integrados que se corresponden con los tipos primitivos de JavaScript: números, cadenas y booleanos. Están definidos en el módulo `Prim` que se importa de manera implícita por todos los módulos. Se llaman `Number`, `String` y `Boolean` respectivamente y puedes verlos en PSCi usando el comando `:type` para imprimir los tipos de algunos valores simples:

```text
$ pulp repl

> :type 1.0
Number

> :type "test"
String

> :type true
Boolean
```

PureScript define otros tipos integrados: enteros, caracteres, formaciones (*arrays* en adelante), registros (*records*) y funciones.

Los enteros se diferencian de los tipos de coma flotante de tipo `Number` en que carecen de coma decimal.

```text
> :type 1
Int
```

Los caracteres literales van rodeados por comillas simples, a diferencia de las cadenas literales que usan dobles comillas:

```text
> :type 'a'
Char
```

Los arrays se corresponden a arrays de JavaScript, pero al contrario que en JavaScript, todos los elementos de un array PureScript deben tener el mismo tipo:

```text
> :type [1, 2, 3]
Array Int

> :type [true, false]
Array Boolean

> :type [1, false]
Could not match type Int with Boolean.
```

El error del último ejemplo es un error del comprobador de tipos, que intenta _unificar_ sin éxito (es decir, igualar) los tipos de los dos elementos.

Los registros se corresponden con los objetos de JavaScript y los registros literales tienen la misma sintaxis que los objetos literales de JavaScript:

```text
> author = { name: "Phil", interests: ["Functional Programming", "JavaScript"] }

> :type author
{ name :: String
, interests :: Array String
}
```

Este tipo indica que el objeto especificado tiene dos _campos_, un campo `name` de tipo `String` y un campo `interests` que tiene tipo `Array String`, es decir, un array de cadenas.

Los campos de los registros se pueden acceder usando un punto, seguido por la etiqueta del campo a acceder:

```text
> author.name
"Phil"

> author.interests
["Functional Programming","JavaScript"]
```

Las funciones de PureScript se corresponden con las funciones de JavaScript. Las bibliotecas estándar de PureScript proporcionan un montón de ejemplos de funciones, y veremos más en este capítulo:

```text
> import Prelude
> :type flip
forall a b c. (a -> b -> c) -> b -> a -> c

> :type const
forall a b. a -> b -> a
```

Las funciones pueden ser definidas en el nivel superior de un fichero especificando los argumentos antes del signo igual:

```haskell
add :: Int -> Int -> Int
add x y = x + y
```

De manera alternativa, las funciones se pueden definir en línea usando una barra diagonal inversa seguida de una lista de nombres de argumento delimitada por espacios. Para introducir una declaración de varias líneas en PSCi, podemos entrar en "modo paste" usando el comando `:paste`. En este modo, las declaraciones se finalizan usando la secuencia de teclas _Control-D_:

```text
> :paste
… add :: Int -> Int -> Int
… add = \x y -> x + y
… ^D
```

Habiendo definido esta función en PSCi, podemos _aplicarla_ a sus argumentos separando los dos argumentos del nombre de la función mediante espacio en blanco:

```text
> add 10 20
30
```

## Tipos cuantificados

En la sección anterior, vimos los tipos de algunas funciones definidas en el Prelude. Por ejemplo, la función `flip` tenía el siguiente tipo:

```text
> :type flip
forall a b c. (a -> b -> c) -> b -> a -> c
```

La palabra clave `forall` indica aquí que `flip` tiene un _tipo universalmente cuantificado_. Significa que podemos sustituir cualquier tipo por `a`, `b` y `c`, y `flip` funcionará con esos tipos.

Por ejemplo, podemos elegir que el tipo de `a` sea `Int`, `b` sea `String` y `c` sea `String`. En ese caso, podríamos _especializar_ el tipo de `flip` a:

```text
(Int -> String -> String) -> String -> Int -> String
```

No tenemos que indicar en el código que queremos especializar un tipo cuantificado, sucede automáticamente. Por ejemplo, podemos simplemente usar `flip` como si ya tuviese este tipo:

```text
> flip (\n s -> show n <> s) "Ten" 10

"10Ten"
```

Aunque podemos elegir cualquier tipo para `a`, `b` y `c`, tenemos que ser consistentes. El tipo de la función que hemos pasado a `flip` tenía que ser consistente con los tipos de los otros argumentos. Es por eso que pasamos la cadena "Ten" como segundo argumento, y el número `10` como el tercero. No funcionaría si invirtiésemos los argumentos:

```text
> flip (\n s -> show n <> s) 10 "Ten"

Could not match type Int with type String
```

## Notas sobre la sangría (*indentation*)

El código PureScript es _sensible a la sangría_, al igual que Haskell y al contrario que JavaScript. Esto significa que el espacio en blanco de tu código no carece de significado. Se usa para agrupar regiones de código, de la misma manera que se usan las llaves en los lenguajes tipo C.

Si una declaración abarca múltiples líneas, entonces cualquier línea excepto la primera debe tener sangría más allá del nivel de la primera línea.

Así, lo siguiente es código PureScript válido:

```haskell
add x y z = x +
  y + z
```

Pero esto no es código válido:

```haskell
add x y z = x +
y + z
```

En el segundo caso, el compilador de PureScript intentará analizar _dos_ declaraciones, una por cada línea.

Generalmente, las declaraciones definidas en el mismo bloque deben tener sangría al mismo nivel. Por ejemplo, en PSCi, las declaraciones en una sentencia `let` deben deben tener la misma sangría. Esto es válido:

```text
> :paste
… x = 1
… y = 2
… ^D
```

Pero esto no lo es:

```text
> :paste
… x = 1
…  y = 2
… ^D
```

Algunas palabras clave de PureScript (como `where`, `of` y `let`) introducen un nuevo bloque de código, en el cual las declaraciones deben tener mayor nivel de sangría:

```haskell
example x y z = foo + bar
  where
    foo = x * y
    bar = y * z
```

Date cuenta de que las declaraciones de `foo` y `bar` tienen mayor nivel de sangría que la declaración de `example`.

La única excepción a esta regla es la palabra clave `where` en la declaración de `module` inicial al comienzo del fichero fuente.

## Definiendo nuestros tipos

Un buen primer paso cuando se aborda un nuevo problema en PureScript es escribir las definiciones de tipos para cualquier valor con el que vayas a trabajar. Primero, definamos un tipo para los registros de nuestra agenda:

```haskell
type Entry =
  { firstName :: String
  , lastName  :: String
  , address   :: Address
  }
```

Esto define un _sinónimo de tipo_ llamado `Entry`. El tipo `Entry` es equivalente al tipo a la derecha del símbolo igual: un registro con tres campos: `firstName`, `lastName` y `address`. Los dos campos de nombre tendrán tipo `String`, y el campo `address` tendrá tipo `Address` definido como sigue:

```haskell
type Address =
  { street :: String
  , city   :: String
  , state  :: String
  }
```

Date cuenta de que los registros pueden contener otros registros.

Ahora definamos un tercer sinónimo de tipo para nuestra estructura de datos de agenda, que será representada simplemente como una lista enlazada de entradas:

```haskell
type AddressBook = List Entry
```

Date cuenta de que `List Entry` no es lo mismo que `Array Entry`, que representa un array de entradas.

## Constructores de tipo (*type constructors*) y familias (*kinds*)

`List` es un ejemplo de un _constructor de tipo_. Los valores no tienen el tipo `List` directamente, sino `List a` para algún tipo `a`. Esto es, `List` toma un _argumento de tipo_ `a` y _construye_ un nuevo tipo `List a`.

Date cuenta de que al igual que la aplicación de función, los constructores de tipo se aplican a otros tipos simplemente por yuxtaposición: el tipo `List Entry` es de hecho el constructor de tipo `List` _aplicado_ al tipo `Entry`, representa una lista de entradas.

Si tratamos de definir incorrectamente un valor de tipo `List` (usando el operador de anotación de tipo `::`), veremos un nuevo tipo de error:

```text
> import Data.List
> Nil :: List
In a type-annotated expression x :: t, the type t must have kind Type
```

Esto es un _error de familia_. Al igual que los valores se distinguen por su _tipo_, los tipos se distinguen por su _familia_, y al igual que los valores erróneamente tipados acaban en _errores de tipo_, los tipos mal expresados resultan en _errores de familia_.

Hay una familia especial llamada `Type` que representa la familia de todos los tipos que tienen valores, como `Number` y `String`.

Hay también familias para constructores de tipo. Por ejemplo, la familia `Type -> Type` representa una función de tipos a tipos, como `List`. Así, el error ha ocurrido aquí porque se espera que los valores tengan tipos de familia `Type`, pero `List` tiene familia `Type -> Type`.

Para averiguar la familia de un tipo, usa el comando `:kind` en PSCi. Por ejemplo:

```text
> :kind Number
Type

> import Data.List
> :kind List
Type -> Type

> :kind List String
Type
```

El _sistema de familias_ de PureScript soporta otras familias interesantes que veremos más adelante en el libro.

## Mostrando entradas de la agenda

Escribamos nuestra primera función, que representará una entrada de la agenda como una cadena. Empezamos dando a la función un tipo. Esto es opcional, pero es una buena práctica, ya que actúa como una forma de documentación. De hecho, el compilador de PureScript emitirá un aviso si una declaración del nivel superior no contiene una anotación de tipo. Una declaración de tipo separa con el símbolo `::` el nombre de la función de su tipo:

```haskell
showEntry :: Entry -> String
```

La firma de tipo dice que `showEntry` es una función que toma `Entry` como argumento y devuelve una cadena. Aquí está el código para `showEntry`:

```haskell
showEntry entry = entry.lastName <> ", " <>
                  entry.firstName <> ": " <>
                  showAddress entry.address
```

Esta función concatena los tres campos del registro `Entry` en una única cadena, usando la función `showAddress` para convertir el registro contenido en el campo `address` a una cadena. `showAddress` se define de forma similar:

```haskell
showAddress :: Address -> String
showAddress addr = addr.street <> ", " <>
                   addr.city <> ", " <>
                   addr.state
```

Una definición de función comienza con el nombre de la función, seguida por una lista de nombres de argumento. El resultado de la función se especifica tras el signo igual. Los campos se acceden con un punto seguido del nombre de campo. En PureScript, la concatenación usa el operador diamante (`<>`), en lugar del operador de suma que usa JavaScript.

## Prueba temprano, prueba a menudo

El modo interactivo PSCi permite prototipado rápido con retroalimentación inmediata, así que usémoslo para verificar que nuestras primeras funciones se comportan como esperamos:

Primero, construye el código que has escrito:

```text
$ pulp build
```

A continuación, carga PSCi y usa el comando `import` para importar tu nuevo módulo:

```text
$ pulp repl

> import Data.AddressBook
```

Podemos crear una entrada usando un registro literal, que tiene el mismo aspecto que un objeto anónimo en JavaScript. Vamos a ligarlo a un nombre con una expresión `let`:

```text
> address = { street: "123 Fake St.", city: "Faketown", state: "CA" }
```

Ahora intenta aplicar nuestra función al ejemplo:

```text
> showAddress address

"123 Fake St., Faketown, CA"
```

Probemos también `showEntry` creando una entrada de la agenda conteniendo nuestra dirección de ejemplo:

```text
> entry = { firstName: "John", lastName: "Smith", address: address }
> showEntry entry

"Smith, John: 123 Fake St., Faketown, CA"
```

## Creando agendas

Ahora escribamos algunas funciones útiles para trabajar con agendas. Necesitaremos un valor que representa una agenda vacía: una lista vacía.

```haskell
emptyBook :: AddressBook
emptyBook = empty
```

Necesitaremos también una función para insertar un valor en una agenda existente. Llamaremos a esta función `insertEntry`. Comienza dándole su tipo:

```haskell
insertEntry :: Entry -> AddressBook -> AddressBook
```

La firma de tipo dice que `insertEntry` toma `Entry` como primer argumento y `AddressBook` como segundo argumento, y devuelve un nuevo `AddressBook`.

No modificamos el `AddressBook` directamente. En su lugar, devolvemos un nuevo `AddressBook` que contiene la nueva entrada. Así, `AddressBook` es un ejemplo de una _estructura de datos inmutable_. Esta es una idea importante en PureScript: la mutación es un efecto secundario del código e inhibe nuestra habilidad para razonar de manera efectiva sobre su comportamiento, de manera que preferimos funciones puras y datos inmutables donde sea posible.

Para implementar `insertEntry`, usamos la función `Cons` de `Data.List`. Para ver su tipo, abre PSCi y usa el comando `:type`:

```text
$ pulp repl

> import Data.List
> :type Cons

forall a. a -> List a -> List a
```

La firma de tipo dice que `Cons` toma un valor de cierto tipo `a` y una lista de elementos de tipo `a`, y devuelve una nueva lista con entradas del mismo tipo. Especialicemos esto con nuestro tipo `Entry` en el papel de `a`:

```haskell
Entry -> List Entry -> List Entry
```

Pero `List Entry` es lo mismo que `AddressBook`, de manera que esto es equivalente a:

```haskell
Entry -> AddressBook -> AddressBook
```

En nuestro caso, ya tenemos las entradas apropiadas: un `Entry` y un `AddressBook`, de manera que podemos aplicar `Cons` y obtener un nuevo `AddressBook`, ¡que es exactamente lo que queremos!

Aquí está nuestra implementación de `insertEntry`:

```haskell
insertEntry entry book = Cons entry book
```

Esto usa los dos argumentos `entry` y `book` declarados a la izquierda del símbolo igual y les aplica la función `Cons` para crear el resultado.

## Funciones currificadas (*curried functions*)

Las funciones en PureScript toman exactamente un argumento. Aunque parece que la función `insertEntry` toma dos argumentos, es de hecho un ejemplo de una _función currificada_.

El operador `->` en el tipo de `insertEntry` se asocia a la derecha, lo que significa que el compilador analiza el tipo como:

```haskell
Entry -> (AddressBook -> AddressBook)
```

Esto es, `insertEntry` es una función que devuelve una función. Toma un único argumento, un `Entry`, y devuelve una nueva función que a su vez toma un único argumento `AddressBook` y devuelve un nuevo `AddressBook`.

Esto significa que podemos _aplicar parcialmente_ `insertEntry` especificando únicamente su primer argumento, por ejemplo. En PSCi podemos ver el tipo resultante:

```text
> :type insertEntry entry

AddressBook -> AddressBook
```

Como esperábamos, el tipo de retorno es una función. Podemos aplicar la función resultante a un segundo argumento:

```text
> :type (insertEntry entry) emptyBook
AddressBook
```

Date cuenta de que los paréntesis son innecesarios. Lo que sigue es equivalente:

```text
> :type insertEntry example emptyBook
AddressBook
```

Esto es porque la aplicación de función asocia a la izquierda, y esto explica por qué podemos simplemente especificar argumentos de función uno tras otro, separados por espacio en blanco.

En el resto del libro hablaremos de cosas como "funciones de dos argumentos". Sin embargo, hay que entender que esto significa una función currificada que toma un primer argumento y devuelve otra función.

Ahora considera la definición de `insertEntry`:

```haskell
insertEntry :: Entry -> AddressBook -> AddressBook
insertEntry entry book = Cons entry book
```

Si ponemos entre paréntesis de manera explícita la parte derecha, tenemos `(Cons entry) book`. Esto es, `insertEntry entry` es una función cuyo argumento se pasa sin más a la función `(Cons entry)`. Pero si dos funciones tienen el mismo resultado para cualquier entrada entonces son la misma función. Así que podemos quitar el argumento `book` de ambos lados:

```haskell
insertEntry :: Entry -> AddressBook -> AddressBook
insertEntry entry = Cons entry
```

Pero ahora, con el mismo razonamiento, podemos quitar `entry` de ambos lados:

```haskell
insertEntry :: Entry -> AddressBook -> AddressBook
insertEntry = Cons
```

Este proceso se llama _conversión eta_, y se puede usar (junto a otras técnicas) para reescribir funciones en _forma libre de puntos_ (*point-free form*), que significa que las funciones están definidas sin referencia a sus argumentos.

En el caso de `insertEntry`, la _conversión eta_ ha resultado en una definición muy clara de nuestra función: "`insertEntry` es simplemente cons sobre listas". Sin embargo, es discutible si la forma libre de puntos es mejor en general.

## Consultando la agenda

La última función que necesitamos implementar para nuestra aplicación de agenda mínima buscará una persona por nombre y devolverá la `Entry` correcta. Esto será una buena aplicación de la idea de construir programas componiendo pequeñas funciones, una idea clave en la programación funcional.

Podemos primero filtrar la agenda, manteniendo sólo las entradas con los nombres y apellidos correctos. Entonces podemos simplemente devolver la cabeza (es decir, el primer elemento) de la lista resultante.

Con esta especificación de alto nivel de nuestra estrategia, podemos calcular el tipo de nuestra función. Primero abre PSCi y busca los tipos de las funciones `filter` y `head`:

```text
$ pulp repl

> import Data.List
> :type filter

forall a. (a -> Boolean) -> List a -> List a

> :type head

forall a. List a -> Maybe a
```

Vamos a desmontar estos dos tipos para entender su significado.

`filter` es una función currificada de dos argumentos. Su primer argumento es una función que toma un elemento de una lista y devuelve un valor `Boolean` como resultado. Su segundo argumento es una lista de elementos, y el valor de retorno es otra lista.

`head` toma una lista como argumento y devuelve un tipo que no hemos visto antes: `Maybe a`. `Maybe a` representa un valor opcional de tipo `a`, y proporciona una alternativa de tipo seguro al uso de `null` para indicar un valor inexistente en lenguajes como JavaScript. La veremos de nuevo en más detalle en capítulos posteriores.

Los tipos universalmente cuantificados de `filter` y `head` pueden ser _especializados_ por el compilador de PureScript a los siguientes tipos:

```haskell
filter :: (Entry -> Boolean) -> AddressBook -> AddressBook

head :: AddressBook -> Maybe Entry
```

Sabemos que necesitaremos pasar el nombre y apellidos que queremos buscar como argumentos a nuestra función.

También sabemos que necesitaremos una función para pasar a `filter`. Llamemos a esta función `filterEntry`. `filterEntry` tendrá tipo `Entry -> Boolean`. La aplicación `filter filterEntry` tendrá entonces tipo `AddressBook -> AddressBook`. Si pasamos el resultado de esta función a la función `head`, obtenemos nuestro resultado de tipo `Maybe Entry`.

Juntando esto, una firma de tipo razonable para nuestra función, que llamaremos `findEntry`, es:

```haskell
findEntry :: String -> String -> AddressBook -> Maybe Entry
```

Esta firma de tipo dice que `findEntry` toma dos cadenas, nombre y apellido, un `AddressBook`, y retorna un `Entry` opcional. El resultado opcional contendrá un valor sólo si el nombre se encuentra en la agenda.

Y aquí está la definición de `findEntry`:

```haskell
findEntry firstName lastName book = head $ filter filterEntry book
  where
    filterEntry :: Entry -> Boolean
    filterEntry entry = entry.firstName == firstName && entry.lastName == lastName
```

Vamos a repasar el código paso a paso.

`findEntry` pone en contexto tres nombres: `firstName` y `lastName`, ambos representando cadenas, y `book`, un `AddressBook`.

La parte derecha de la definición combina las funciones `filter` y `head`: primero, la lista de entradas es filtrada y luego, la función `head` se aplica al resultado.

La función predicado `filterEntry` se define como una declaración auxiliar dentro de una cláusula `where`. De esta manera, la función `filterEntry` está disponible dentro de la definición de nuestra función pero no fuera de ella. También, puede depender de los argumentos de la función contenedora, lo que es esencial aquí porque `filterEntry` usa los argumentos `firstName` y `lastName` para filtrar la `Entry` especificada.

Date cuenta de que al igual que en el caso de las declaraciones de nivel superior, no es necesario especificar una firma de tipo para `filterEntry`. Sin embargo, se recomienda hacerlo como una forma de documentación.

## Aplicación de funciones infija

En el código de `findEntry` de arriba, usamos una forma diferente de aplicación de función: la función `head` ha sido aplicada a la expresión `filter filterEntry book` usando el símbolo infijo `$`.

Esto es equivalente a la aplicación usual `head (filter filterEntry book)`.

`($)` es simplemente una función normal llamada `apply`, definida en el Prelude como sigue:

```haskell
apply :: forall a b. (a -> b) -> a -> b
apply f x = f x

infixr 0 apply as $
```

Así, `apply` toma una función y un valor, y aplica la función al valor. La palabra reservada `infixr` se usa para definir `($)` como un alias de `apply`.

Pero ¿por qué podríamos querer usar `$` en lugar de aplicación de función normal? La razón es que `$` es un operador de baja precedencia asociativo por la derecha. Esto significa que `$` nos permite quitar pares de paréntesis para aplicaciones anidadas profundamente.

Por ejemplo, la siguiente aplicación de función anidada que encuentra la calle en la dirección del jefe de un empleado:

```haskell
street (address (boss employee))
```

Es probablemente más legible cuando se expresa usando `$`:

```haskell
street $ address $ boss employee
```

## Composición de funciones

Al igual que hemos sido capaces de simplificar la función `insertEntry` usando conversión eta, podemos simplificar la definición de `findEntry` razonando sobre sus argumentos.

Date cuenta de que el argumento `book` se pasa a la función `filter filterEntry`, y el resultado de esta aplicación se pasa a `head`. En otras palabras, `book` se pasa a la _composición_ de las funciones `filter filterEntry` y `head`.

En PureScript, los operadores de composición de función son `<<<` y `>>>`. El primero es "composición hacia atrás" (*backwards composition*) y el segundo es "composición hacia delante" (*forwards composition*).

Podemos reescribir la parte derecha de `findEntry` usando cualquier operador. Usando composición hacia atrás, la parte derecha sería:

```
(head <<< filter filterEntry) book
```

En esta forma, podemos aplicar el truco anterior de conversión eta para llegar a la forma final de `findEntry`:

```haskell
findEntry firstName lastName = head <<< filter filterEntry
  where
    ...
```

Una parte derecha igualmente válida sería:

```haskell
filter filterEntry >>> head
```

De cualquier modo, esto nos da una definición clara de la función `findEntry`: "`findEntry` es la composición de una función de filtrado y la función `head`".

Voy a dejar que tomes tu propia decisión sobre qué definición es más fácil de entender, pero a menudo es útil pensar en las funciones como bloques de construcción de esta manera. Cada función ejecutando una única tarea y soluciones ensambladas usando composición de funciones.

## Prueba, prueba, prueba...

Ahora que tenemos el núcleo de una aplicación, probémosla usando PSCi:

```text
$ pulp repl

> import Data.AddressBook
```

Vamos primero a intentar buscar una entrada en una agenda vacía (obviamente esperamos que esto devuelva un resultado vacío):

```text
> findEntry "John" "Smith" emptyBook

No type class instance was found for

    Data.Show.Show { firstName :: String
                   , lastName :: String
                   , address :: { street :: String
                                , city :: String
                                , state :: String
                                }
                   }
```

¡Un error! No hay que preocuparse, este error simplemente significa que PSCi no sabe cómo imprimir un valor de tipo `Entry` como String.

El tipo de retorno de `findEntry` es `Maybe Entry`, que podemos convertir a `String` a mano.

Nuestra función `showEntry` espera un argumento de tipo `Entry`, pero tenemos un valor de tipo `Maybe Entry`. Recuerda que esto significa que la función devuelve un valor opcional de tipo `Entry`. Lo que necesitamos es aplicar la función `showEntry` si el valor opcional está presente y propagar el valor ausente si no lo está.

Afortunadamente, el módulo Prelude proporciona una manera de hacer esto. El operador `map` se puede usar para elevar (*lift*) una función sobre un constructor de tipo apropiado como `Maybe` (veremos más sobre esta función y otras como ella más tarde cuando hablemos de funtores):

```text
> import Prelude
> map showEntry (findEntry "John" "Smith" emptyBook)

Nothing
```

Eso está mejor. El valor de retorno `Nothing` indica que el valor de retorno opcional no contiene un valor, como esperábamos.

Para facilitar el uso, podemos crear una función que imprime `Entry` como una String, de manera que no tengamos que usar `showEntry` cada vez:

```text
> printEntry firstName lastName book = map showEntry (findEntry firstName lastName book)
```

Ahora creemos una agenda no vacía e intentemos de nuevo. Reutilizaremos nuestra entrada de ejemplo anterior:

```text
> book1 = insertEntry entry emptyBook

> printEntry "John" "Smith" book1

Just ("Smith, John: 123 Fake St., Faketown, CA")
```

Esta vez, el resultado contenía el valor correcto. Intenta definir una agenda `book2` con dos nombres insertando otro nombre en `book1` y busca cada entrada por nombre.

X> ## Ejercicios
X>
X> 1. (Fácil) Comprueba que entiendes la función `findEntry` escribiendo los tipos de cada una de sus expresiones principales. Por ejemplo, el tipo de la función `head` en este uso se especializa a `AddressBook -> Maybe Entry`.
X> 1. (Medio) Escribe una función que busca una `Entry` dada una dirección reutilizando el código existente en `findEntry`. Prueba tu función en PSCi.
X> 1. (Medio) Escribe una función que comprueba si un nombre aparece en `AddressBook` devolviendo un valor Boolean. _Pista_: Usa PSCi para buscar el tipo de la función `Data.List.null` que comprueba si una lista está o no vacía.
X> 1. (Difícil) Escribe una función `removeDuplicates` que elimina entradas duplicadas de la agenda con el mismo nombre y apellido. _Pista_: Usa PSCi para buscar el tipo de la función `Data.List.nubBy` que elimina elementos duplicados de una lista basándose en un predicado de igualdad.

## Conclusión

En este capítulo hemos cubierto varios conceptos de programación funcional:

- Cómo usar el modo interactivo PSCi para experimentar con funciones y probar ideas.
- El papel de los tipos como herramienta de corrección e implementación.
- El uso de funciones currificadas para representar funciones de múltiples argumentos.
- Crear programas a partir de componentes más pequeñas mediante composición.
- Estructurar el código de manera limpia usando expresiones `where`.
- Como evitar valores nulos usando el tipo `Maybe`.
- El uso de técnicas como la conversión eta y la composición de funciones para refactorizar código.

En los siguientes capítulos nos basaremos en estas ideas.
