# Validación aplicativa (*applicative validation*)

## Objetivos del capítulo

En este capítulo, vamos a conocer una importante abstracción nueva, el _funtor aplicativo_ (*applicative functor*), descrito por la clase de tipos `Applicative`. No te preocupes si el nombre suena raro. Daremos un motivo para el concepto con un ejemplo práctico: validar datos de formulario. Esta técnica nos permite convertir código que normalmente implica un montón de código de comprobación repetitivo en una descripción declarativa de nuestro formulario.

Veremos también otra clase de tipos, `Traversable`, que describe los _funtores transitables_ (*traversable functors*), y veremos cómo este concepto también aparece de manera muy natural a partir de soluciones a problemas del mundo real.

El código de ejemplo para este capítulo será una continuación del ejemplo de la agenda del capítulo 3. Esta vez, extenderemos los tipos de datos de nuestra agenda y escribiremos funciones para validar los valores de esos tipos. Estas funciones podrían usarse, por ejemplo, en una interfaz de usuario web para mostrar errores al usuario como parte del formulario de entrada de datos.

## Preparación del proyecto

El código fuente para este capítulo está definido en los ficheros `src/Data/AddressBook.purs` y `src/Data/AddressBook/Validation.purs`.

El proyecto tiene unas cuantas dependencias Bower, muchas de las cuales ya hemos visto. Hay dos dependencias nuevas:

- `purescript-control`, que define funciones para abstraer el control de flujo usando clases de tipos como `Applicative`.
- `purescript-validation`, que define un funtor para _validación aplicativa_, el tema de este capítulo.

El módulo `Data.AddressBook` define tipos de datos e instancias `Show` para los tipos de nuestro proyecto, y el módulo `Data.AddressBook.Validation` contiene reglas de validación para esos tipos.

## Generalizando la aplicación de funciones

Para explicar el concepto de un _funtor aplicativo_, consideremos el constructor de tipo `Maybe` que vimos antes.

El código fuente para este módulo define una función `address` que tiene el siguiente tipo:

```haskell
address :: String -> String -> String -> Address
```

Esta función se usa para construir valores de tipo `Address` a partir de tres cadenas: un nombre de calle, una ciudad y un estado.

Podemos aplicar esta función fácilmente y ver el resultado en PSCi:

```text
> import Data.AddressBook

> address "123 Fake St." "Faketown" "CA"
Address { street: "123 Fake St.", city: "Faketown", state: "CA" }
```

Sin embargo, supongamos que no necesariamente tendremos una calle, ciudad, o estado, y queremos usar el tipo `Maybe` para indicar la ausencia de valor en cada uno de esos tres casos.  

En un caso, podemos carecer de la ciudad. Si intentamos aplicar nuestra función directamente, recibiremos un error del comprobador de tipos:

```text
> import Data.Maybe
> address (Just "123 Fake St.") Nothing (Just "CA")

Could not match type

  Maybe String

with type

  String
```

Por supuesto, esperábamos este error. `address` toma cadenas como argumentos, no valores de tipo `Maybe String`.

Sin embargo, es razonable esperar que podamos "elevar" la función `address` para trabajar con valores opcionales descritos por el tipo `Maybe`. De hecho podemos, y el módulo `Control.Apply` proporciona la función `lift3` que hace exactamente lo que necesitamos: 

```text
> import Control.Apply
> lift3 address (Just "123 Fake St.") Nothing (Just "CA")

Nothing
```

En este caso, el resultado es `Nothing` porque uno de los argumentos (la ciudad) no está presente. Si proporcionamos los tres argumentos usando el constructor `Just`, el resultado contendrá un valor también:

```text
> lift3 address (Just "123 Fake St.") (Just "Faketown") (Just "CA")

Just (Address { street: "123 Fake St.", city: "Faketown", state: "CA" })
```

El nombre de la función `lift3` indica que se puede usar para elevar funciones de 3 argumentos. Hay funciones similares definidas en `Control.Apply` para funciones de otro número de argumentos.

## Elevando funciones arbitrarias

Podemos elevar funciones de un pequeño número de argumentos usando `lift2`, `lift3`, etc. ¿Pero cómo podemos generalizar esto a funciones arbitrarias?

Es instructivo ver el tipo de `lift3`:

```text
> :type lift3
forall a b c d f. Apply f => (a -> b -> c -> d) -> f a -> f b -> f c -> f d
```

En el ejemplo de `Maybe` anterior, el constructor de tipo `f` es `Maybe`, de manera que `lift3` se especializa al siguiente tipo:

```haskell
forall a b c d. (a -> b -> c -> d) -> Maybe a -> Maybe b -> Maybe c -> Maybe d
```

Este tipo dice que podemos tomar cualquier función de tres argumentos y elevarla para darnos una nueva función cuyos tipos de argumento y resultado están envueltos en `Maybe`.

Ciertamente esto no es posible para cualquier constructor de tipo `f`, de manera que ¿qué tiene Maybe que nos permite hacer esto? Bien, al especializar el tipo antes, hemos quitado la restricción de clase de tipos sobre `f` de la clase `Apply`. `Apply` se define en el Prelude como sigue:

```haskell
class Functor f where
  map :: forall a b. (a -> b) -> f a -> f b

class Functor f <= Apply f where
  apply :: forall a b. f (a -> b) -> f a -> f b
```

La clase de tipos `Apply` es una subclase de `Functor` y define una función adicional `apply`. Al igual que `<$>` se ha definido como un sinónimo de `map`, el módulo `Prelude` define `<*>` como un alias para `apply`. Como veremos, estos dos operadores se usan a menudo juntos.

El tipo de `apply` se parece bastante al tipo de `map`. La diferencia entre `map` y `apply` es que `map` toma una función como argumento, mientras que el primer argumento de `apply` está envuelto en el constructor de tipo `f`. Veremos cómo se usa pronto, pero primero veamos cómo implementar la clase de tipos `Apply` para el tipo `Maybe`:

```haskell
instance functorMaybe :: Functor Maybe where
  map f (Just a) = Just (f a)
  map f Nothing  = Nothing

instance applyMaybe :: Apply Maybe where
  apply (Just f) (Just x) = Just (f x)
  apply _        _        = Nothing
```

Esta instancia de clase de tipos dice que podemos aplicar una función opcional a un valor opcional, y el resultado está definido sólo si ambos están definidos.

Ahora veremos cómo `map` y `apply` se pueden usar juntas para elevar funciones de un número arbitrario de argumentos.

Para funciones de un argumento, podemos simplemente usar `map` directamente.

Para funciones de dos argumentos, digamos que tenemos una función currificada `g` de tipo `a -> b -> c`. Esto es equivalente al tipo `a -> (b -> c)`, de manera que podemos aplicar `map` a `g` para obtener una nueva función de tipo `f a -> f (b -> c)` para cualquier constructor de tipo `f` con una instancia de `Functor`. Al aplicar parcialmente esta función al primer argumento elevado (de tipo `f a`), obtenemos una nueva función envuelta de tipo `f (b -> c)`. Si también tenemos una instancia de `Apply` para `f`, podemos entonces usar `apply` para aplicar el segundo argumento elevado (de tipo `f b`) para obtener nuestro valor final de tipo `f c`.

Para juntarlo todo, vemos que si tenemos valores `x :: f a` y `y :: f b`, entonces la expresión `(g <$> x) <*> y` tiene tipo `f c` (recuerda, esta expresión es equivalente a `apply (map g x) y`). Las reglas de precedencia definidas en el Prelude nos permiten quitar los paréntesis: `g <$> x <*> y`.

En general, podemos usar `<$>` sobre el primer argumento y `<*>` para los argumentos restantes, como se muestra aquí para `lift3`:

```haskell
lift3 :: forall a b c d f
       . Apply f
      => (a -> b -> c -> d)
      -> f a
      -> f b
      -> f c
      -> f d
lift3 f x y z = f <$> x <*> y <*> z
```

Se deja como ejercicio para el lector verificar los tipos involucrados en esta expresión.

Como ejemplo, podemos intentar elevar la función `address` sobre `Maybe` directamente usando las funciones `<$>` y `<*>`:

```text
> address <$> Just "123 Fake St." <*> Just "Faketown" <*> Just "CA"
Just (Address { street: "123 Fake St.", city: "Faketown", state: "CA" })

> address <$> Just "123 Fake St." <*> Nothing <*> Just "CA"
Nothing
```

Intenta elevar otras funciones de varios argumentos sobre `Maybe` de esta manera.

## La clase de tipos Applicative

Hay una clase de tipos relacionada llamada `Applicative`, definida como sigue:

```haskell
class Apply f <= Applicative f where
  pure :: forall a. a -> f a
```

`Applicative` es una subclase de `Apply` y define la función `pure`. `pure` toma un valor y devuelve un valor cuyo tipo ha sido envuelto en el constructor de tipo `f`.

Aquí está la instancia `Applicative` para `Maybe`:

```haskell
instance applicativeMaybe :: Applicative Maybe where
  pure x = Just x
```

Si pensamos en los funtores aplicativos como funtores que permiten elevar funciones, entonces `pure` puede verse como una función que eleva funciones de cero argumentos.

## Intuición para Applicative

Las funciones en PureScript son puras y no soportan efectos secundarios. Los funtores aplicativos nos permiten trabajar en "lenguajes de programación" más grandes que soportan algún tipo de efectos secundarios codificados por el funtor `f`.

Por ejemplo, el funtor `Maybe` representa el efecto secundario de valores potencialmente ausentes. Otros ejemplos incluyen `Either err`, que representa el efecto secundario de posibles errores de tipo `err`, y el funtor flecha `r ->` que representa el efecto secundario de leer de una configuración global. Por ahora consideraremos el funtor `Maybe`.

Si el funtor `f` representa este lenguaje de programación más grande con efectos, entonces las instancias `Apply` y `Applicative` nos permiten elevar valores y aplicación de funciones de nuestro lenguaje de programación más pequeño (PureScript) al nuevo lenguaje.

`pure` eleva valores puros (libres de efectos secundarios) al lenguaje mayor, y para funciones podemos usar `map` y `apply` como hemos descrito antes.

Esto plantea una pregunta: si podemos usar `Applicative` para empotrar funciones y valores PureScript en este nuevo lenguaje, ¿de que manera es el nuevo lenguaje mayor? La respuesta depende del funtor `f`. Si podemos encontrar expresiones de tipo `f a` que no pueden expresarse como `pure x` para algún `x`, entonces esa expresión representa un término que sólo existe en el lenguaje mayor.

Cuando `f` es `Maybe`, un ejemplo es la expresión `Nothing`: no podemos escribir `Nothing` como `pure x` para cualquier `x`. Por lo tanto, podemos pensar que PureScript se ha agrandado para incluir el nuevo término `Nothing` que representa un valor ausente. 

## Más efectos

Veamos unos cuantos ejemplos de elevar funciones sobre distintos funtores aplicativos.

Aquí hay una simple función de ejemplo definida en PSCi que une tres nombres para formar un nombre completo:

```text
> import Prelude

> fullName first middle last = last <> ", " <> first <> " " <> middle

> fullName "Phillip" "A" "Freeman"
Freeman, Phillip A
```

Ahora supongamos que esta función forma la implementación de un servicio web con tres argumentos proporcionados como parámetros de la consulta. Queremos asegurarnos de que el usuario ha proporcionado los tres parámetros, así que usamos el tipo `Maybe` para indicar la presencia o ausencia de un parámetro. Podemos elevar `fullName` sobre `Maybe` para crear una implementación del servicio web que comprueba parámetros ausentes:

```text
> import Data.Maybe

> fullName <$> Just "Phillip" <*> Just "A" <*> Just "Freeman"
Just ("Freeman, Phillip A")

> fullName <$> Just "Phillip" <*> Nothing <*> Just "Freeman"
Nothing
```

Date cuenta de que la función elevada devuelve `Nothing` si cualquiera de los argumentos era `Nothing`.

Esto es bueno, porque ahora podemos devolver una respuesta de error desde nuestro servicio web si los parámetros son inválidos. Sin embargo, sería mejor si pudiésemos indicar qué campo era incorrecto en la respuesta. 

En lugar de elevar sobre `Maybe`, podemos elevar sobre `Either String` que permite devolver un mensaje de error. Primero, escribamos un operador para convertir entradas opcionales en cálculos que señalan un error usando `Either String`:

```text
> :paste
… withError Nothing  err = Left err
… withError (Just a) _   = Right a
… ^D
```

_Nota_: En el funtor aplicativo `Either err`, el constructor `Left` indica un error y el `Right` indica éxito.

Ahora podemos elevar sobre `Either String` proporcionando un mensaje de error apropiado para cada parámetro:

```text
> :paste
… fullNameEither first middle last =
…   fullName <$> (first  `withError` "First name was missing")
…            <*> (middle `withError` "Middle name was missing")
…            <*> (last   `withError` "Last name was missing")
… ^D

> :type fullNameEither
Maybe String -> Maybe String -> Maybe String -> Either String String
```

Ahora nuestra función toma tres parámetros opcionales usando `Maybe` y devuelve o bien un mensaje de error `String` o un resultado `String`.

Podemos probar la función con diferentes entradas:

