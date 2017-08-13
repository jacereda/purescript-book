# Lenguajes específicos del dominio (*domain-specific languages*)

## Objetivos del capítulo

En este capítulo exploraremos la implementación de _lenguajes específicos del dominio_ (o _DSLs_) en PureScript, usando un número de técnicas estándar.

Un lenguaje específico del dominio es un lenguaje que es particularmente apropiado para desarrollar en un dominio de problemas concreto. Su sintaxis y funciones se eligen para maximizar la legibilidad del código usado para expresar ideas en ese dominio. Hemos visto varios ejemplos de lenguajes específicos del dominio en este libro:

- La mónada `Game` y sus acciones asociadas, desarrolladas en el capítulo 11, constituyen un lenguaje específico del dominio para el dominio del _desarrollo de juegos de aventuras textuales_.
- La biblioteca de combinadores que escribimos para los funtores `Async` y `Parallel` del capítulo 12 se pueden considerar un ejemplo de lenguaje específico del dominio para el dominio de la _programación asíncrona_.
- El paquete `purescript-quickcheck` cubierto en el capítulo 13 es un lenguaje específico del dominio para el dominio de _verificación generativa_. Sus combinadores permiten una notación particularmente expresiva para escribir propiedades de verificación.

Este capítulo tomará una aproximación más estructurada a algunas de las técnicas estándar para la implementación de lenguajes específicos del dominio. No es de ninguna manera una exposición completa del tema, pero debe proporcionarte suficiente conocimiento para construir algunos DSLs prácticos para tus tareas.

Nuestro ejemplo será un lenguaje específico del dominio para crear documentos HTML. Nuestro objetivo será desarrollar un lenguaje de tipos seguros para describir documentos HTML correctos, y trabajaremos mejorando en pequeños pasos una implementación ingenua.

## Preparación del proyecto

El proyecto que acompaña este capítulo añade una nueva dependencia Bower; la biblioteca `purescript-free` que define la _mónada libre_ (*free monad*), una de las herramientas que usaremos.

Probaremos el proyecto de este capítulo en PSCi. 

## Un tipo de datos HTML

La versión más básica de nuestra biblioteca HTML está definida el módulo `Data.DOM.Simple`. El módulo contiene las siguientes definiciones de tipos:

```haskell
newtype Element = Element
  { name         :: String
  , attribs      :: Array Attribute
  , content      :: Maybe (Array Content)
  }

data Content
  = TextContent String
  | ElementContent Element

newtype Attribute = Attribute
  { key          :: String
  , value        :: String
  }
```

El tipo `Element` representa elementos HTML. Cada elemento consiste en un nombre de elemento, un array de pares de atributos y algún contenido. La propiedad de contenido usa el tipo `Maybe` para indicar si un elemento puede estar abierto (contiene otros elementos y texto) o cerrado.

La función clave de nuestra biblioteca es una función

```haskell
render :: Element -> String
```

que representa elementos HTML como cadenas HTML. Podemos probar esta versión de la biblioteca construyendo valores de los tipos apropiados explícitamente en PSCi:

```text
$ pulp repl

> import Prelude
> import Data.DOM.Simple
> import Data.Maybe
> import Control.Monad.Eff.Console

> :paste
… log $ render $ Element
…   { name: "p"
…   , attribs: [
…       Attribute
…         { key: "class"
…         , value: "main"
…         }
…     ]
…   , content: Just [
…       TextContent "Hello World!"
…     ]
…   }
… ^D

<p class="main">Hello World!</p>
unit
```

Esta biblioteca tiene varios problemas tal cual está:

- Crear documentos HTML es difícil; cada nuevo elemento requiere al menos un registro y un constructor de datos.
- Es posible representar documentos inválidos:
    - El desarrollador puede escribir mal el nombre del elemento
    - El desarrollador puede asociar un atributo con el tipo de elemento erróneo
    - El desarrollador puede usar un elemento cerrado cuando lo correcto es un elemento abierto