```text
> fullNameEither (Just "Phillip") (Just "A") (Just "Freeman")
(Right "Freeman, Phillip A")

> fullNameEither (Just "Phillip") Nothing (Just "Freeman")
(Left "Middle name was missing")

> fullNameEither (Just "Phillip") (Just "A") Nothing
(Left "Last name was missing")
```

En este caso, vemos que el mensaje de error correspondiente al primer campo ausente, o un resultado exitoso si todos los campos han sido proporcionados. Sin embargo, si carecemos de varias entradas sólo vemos el primer error:

```text
> fullNameEither Nothing Nothing Nothing
(Left "First name was missing")
```

Esto puede ser suficientemente bueno, pero si queremos ver una lista de todos los campos ausentes en el error necesitamos algo más potente que `Either String`. Veremos una solución más adelante en este capítulo.

## Combinando efectos

Como ejemplo de la forma de trabajar con funtores aplicativos de manera abstracta, esta sección mostrará cómo escribir una función que combinará de manera genérica efectos secundarios codificados por un funtor aplicativo `f`.

¿Qué significa esto? Bien, supongamos que tenemos una lista de valores envueltos de tipo `f a` para algún `a`. Esto es, supongamos que tenemos una lista de tipo `List (f a)`. De manera intuitiva, esto representa una lista de cálculos con efectos secundarios registrados por `f`, cada uno con tipo de retorno `a`. Si pudiésemos ejecutar todos estos cálculos en orden, obtendríamos una lista de resultados de tipo `List a`. Sin embargo, seguiríamos teniendo efectos secundarios registrados por `f`. Esto es, esperamos ser capaces de convertir algo de tipo `List (f a)` en algo de tipo `f (List a)` "combinando" los efectos dentro de la lista original.

Para cualquier lista de tamaño fijo `n`, hay una función de `n` argumentos que construye una lista de tamaño `n` a partir de esos argumentos. Por ejemplo, si `n` es `3`, la función es `\x y z -> x : y : z : Nil`. Esta función tiene tipo `a -> a -> a -> List a`. Podemos usar la instancia de `Applicative` para `List` para elevar esta función sobre `f`, obteniendo una función de tipo `f a -> f a -> f a -> f (List a)`. Pero ya que podemos hacer esto para cualquier `n`, tiene sentido que podamos ser capaces de realizar la misma elevación para cualquier _lista_ de argumentos. 

Esto significa que debemos ser capaces de escribir esta función:

```haskell
combineList :: forall f a. Applicative f => List (f a) -> f (List a)
```

Esta función toma una lista de argumentos, que posiblemente tienen efectos secundarios, y devolverá una única lista envuelta, aplicando los efectos secundarios de cada uno.

Para escribir esta función, consideraremos la longitud de la lista de argumentos. Si la lista está vacía, no necesitamos realizar ningún efecto y podemos usar `pure` para simplemente devolver una lista vacía:

```haskell
combineList Nil = pure Nil
```

De hecho, !esto es lo único que podemos hacer!

Si la lista no está vacía, tenemos un elemento a la cabeza que es un argumento envuelto de tipo `f a`, y una cola de tipo `List (f a)`. Podemos combinar los efectos de manera recursiva en la cola, devolviendo un resultado de tipo `f (List a)`. Podemos entonces usar `<$>` y `<*>` para elevar el constructor `Cons` sobre la cabeza y la nueva cola:

```haskell
combineList (Cons x xs) = Cons <$> x <*> combineList xs
```

De nuevo, esta era la única implementación sensata basándose en los tipos que nos han dado.

Podemos probar esta función en PSCi, usando el constructor de tipo `Maybe` como ejemplo:

```text
> import Data.List
> import Data.Maybe

> combineList (fromFoldable [Just 1, Just 2, Just 3])
(Just (Cons 1 (Cons 2 (Cons 3 Nil))))

> combineList (fromFoldable [Just 1, Nothing, Just 2])
Nothing
```

Cuando se especializa a `Maybe`, nuestra función devuelve un `Just` sólo si todos los elementos de la lista eran `Just`, de otra manera devuelve `Nothing`. Esto es consistente con nuestra intuición de trabajar en un lenguaje mayor que soporta valores opcionales; una lista de cálculos que devuelven valores opcionales sólo tiene un resultado si todos los cálculos contenían un resultado.

Pero la función `combineList` ¡funciona para cualquier `Applicative`! Podemos usarla para combinar cálculos que posiblemente señalan un error usando `Either err`, o que leen de una configuración global usando `r ->`.

Veremos la función `combineList` de nuevo más tarde cuando consideremos los funtores `Traversable`.

X> ## Ejercicios
X>
X> 1. (Fácil) Usa `lift2` para escribir versiones elevadas de los operadores `+`, `-`, `*` y `/` que funcionen con valores opcionales.
X> 1. (Medio) Convéncete de que la definición de `lift3` dada antes en términos de `<$>` y `<*>` pasa la comprobación de tipos.
X> 1. (Difícil) Escribe una función `combineMaybe` de tipo `forall a f. Applicative f => Maybe (f a) -> f (Maybe a)`. Esta función toma un cálculo opcional con efectos secundarios y devuelve un cálculo con efectos secundarios que tiene un resultado opcional.

## Validación aplicativa

El código fuente para este capítulo define varios tipos de datos que pueden ser usados en aplicaciones de agenda. Omitimos los detalles aquí, pero las funciones clave que exporta el módulo `Data.AddressBook` tienen los siguientes tipos:

```haskell
address :: String -> String -> String -> Address

phoneNumber :: PhoneType -> String -> PhoneNumber

person :: String -> String -> Address -> Array PhoneNumber -> Person
```

Donde `PhoneType` se define como un tipo de datos algebraico:

```haskell
data PhoneType = HomePhone | WorkPhone | CellPhone | OtherPhone
```

Estas funciones se pueden usar para construir una `Person` representando una entrada de la agenda. Por ejemplo, el siguiente valor está definido en `Data.AddressBook`:

```haskell
examplePerson :: Person
examplePerson =
  person "John" "Smith"
         (address "123 Fake St." "FakeTown" "CA")
  	     [ phoneNumber HomePhone "555-555-5555"
         , phoneNumber CellPhone "555-555-0000"
  	     ]
```

Prueba este valor en PSCi (hemos dado formato al resultado):

```text
> import Data.AddressBook

> examplePerson
Person
  { firstName: "John",
  , lastName: "Smith",
  , address: Address
      { street: "123 Fake St."
      , city: "FakeTown"
      , state: "CA"
      },
  , phones: [ PhoneNumber
                { type: HomePhone
                , number: "555-555-5555"
                }
            , PhoneNumber
                { type: CellPhone
                , number: "555-555-0000"
                }
            ]
  }  
```

Vimos en una sección anterior cómo podíamos usar el funtor `Either String` para validar estructuras de datos de tipo `Person`. Por ejemplo, dadas las funciones para validar los dos nombres de la estructura, podemos validar la estructura de datos completa como sigue:

```haskell
nonEmpty :: String -> Either String Unit
nonEmpty "" = Left "Field cannot be empty"
nonEmpty _  = Right unit

validatePerson :: Person -> Either String Person
validatePerson (Person o) =
  person <$> (nonEmpty o.firstName *> pure o.firstName)
         <*> (nonEmpty o.lastName  *> pure o.lastName)
         <*> pure o.address
         <*> pure o.phones
```

En las dos primeras líneas, usamos la función `nonEmpty` para validar una cadena no vacía. `nonEmpty` devuelve un código de error (indicado por el constructor `Left`) si su entrada es vacía, o un valor exitoso vacío (`unit`) usando el constructor `Right` en caso contrario. Usamos el operador de secuenciación `*>` para indicar que queremos realizar dos validaciones, devolviendo el resultado del validador de la derecha. En este caso, el validador de la derecha simplemente usa `pure` para devolver la entrada sin cambios.

Las líneas finales no realizan ninguna validación sino que simplemente proporcionan los campos `address` y `phones` a la función `person` como argumentos restantes.

Podemos ver que esta función funciona en PSCi, pero tiene una limitación que ya hemos visto;

```haskell
> validatePerson $ person "" "" (address "" "" "") []
(Left "Field cannot be empty")
```

El funtor aplicativo `Either String` sólo proporciona el primer error encontrado. Dada la entrada que hemos pasado, preferiríamos ver dos errores, uno para el nombre y otro para el apellido.

Hay otro funtor aplicativo proporcionado por la biblioteca `purescript-validation`. Este funtor se llama `V` y proporciona la capacidad de devolver errores en cualquier _semigrupo_. por ejemplo, podemos usar `V (Array String)` para devolver un array de `String`s como errores, concatenando nuevos errores al final del array.

El módulo `Data.AddressBook.Validation` usa el funtor aplicativo `V (Array String)` para validar las estructuras de datos del módulo `Data.AddressBook`.

Aquí hay un ejemplo de validador tomado del módulo `Data.AddressBook.Validation`:

```haskell
type Errors = Array String

nonEmpty :: String -> String -> V Errors Unit
nonEmpty field "" = invalid ["Field '" <> field <> "' cannot be empty"]
nonEmpty _     _  = pure unit

lengthIs :: String -> Number -> String -> V Errors Unit
lengthIs field len value | S.length value /= len =
  invalid ["Field '" <> field <> "' must have length " <> show len]
lengthIs _     _   _     =
  pure unit

validateAddress :: Address -> V Errors Address
validateAddress (Address o) =
  address <$> (nonEmpty "Street" o.street *> pure o.street)
          <*> (nonEmpty "City"   o.city   *> pure o.city)
          <*> (lengthIs "State" 2 o.state *> pure o.state)
```

`validateAddress` valida una estructura `Address`. Comprueba que los campos `street` y `city` no están vacíos y comprueba que la cadena del campo `state` tiene longitud 2.

Date cuenta de cómo las funciones validadoras `nonEmpty` y `lengthIs` usan la función `invalid` proporcionada por el módulo `Data.Validation` para indicar un error. Como estamos trabajando en el semigrupo `Array String`, `invalid` toma un array de cadenas como argumento.

Podemos probar esta función en PSCi:

```text
> import Data.AddressBook
> import Data.AddressBook.Validation

> validateAddress $ address "" "" ""
(Invalid [ "Field 'Street' cannot be empty"
         , "Field 'City' cannot be empty"
         , "Field 'State' must have length 2"
         ])

> validateAddress $ address "" "" "CA"
(Invalid [ "Field 'Street' cannot be empty"
         , "Field 'City' cannot be empty"
         ])
```

Esta vez, recibimos un array de todos los errores de validación.

## Validadores con expresiones regulares (*regular expression validators*)

La función `validatePhoneNumber` usa una expresión regular para validar la forma de sus argumentos. La clave es una función de validación `matches`, que usa una `Regex` del módulo `Data.String.Regex` para validar su entrada:

```haskell
matches :: String -> R.Regex -> String -> V Errors Unit
matches _     regex value | R.test regex value =
  pure unit
matches field _     _     =
  invalid ["Field '" <> field <> "' did not match the required format"]
```

De nuevo, date cuenta de cómo `pure` se usa para indicar una validación exitosa, e `invalid` se usa para señalar un array de errores.

`validatePhoneNumber` se construye sobre `matches` igual que antes:

```haskell
validatePhoneNumber :: PhoneNumber -> V Errors PhoneNumber
validatePhoneNumber (PhoneNumber o) =
  phoneNumber <$> pure o."type"
              <*> (matches "Number" phoneNumberRegex o.number *> pure o.number)
```

De nuevo, intenta ejecutar este validador contra entradas válidas e inválidas en PSCi:

```text
> validatePhoneNumber $ phoneNumber HomePhone "555-555-5555"
Valid (PhoneNumber { type: HomePhone, number: "555-555-5555" })

> validatePhoneNumber $ phoneNumber HomePhone "555.555.5555"
Invalid (["Field 'Number' did not match the required format"])
```

X> ## Ejercicios
X>
X> 1. (Fácil) Usa un validador de expresiones regulares para asegurarte de que el campo `state` del tipo `Address` contiene dos caracteres alfabéticos. _Pista_: mira el código fuente de `phoneNumberRegex`.
X> 1. (Medio) Usando el validador `matches`, escribe una función de validación que compruebe que una cadena no está compuesta completamente por espacios en blanco. Úsala para reemplazar `nonEmpty` donde sea apropiado.

## Funtores transitables (*traversable functors*)

El validador restante es `validatePerson`, que combina los validadores que hemos visto hasta ahora para validar una estructura `Person` completa:

```haskell
arrayNonEmpty :: forall a. String -> Array a -> V Errors Unit
arrayNonEmpty field [] =
  invalid ["Field '" <> field <> "' must contain at least one value"]
arrayNonEmpty _     _  =
  pure unit

validatePerson :: Person -> V Errors Person
validatePerson (Person o) =
  person <$> (nonEmpty "First Name" o.firstName *>
              pure o.firstName)
         <*> (nonEmpty "Last Name"  o.lastName  *>
              pure o.lastName)
	       <*> validateAddress o.address
         <*> (arrayNonEmpty "Phone Numbers" o.phones *>
              traverse validatePhoneNumber o.phones)
```

Hay otra función interesante aquí que no hemos visto todavía: `traverse`, que aparece en la última línea.