En el resto del capítulo, aplicaremos ciertas técnicas para resolver estos problemas y convertir nuestra biblioteca en un lenguaje específico del dominio para crear documentos HTML.

## Constructores inteligentes (*smart constructors*)

La primera técnica que aplicaremos es simple pero puede ser muy efectiva. En lugar de exponer la representación de los datos a los usuarios del módulo, podemos usar la lista de exportaciones del módulo para ocultar los constructores de datos `Element`, `Content` y `Attribute`, y exportar únicamente los llamados _constructores inteligentes_, que construyen datos que se saben correctos.

Aquí tenemos un ejemplo. Primero proporcionamos una función de conveniencia para crear elementos HTML:

```haskell
element :: String -> Array Attribute -> Maybe (Array Content) -> Element
element name attribs content = Element
  { name:      name
  , attribs:   attribs
  , content:   content
  }
```

A continuación, creamos constructores inteligentes para aquellos elementos HTML que queremos que puedan crear nuestros usuarios, aplicando la función `element`:

```haskell
a :: Array Attribute -> Array Content -> Element
a attribs content = element "a" attribs (Just content)

p :: Array Attribute -> Array Content -> Element
p attribs content = element "p" attribs (Just content)

img :: Array Attribute -> Element
img attribs = element "img" attribs Nothing
```

Finalmente, actualizamos la lista de exportaciones del módulo para que exporte sólo las funciones que sabemos que construyen estructuras de datos correctas:

```haskell
module Data.DOM.Smart
  ( Element
  , Attribute(..)
  , Content(..)

  , a
  , p
  , img

  , render
  ) where
```

La lista de exportaciones del módulo se proporciona entre paréntesis inmediatamente después del nombre de módulo. Cada exportación del módulo puede ser de tres tipos:

- Un valor (o función) indicado/a por su nombre.
- Una clase de tipos indicada por el nombre de la clase.
- Un constructor de tipo y cualquier constructor de datos asociado, indicado por el nombre del tipo seguido por una lista entre paréntesis de constructores de datos exportados.

Aquí exportamos el _tipo_ `Element`, pero no exportamos sus constructores de datos. Si lo hiciésemos, el usuario sería capaz de construir elementos HTML inválidos.

En el caso de los tipos `Attribute` y `Content`, seguimos exportando todos los constructores de datos (usando el símbolo `..` en la lista de exportación). Aplicaremos la técnica de los constructores inteligentes a estos tipos en breve.

Fíjate en que ya hemos hecho grandes mejoras a nuestra biblioteca:

- Es imposible representar elementos HTML con nombres inválidos (por supuesto, estamos restringidos al conjunto de nombres de elemento proporcionados por la biblioteca).
- Los elementos cerrados no pueden tener contenido por construcción.

Podemos aplicar esta técnica al tipo `Content` muy fácilmente. Simplemente quitamos los constructores de datos del tipo `Content` de la lista de exportación y proporcionamos los siguientes constructores inteligentes:

```haskell
text :: String -> Content
text = TextContent

elem :: Element -> Content
elem = ElementContent
```

Apliquemos la misma técnica al tipo `Attribute`. Primero proporcionamos un constructor inteligente de propósito general para los atributos. Aquí hay un primer intento:

```haskell
attribute :: String -> String -> Attribute
attribute key value = Attribute
  { key: key
  , value: value
  }

infix 4 attribute as :=
```

Esta representación sufre del mismo problema que el tipo `Element` original; es posible representar atributos que no existen o cuyos nombres se han escrito mal. Para resolver este problema, podemos crear un newtype que representa nombres de atributo:

```haskell
newtype AttributeKey = AttributeKey String
```

Con eso, podemos modificar nuestro operador como sigue:

```haskell
attribute :: AttributeKey -> String -> Attribute
attribute (AttributeKey key) value = Attribute
  { key: key
  , value: value
  }
```