`traverse` se define en el módulo `Data.Traversable` en la clase de tipos `Traversable`:

```haskell
class (Functor t, Foldable t) <= Traversable t where
  traverse :: forall a b f. Applicative f => (a -> f b) -> t a -> f (t b)
  sequence :: forall a f. Applicative f => t (f a) -> f (t a)
```

`Traversable` define la clase de _funtores transitables_. Los tipos de sus funciones pueden resultar un poco intimidatorios, pero `validatePerson` proporciona un buen ejemplo del motivo.

Todo funtor transitable es a la vez un `Functor` y un `Foldable` (recuerda que un _funtor plegable_ era un constructor de tipo que soportaba una operación de pliegue, reduciendo una estructura a un valor único). Además, un funtor transitable proporciona la capacidad de combinar una colección de efectos secundarios que dependen de su estructura.

Esto puede sonar complicado, pero simplifiquemos las cosas especializando para el caso de los arrays. El constructor de tipo array es transitable, lo que significa que hay una función:

```haskell
traverse :: forall a b f. Applicative f => (a -> f b) -> Array a -> f (Array b)
```

De manera intuitiva, dado cualquier funtor aplicativo `f` y una función que toma un valor de tipo `a` y devuelve un valor de tipo `b` (con efectos secundarios registrados por `f`), podemos aplicar la función a cada elemento del array de tipo `Array a` para obtener un resultado de tipo `Array b` (con efectos secundarios registrados por `f`).

¿Todavía no está claro? Especialicemos todavía más al caso en que `m` es el funtor aplicativo `V Errors` de arriba. Ahora tenemos una función de tipo:

```haskell
traverse :: forall a b. (a -> V Errors b) -> Array a -> V Errors (Array b)
```

Esta firma de tipo dice que si tenemos una función de validación `f` para un tipo `a`, entonces `traverse f` es una función de validación para arrays de tipo `Array a`. ¡Pero eso es exactamente lo que necesitamos para poder validar el campo `phones` de la estructura de datos `Person`! Podemos pasar `validatePhoneNumber` a `traverse` para crear una función de validación que valida cada elemento sucesivamente.

En general, `traverse` recorre los elementos de una estructura de datos, realizando cálculos con efectos secundarios y acumulando un resultado.

La firma de tipo para la otra función de `Traversable`, `sequence`, nos puede parecer más familiar:

```haskell
sequence :: forall a f. Applicative f => t (f a) -> f (t a)
```

De hecho, la función `combineList` que escribimos antes es sólo un caso especial de la función `sequence` de la clase de tipos `Traversable`. Fijando `t` para que sea el constructor de tipo `List`, podemos recuperar el tipo de la función `combineList`:

```haskell
combineList :: forall f a. Applicative f => List (f a) -> f (List a)
```

Los funtores transitables capturan la idea de recorrer una estructura de datos, recopilando un conjunto de cálculos con efectos, y combinando sus efectos. De hecho, `sequence` y `traverse` son igualmente importantes para la definición de `Traversable`; cada uno puede ser implementado a partir del otro. Esto se deja como un ejercicio para el lector interesado.

La instancia `Traversable` para listas está en el módulo `Data.List`. La definición de `traverse` es esta:

```haskell
-- traverse :: forall a b f. Applicative f => (a -> f b) -> List a -> f (List b)
traverse _ Nil = pure Nil
traverse f (Cons x xs) = Cons <$> f x <*> traverse f xs
```

En el caso de una lista vacía podemos simplemente devolver una lista vacía usando `pure`. Si la lista no está vacía, podemos usar la función `f` para crear un cálculo de tipo `f b` a partir del elemento frontal. Podemos también llamar a `traverse` recursivamente sobre la cola. Finalmente, podemos elevar el constructor `Cons` sobre el funtor aplicativo `f` para combinar estos dos resultados.

Pero hay más ejemplos de funtores transitables aparte de arrays y listas. El constructor de tipo `Maybe` que vimos antes también tiene una instancia de `Traversable`. Podemos probarlo en PSCi:

```text
> import Data.Maybe
> import Data.Traversable

> traverse (nonEmpty "Example") Nothing
(Valid Nothing)

> traverse (nonEmpty "Example") (Just "")
(Invalid ["Field 'Example' cannot be empty"])

> traverse (nonEmpty "Example") (Just "Testing")
(Valid (Just unit))
```