Si no exportamos el constructor de datos `AttributeKey`, el usuario no tiene manera de construir valores de tipo `AttributeKey` que no sea usando las funciones que exportamos explícitamente. Aquí hay algunos ejemplos:

```haskell
href :: AttributeKey
href = AttributeKey "href"

_class :: AttributeKey
_class = AttributeKey "class"

src :: AttributeKey
src = AttributeKey "src"

width :: AttributeKey
width = AttributeKey "width"

height :: AttributeKey
height = AttributeKey "height"
```

Aquí tenemos la lista final de exportaciones de nuestro nuevo módulo. Date cuenta de que ya no exportamos ningún constructor de datos directamente:

```haskell
module Data.DOM.Smart
  ( Element
  , Attribute
  , Content
  , AttributeKey

  , a
  , p
  , img

  , href
  , _class
  , src
  , width
  , height

  , attribute, (:=)
  , text
  , elem

  , render
  ) where
```

Si probamos este nuevo módulo en PSCi, podemos ver grandes mejoras en la concisión del código de usuario:

```text
$ pulp repl

> import Prelude
> import Data.DOM.Smart
> import Control.Monad.Eff.Console
> log $ render $ p [ _class := "main" ] [ text "Hello World!" ]

<p class="main">Hello World!</p>
unit
```

Fíjate sin embargo en que no tuvimos que hacer cambios a la función `render`, ya que la representación de los datos subyacentes no ha cambiado. Este es uno de los beneficios de la aproximación de los constructores inteligentes: nos permite separar la representación interna de los datos de un módulo y la representación percibida por los usuarios de su API externa.

X> ## Ejercicios
X>
X> 1. (Fácil) Usa el módulo `Data.DOM.Smart` para experimentar creando nuevos documentos HTML usando `render`.
X> 1. (Medio) Algunos atributos HTML como `checked` y `disabled` no requieren valores, y pueden representarse como _atributos vacíos_:
X>
X>     ```html
X>     <input disabled>
X>     ```
X>
X>     Modifica la representación de un `Attribute` para considerar los atributos vacíos. Escribe una función que se pueda usar en lugar de `attribute` o `:=` para añadir un atributo vacío a un elemento.

## Tipos fantasma (*phantom types*)

Para motivar la siguiente técnica, considera el siguiente código:

```text
> log $ render $ img
    [ src    := "cat.jpg"
    , width  := "foo"
    , height := "bar"
    ]

<img src="cat.jpg" width="foo" height="bar" />
unit
```

El problema aquí es que hemos proporcionado valores cadena para los atributos `width` y `height`, donde se esperaban valores numéricos en unidades de píxeles o porcentajes.

Para resolver este problema, podemos introducir un argumento de _tipo fantasma_ a nuestro tipo `AttributeKey`:

```haskell
newtype AttributeKey a = AttributeKey String
```

La variable de tipo `a` se llama _tipo fantasma_ porque no hay valores de tipo `a` involucrados en la parte derecha de la definición. El tipo `a` sólo existe para proporcionar más información en tiempo de compilación. Cualquier valor de tipo `AttributeKey a` es simplemente una cadena en tiempo de ejecución, pero en tiempo de compilación, el tipo del valor nos dice el tipo deseado para los valores asociados con esta clave.

Podemos modificar el tipo de nuestra función `attribute` para que tome en consideración la nueva forma de `AttributeKey`:

```haskell
attribute :: forall a. IsValue a => AttributeKey a -> a -> Attribute
attribute (AttributeKey key) value = Attribute
  { key: key
  , value: toValue value
  }
```

Aquí, el argumento de tipo fantasma `a` se usa para asegurarnos de que la clave y valor del atributo tienen tipos compatibles. Como el usuario no puede crear valores de tipo `AttributeKey a` directamente (sólo mediante las constantes que proporcionamos en la biblioteca), todos los atributos serán correctos por construcción.