Estos ejemplos muestran que recorrer el valor `Nothing` devuelve `Nothing` sin validación, y recorrer `Just x` usa la función de validación para validar `x`. Esto es, `traverse` toma una función de validación para el tipo `a` y devuelve una función de validación para `Maybe a`, es decir, una función de validación para valores opcionales de tipo `a`.

Otros funtores transitables son `Array a`, `Tuple a` y `Either a` para cualquier tipo `a`. Generalmente, la mayoría de constructores de tipos de datos "contenedores" tienen instancias de `Traversable`. Como ejemplo, los ejercicios incluyen escribir una instancia de `Traversable` par un tipo de árboles binarios.

X> ## Ejercicios
X>
X> 1. (Medio) Escribe una instancia de `Traversable` para la siguiente estructura de datos de árbol binario, que combina efectos secundarios de izquierda a derecha:
X>
X>     ```haskell
X>     data Tree a = Leaf | Branch (Tree a) a (Tree a)
X>     ```
X>
X>     Esto corresponde a un recorrido in-orden. ¿Y para un recorrido pre-orden? ¿Y para uno post-orden?
X>
X> 1. (Medio) Modifica el código para hacer que el campo `address` del tipo `Person` sea opcional usando `Data.Maybe`. _Pista_: Usa `traverse` para validar un campo de tipo `Maybe a`.
X> 1. (Difícil) Intenta escribir `sequence` en términos de `traverse`. ¿Puedes escribir `traverse` en términos de `sequence`?

## Funtores aplicativos para paralelismo

Anteriormente he elegido la palabra "combinar" para describir cómo los funtores aplicativos "combinan efectos secundarios". Sin embargo, en todos los ejemplos dados, sería igualmente válido decir que los funtores aplicativos nos permiten "secuenciar" efectos. Esto sería consistente con la intuición de que los funtores transitables proporcionan una función `sequence` para combinar efectos en secuencia basados en una estructura de datos. 

Sin embargo, en general, los funtores aplicativos son más generales que esto. Las leyes de funtor aplicativo no imponen ningún orden para los efectos secundarios que sus cálculos realizan. De hecho, sería válido para un funtor aplicativo realizar sus efectos secundarios en paralelo.

Por ejemplo, el funtor de validación `V` devolvía un _array_ de errores, pero funcionaría igualmente bien si eligiésemos el semigrupo `Set`, en cuyo caso no importaría en qué orden ejecutásemos los distintos validadores. ¡Podríamos incluso ejecutarlos en paralelo sobre la estructura de datos!

Como segundo ejemplo, el paquete `purescript-parallel` proporciona un constructor de tipo `Parallel` que representa _cálculos paralelos_. `Parallel` proporciona una función `parallel` que usa un funtor aplicativo para calcular el resultado de sus cálculos de entrada _en paralelo_:

```haskell
f <$> parallel computation1
  <*> parallel computation2
```

Este cálculo comenzaría a calcular valores de manera asíncrona usando `computation 1` y `computation 2`. Cuando ambos resultados hayan sido calculados, serán combinados en un resultado único usando la función `f`.

Veremos esta idea en más detalle cuando apliquemos los funtores aplicativos al problema del _infierno de retrollamadas_ (*callback hell*) más adelante.

Los funtores aplicativos son una manera natural de capturar efectos secundarios en paralelo que pueden ser combinados.

## Conclusión

En este capítulo, hemos cubierto un montón de nuevas ideas:

- Hemos presentado el concepto de _funtor aplicativo_ que generaliza la idea de aplicación de función a constructores de tipo que capturan alguna noción de efecto secundario.
- Hemos visto cómo los funtores aplicativos dieron una solución al problema de validar estructuras de datos, y cómo cambiando el funtor aplicativo podemos cambiar de informar de un único error a informar de todos los errores en una estructura de datos.
- Hemos conocido la clase de tipos `Traversable` que encapsula la idea de _funtor transitable_, un contenedor cuyos elementos se pueden usar para combinar valores con efectos secundarios.

Los funtores aplicativos son una abstracción interesante que proporciona soluciones limpias a varios problemas. Los veremos unas cuantas veces más a lo largo del libro. En este caso, el funtor aplicativo de validación proporcionó una forma de escribir validadores en estilo declarativo, permitiéndonos definir _qué_ validan nuestros validadores y no _cómo_ deben realizar la validación. En general, veremos que los funtores aplicativos son una herramienta útil para diseñar _lenguajes específicos del dominio_ (domain specific languages).

En el siguiente capítulo veremos una idea relacionada, la clase de las _mónadas_, y extenderemos nuestro ejemplo de la agenda para funcionar en el navegador.