Fíjate en que la restricción `IsValue` nos asegura que asociemos el tipo de valor que asociemos a una clave, sus valores se pueden convertir en cadenas para mostrarse en el HTML generado. La clase de tipos `IsValue` se define como sigue:

```haskell
class IsValue a where
  toValue :: a -> String
```

Proporcionamos instancias de la clase de tipos para los tipos `String` e `Int`:

```haskell
instance stringIsValue :: IsValue String where
  toValue = id

instance intIsValue :: IsValue Int where
  toValue = show
```

Tenemos también que actualizar nuestras constantes `AttributeKey` de manera que sus tipos reflejen el nuevo parámetro de tipo:

```haskell
href :: AttributeKey String
href = AttributeKey "href"

_class :: AttributeKey String
_class = AttributeKey "class"

src :: AttributeKey String
src = AttributeKey "src"

width :: AttributeKey Int
width = AttributeKey "width"

height :: AttributeKey Int
height = AttributeKey "height"
```

Ahora nos encontramos con que es imposible representar estos documentos HTML inválidos, y que estamos obligados a usar números para representar los atributos `width` y `height`:

```text
> import Prelude
> import Data.DOM.Phantom
> import Control.Monad.Eff.Console

> :paste
… log $ render $ img
…   [ src    := "cat.jpg"
…   , width  := 100
…   , height := 200
…   ]
… ^D

<img src="cat.jpg" width="100" height="200" />
unit
```

X> ## Ejercicios
X>
X> 1. (Fácil) Crea un tipo de datos que represente longitudes como píxeles o porcentajes. Escribe una instancia de `IsValue` para tu tipo. Modifica los atributos `width` y `height` para que usen tu nuevo tipo.
X> 1. (Difícil) Definiendo representantes a nivel de tipos de los valores booleanos `true` y `false`, podemos usar un tipo fantasma para codificar si un `AttributeKey` representa un _atributo vacío_ como `disabled` o `checked`.
X>
X>     ```haskell
X>     data True
X>     data False
X>     ```
X>
X>     Modifica tu solución al ejercicio anterior para usar un tipo fantasma con el fin de evitar que el usuario use el operador `attribute` con un atributo vacío.

## La mónada libre (*free monad*)

En nuestro conjunto final de modificaciones a nuestra API, usaremos un constructor llamado la _mónada libre_ para convertir nuestro tipo `Content` en una mónada, habilitando la notación do. Esto nos permitirá estructurar nuestro documento HTML en una forma en que el anidamiento de elementos se vuelve más claro; en lugar de esto:

```haskell
p [ _class := "main" ]
  [ elem $ img
      [ src    := "cat.jpg"
      , width  := 100
      , height := 200
      ]
  , text "A cat"
  ]
```

podremos escribir esto:

```haskell
p [ _class := "main" ] $ do
  elem $ img
    [ src    := "cat.jpg"
    , width  := 100
    , height := 200
    ]
  text "A cat"
```

Sin embargo, la notación do no es el único beneficio de una mónada libre. La mónada libre nos permite separar la _representación_ de nuestras acciones monádicas de su _interpretación_, e incluso soportar _múltiples interpretaciones_ de las mismas acciones.

La mónada `Free` se define en la biblioteca `purescript-free` en el módulo `Control.Monad.Free`. Podemos averiguar alguna información básica sobre ella en PSCi como sigue:

```text
> import Control.Monad.Free

> :kind Free
(Type -> Type) -> Type -> Type
```

La familia de `Free` indica que toma un constructor de tipo como argumento y devuelve otro constructor de tipo. De hecho, ¡la mónada `Free` se puede usar para convertir cualquier `Functor` en una `Monad`!

Comenzamos definiendo la _representación_ de nuestras acciones monádicas. Para hacer esto necesitamos crear un `Functor` con un constructor de datos para cada acción monádica que deseamos soportar. En nuestro caso, nuestras dos acciones monádicas serán `elem` y `text`. De hecho podemos simplemente modificar nuestro tipo `Content` como sigue:

```haskell
data ContentF a
  = TextContent String a
  | ElementContent Element a

instance functorContentF :: Functor ContentF where
  map f (TextContent s x) = TextContent s (f x)
  map f (ElementContent e x) = ElementContent e (f x)
```

Aquí el constructor de tipo `ContentF` se parece a nuestro viejo tipo de datos `Content`; sin embargo, ahora toma un argumento de tipo `a`, y cada constructor de datos se ha modificado para que tome un valor de tipo `a` como argumento adicional. La instancia `Functor` simplemente aplica la función `f` al valor de tipo `a` en cada constructor de datos.

Con eso, podemos definir nuestra nueva mónada `Content` como un sinónimo de tipo para la mónada `Free`, que construimos usando nuestro constructor de tipo `ContentF` como primer argumento de tipo:

```haskell
type Content = Free ContentF
```

En lugar de un sinónimo de tipo podemos usar un `newtype` para evitar exponer la representación interna de nuestra biblioteca a los usuarios. Ocultando el constructor de datos `Content` restringimos a nuestros usuarios a que usen únicamente las acciones monádicas que suministramos.

Como `ContentF` es un `Functor`, obtenemos automáticamente una instancia `Monad` para `Free ContentF`.

Tenemos que modificar nuestro tipo de datos `Element` ligeramente para tomar en cuenta el nuevo argumento de tipo de `Content`. Simplemente requeriremos que el tipo de retorno de nuestros cálculos monádicos sea `Unit`:

```haskell
newtype Element = Element
  { name         :: String
  , attribs      :: Array Attribute
  , content      :: Maybe (Content Unit)
  }
```

Además tenemos que modificar nuestras funciones `elem` y `text`, que se convertirán en nuestras nuevas acciones monádicas para la mónada `Content`. Para hacer esto podemos usar la función `liftF` suministrada por el módulo `Control.Monad.Free`. Aquí está su tipo:

```haskell
liftF :: forall f a. f a -> Free f a
```

`liftF` nos permite construir una acción en nuestra mónada libre a partir de un valor de tipo `f a` para algún tipo `a`. En nuestro caso, podemos simplemente usar los constructores de datos de nuestro constructor de tipo `ContentF` directamente:

```haskell
text :: String -> Content Unit
text s = liftF $ TextContent s unit

elem :: Element -> Content Unit
elem e = liftF $ ElementContent e unit
```

Hay que hacer otras modificaciones rutinarias, pero los cambios interesantes están en la función `render`, donde tenemos que _interpretar_ nuestra mónada libre.

## Interpretando la mónada

El módulo `Control.Monad.Free` proporciona funciones para interpretar un cálculo en una mónada libre:

```haskell
runFree
  :: forall f a
   . Functor f
  => (f (Free f a) -> Free f a)
  -> Free f a
  -> a

runFreeM
  :: forall f m a
   . (Functor f, MonadRec m)
  => (f (Free f a) -> m (Free f a))
  -> Free f a
  -> m a
```

La función `runFree` se usa para calcular un resultado _puro_. La función `runFreeM` nos permite usar una mónada para interpretar las acciones de nuestra mónada libre.

_Nota_: Técnicamente, estamos restringidos a usar monadas `m` que satisfagan la restricción más fuerte `MonadRec`. En la práctica, esto significa que no necesitamos preocuparnos por los desbordamientos de pila, ya que `m` soporta _recursividad final mónadica_ segura.

Primero tenemos que elegir una mónada en la que podamos interpretar nuestras acciones. Usaremos la mónada `Writer String` para acumular una cadena HTML como resultado.

Nuestro nuevo método `render` comienza delegando en una función auxiliar `renderElement` y usando `execWriter` para ejecutar nuestro cálculo en la mónada `Writer`: 

```haskell
render :: Element -> String
render = execWriter <<< renderElement
```

`renderElement` se define en un bloque where:

```haskell
  where
    renderElement :: Element -> Writer String Unit
    renderElement (Element e) = do
```

La definición de `renderElement` es simple, usa la acción `tell` de la mónada `Writer` para acumular varias cadenas pequeñas:

```haskell
      tell "<"
      tell e.name
      for_ e.attribs $ \x -> do
        tell " "
        renderAttribute x
      renderContent e.content
```

A continuación definimos la función `renderAttribute` que es igualmente simple:

```haskell
    where
      renderAttribute :: Attribute -> Writer String Unit
      renderAttribute (Attribute x) = do
        tell x.key
        tell "=\""
        tell x.value
        tell "\""
```

La función `renderContent` es más interesante. Aquí usamos la función `runFreeM` para interpretar el cálculo dentro de la mónada libre, delegando en una función auxiliar `renderContentItem`:

```haskell
      renderContent :: Maybe (Content Unit) -> Writer String Unit
      renderContent Nothing = tell " />"
      renderContent (Just content) = do
        tell ">"
        runFreeM renderContentItem content
        tell "</"
        tell e.name
        tell ">"
```

El tipo de `renderContentItem` se puede deducir de la firma de tipo de `runFreeM`. El funtor `f` es nuestro constructor de tipo `ContentF`, y la mónada `m` es la mónada en la que estamos interpretando el cálculo, a saber, `Writer String`. Esto nos da la siguiente firma de tipo para `renderContentItem`:

```haskell
      renderContentItem :: ContentF (Content Unit) -> Writer String (Content Unit)
```

Podemos implementar esta función simplemente mediante ajuste de patrones sobre los dos constructores de datos de `ContentF`:

```haskell
      renderContentItem (TextContent s rest) = do
        tell s
        pure rest
      renderContentItem (ElementContent e rest) = do
        renderElement e
        pure rest
```

En cada caso, la expresión `rest` tiene el tipo `Content Unit`, y representa el resto del cálculo interpretado. Podemos completar cada caso devolviendo la acción `rest`.

¡Eso es todo! Podemos probar nuestra nueva API monádica en PSCi como sigue:

```text
> import Prelude
> import Data.DOM.Free
> import Control.Monad.Eff.Console

> :paste
… log $ render $ p [] $ do
…   elem $ img [ src := "cat.jpg" ]
…   text "A cat"
… ^D

<p><img src="cat.jpg" />A cat</p>
unit
```

X> ## Ejercicios
X>
X> 1. (Medio) Añade un nuevo constructor de datos al tipo `ContentF` para soportar una nueva acción `comment`, que representa un comentario en el HTML generado. Implementa la nueva acción usando `liftF`. Actualiza la interpretación de `renderContentItem` para interpretar tu nuevo constructor de manera apropiada.

## Extendiendo el lenguaje

Una mónada en la que todas las acciones devuelven algo de tipo `Unit` no es particularmente interesante. De hecho, aparte de una sintaxis probablemente más bonita, nuestra mónada no añade funcionalidad extra con respecto a un `Monoid`.

Ilustremos la potencia de la construcción de mónada libre extendiendo nuestro lenguaje con una nueva acción monádica que devuelve un resultado no trivial.

Supongamos que queremos generar documentos HTML que contienen hipervínculos a diferentes secciones del documento usando _anclas_ (*anchors*). Podemos conseguir esto actualmente generando nombres de ancla a mano e incluyéndolos al menos dos veces en el documento: una vez en la definición de la propia ancla, y otra en cada hipervínculo. Sin embargo, este enfoque tiene algunos problemas básicos:

- El desarrollador puede equivocarse generando nombres de ancla únicos.
- El desarrollador puede equivocarse al escribir una o más instancias del nombre del ancla.

Queriendo proteger al desarrollador de sus propios errores, podemos introducir un nuevo tipo que representa nombres de ancla y proporcionar una acción monádica para generar nuevos nombres únicos.

El primer paso es añadir un nuevo tipo para los nombres:

```haskell
newtype Name = Name String

runName :: Name -> String
runName (Name n) = n
```

De nuevo, definimos esto como un newtype en torno a `String`, pero debemos ser cuidadosos de no exportar el constructor de datos en la lista de exportaciones del módulo.

A continuación definimos una instancia de la clase de tipos `IsValue` para nuestro nuevo tipo, de manera que seamos capaces de usar nombres en valores de atributo:

```haskell
instance nameIsValue :: IsValue Name where
  toValue (Name n) = n
```

Definimos también un nuevo tipo de datos para hipervínculos que pueden aparecer en elementos `a` como sigue:

```haskell
data Href
  = URLHref String
  | AnchorHref Name

instance hrefIsValue :: IsValue Href where
  toValue (URLHref url) = url
  toValue (AnchorHref (Name nm)) = "#" <> nm
```

Con este nuevo tipo, podemos modificar el tipo de valor del atributo `href`, forzando a nuestros usuarios a usar nuestro nuevo tipo `Href`. Podemos también crear un nuevo atributo `name` que se puede usar para convertir un elemento en un ancla:

```haskell
href :: AttributeKey Href
href = AttributeKey "href"

name :: AttributeKey Name
name = AttributeKey "name"
```

El problema restante es que nuestros usuarios no tienen actualmente manera de generar nombres nuevos. Podemos proporcionar esta funcionalidad en nuestra mónada `Content`. Primero necesitamos añadir un nuevo constructor de datos a nuestro constructor de tipo `ContentF`:

```haskell
data ContentF a
  = TextContent String a
  | ElementContent Element a
  | NewName (Name -> a)
```

El constructor de datos `NewName` corresponde a una acción que devuelve un valor de tipo `Name`. Fíjate en que en lugar de requerir un `Name` como argumento de constructor de datos, requerimos que el usuario proporcione una _función_ de tipo `Name -> a`. Recordando que el tipo `a` representa el _resto del cálculo_, podemos ver que esta función proporciona una manera de continuar el cálculo después de que un valor de tipo `Name` se haya devuelto.

Necesitamos también actualizar la instancia `Functor` de `ContentF` para que tenga cuenta el nuevo constructor de datos como sigue:

```haskell
instance functorContentF :: Functor ContentF where
  map f (TextContent s x) = TextContent s (f x)
  map f (ElementContent e x) = ElementContent e (f x)
  map f (NewName k) = NewName (f <<< k)
```

Podemos ahora construir nuestra nueva acción mediante la función `liftF` como antes:

```haskell
newName :: Content Name
newName = liftF $ NewName id
```

Date cuenta de que proporcionamos la función `id` como nuestra continuación, lo que significa que devolvemos el resultado de tipo `Name` sin cambiar.

Finalmente necesitamos actualizar nuestra función de interpretación para interpretar la nueva acción. Previamente hemos usado la mónada `Writer String` para interpretar nuestros cálculos, pero esa mónada no tiene la capacidad de generar nuevos nombres, de manera que cambiamos a otra cosa. El transformador de mónada `WriterT` se puede usar con la mónada `State` para combinar los efectos que necesitamos. Definimos nuestra mónada de interpretación como un sinónimo de tipo para mantener nuestras firmas de tipo cortas:

```haskell
type Interp = WriterT String (State Int)
```

Aquí, el estado de tipo `Int` actuará como un contador incremental usado para generar nombres únicos.

Ya que las mónadas `Writer` y `WriterT` usan los mismos miembros de clase de tipos para abstraer sus acciones, no necesitamos cambiar ninguna acción; sólo tenemos que cambiar todas las referencias a `Writer String` por `Interp`. Sin embargo tenemos que modificar el gestor usado para ejecutar nuestro cálculo. En lugar de usar `execWriter`, tenemos ahora que usar `evalState` y `execWriter`:

```haskell
render :: Element -> String
render e = evalState (execWriterT (renderElement e)) 0
```

También necesitamos añadir un nuevo caso a `renderContentItem` para interpretar el constructor de datos `NewName`:

```haskell
renderContentItem (NewName k) = do
  n <- get
  let fresh = Name $ "name" <> show n
  put $ n + 1
  pure (k fresh)
```

Aquí se nos da una continuación `k` de tipo `Name -> Content a`, y necesitamos construir una interpretación de tipo `Content a`. Nuestra interpretación es simple: usamos `get` para leer el estado, usamos ese estado para generar un nombre único, y usamos `put` para incrementar el estado. Finalmente pasamos nuestro nuevo nombre a la continuación para completar el cálculo.

Con eso, podemos probar nuestra nueva funcionalidad en PSCi, generando un nombre único dentro de la mónada `Content` y usándolo tanto como nombre de elemento como destino de un hipervínculo:

```text
> import Prelude
> import Data.DOM.Name
> import Control.Monad.Eff.Console

> :paste
… render $ p [ ] $ do
…   top <- newName
…   elem $ a [ name := top ] $
…     text "Top"
…   elem $ a [ href := AnchorHref top ] $
…     text "Back to top"
… ^D

<p><a name="name0">Top</a><a href="#name0">Back to top</a></p>
unit
```

Puedes verificar que múltiples llamadas a `newName` resultan de hecho en nombres únicos.

X> ## Ejercicios
X>
X> 1. (Medio) Podemos simplificar más la API escondiendo el tipo `Element` de los usuarios. Haz estos cambios siguiendo estos pasos:
X>     
X>     - Combina funciones como `p` e `img` (con tipo de retorno `Element`) con la acción `elem` para crear nuevas acciones con tipo de retorno `Content Unit`.
X>     - Cambia la función `render` para que acepte un argumento de tipo `Content Unit` en lugar de `Element`.
X> 1. (Medio) Esconde la implementación de la mónada `Content` usando un newtype en lugar de un sinónimo de tipo. No debes exportar el constructor de datos para tu newtype.
X> 1. (Difícil) Modifica el tipo `ContentF` para soportar una nueva acción
X>
X>     ```haskell
X>     isMobile :: Content Boolean
X>     ```
X>
X>     que devuelve un valor booleano indicando si el documento debe o no representarse para ser mostrado en un dispositivo móvil.
X>
X>     _Hint_: usa la acción `ask` y el transformador de mónada `ReaderT` para interpretar esta acción. Alternativamente, puedes preferir usar la mónada `RWS`.

## Conclusión

En este capítulo hemos desarrollado un lenguaje específico del dominio para crear documentos HTML, mejorando incrementalmente una implementación ingenua usando algunas técnicas estándar:

- Hemos usado _constructores inteligentes_ para esconder detalles de nuestra representación de datos, permitiendo únicamente al usuario crear documentos que sean _correctos por construcción_.
- Hemos usado un _operador binario infijo definido por el usuario_ para mejorar la sintaxis del lenguaje.
- Hemos usado _tipos fantasma_ para codificar información adicional en los tipos de nuestros datos, evitando que el usuario proporcione valores de tipo erróneo.
- Hemos usado la _mónada libre_ para convertir nuestra representación array del contenido en una representación monádica que soporta notación do. Hemos extendido esta representación para soportar una nueva acción monádica, y hemos interpretado los cálculos monádicos usando transformadores de mónada estándar.

Estas técnicas aprovechan el sistema de tipos y módulos de JavaScript, bien para evitar que el usuario cometa errores, bien para mejorar la sintaxis del lenguaje específico del dominio.

La implementación de lenguajes específicos del dominio en lenguajes de programación funcional es un área de investigación activa, pero con suerte esto proporciona una introducción útil a algunas técnicas simples e ilustra la potencia de trabajar en un lenguaje con tipos expresivos.
