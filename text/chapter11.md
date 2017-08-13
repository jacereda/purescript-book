# Aventuras monádicas

## Objetivos del capítulo

El objetivo de este capítulo será aprender sobre los _transformadores de mónadas_, que ofrecen una manera de combinar efectos secundarios proporcionados por diferentes mónadas. El ejemplo motivador será una juego de aventuras textual que se puede jugar en la consola de NodeJS. Los diferentes efectos secundarios del juego (registro de mensajes, estado y configuración) se proporcionarán por medio de una pila de transformadores de mónadas.

## Preparación del proyecto

El módulo de este proyecto añade las siguientes dependencias Bower nuevas:

- `purescript-maps`, que proporciona un tipo de datos para asociaciones inmutables
- `purescript-sets`, que proporciona un tipo de datos para conjuntos inmutables
- `purescript-transformers`, que proporciona implementaciones de transformadores de mónadas estándar
- `purescript-node-readline`, que proporciona ligaduras FFI a la interfaz [`readline`](http://nodejs.org/api/readline.html) de NodeJS
- `purescript-yargs`, que proporciona una interfaz aplicativa a la biblioteca de procesamiento de argumentos de línea de comandos [`yargs`](https://www.npmjs.org/package/yargs)

También es necesario instalar el módulo `yargs` usando NPM:

```text
npm install
```

## Cómo jugar

Para ejecutar el proyecto, usa `pulp run`.

Por defecto verás un mensaje de uso:

```text
node ./dist/Main.js -p <player name>

Options:
  -p, --player  Player name  [required]
  -d, --debug   Use debug mode

Missing required arguments: p
The player name is required
```

Proporciona el nombre del jugador usando la opción `-p`:

```text
pulp run -- -p Phil
>
```

Desde el símbolo de espera de órdenes puedes introducir comandos como `look`, `inventory`, `take`, `use`, `north`, `south`, `east`, y `west`. Hay también un comando `debug` que se puede usar para imprimir el estado del juego cuando se proporciona la opción de línea de comandos `--debug`.

El juego se desarrolla en una rejilla bidimensional, y el jugador se mueve mandando comandos `north`, `south`, `east`, and `west`. El juego contiene una colección de objetos que pueden estar en posesión del jugador (en el _inventario_ del usuario), o en la rejilla del juego en alguna posición. Los objetos pueden ser recogidos por el jugador usando el comando `take`.

Para referencia, aquí hay un recorrido completo del juego:

```text
$ pulp run -- -p Phil

> look
You are at (0, 0)
You are in a dark forest. You see a path to the north.
You can see the Matches.

> take Matches
You now have the Matches

> north
> look
You are at (0, 1)
You are in a clearing.
You can see the Candle.

> take Candle
You now have the Candle

> inventory
You have the Candle.
You have the Matches.

> use Matches
You light the candle.
Congratulations, Phil!
You win!
```

El juego es muy simple, pero el objetivo del capítulo es usar el paquete `purescript-transformers` para construir una biblioteca que permita desarrollo rápido de este tipo de juegos.

## La mónada State

Comenzaremos viendo algunas de las mónadas suministradas por el paquete `purescript-transformers`.

El primer ejemplo es la mónada `State`, que proporciona una forma de modelar _estado mutable_ en código puro. Hemos visto ya dos enfoques al estado mutable proporcionados por la mónada `Eff`, a saber, los efectos `REF` y `ST`. `State` proporciona una tercera alternativa que no está implementada usando la mónada `Eff`.

El constructor de tipo `State` toma dos parámetros de tipo: el tipo `s` del estado y un tipo de retorno `a`. Aunque hablamos de la "mónada `State`", la instancia de la clase de tipos `Monad` está de hecho proporcionada para el constructor de tipo `State s`, para cualquier tipo `s`.

El módulo `Control.Monad.State` proporciona la siguiente API:

```haskell
get    :: forall s.             State s s
put    :: forall s. s        -> State s Unit
modify :: forall s. (s -> s) -> State s Unit
```

Esto parece muy similar a la API proporcionada por los efectos `REF` y `ST`. Sin embargo, fíjate en que no pasamos una referencia a celda mutable como `Ref` o `STRef` a las acciones. La diferencia entre `State` y las soluciones provistas por la mónada `Eff` es que la mónada `State` sólo soporta un único fragmento de estado que está implícito; el estado se implementa como un argumento de función oculto en el constructor de datos de la mónada `State`, de manera que no hay que pasar una referencia explícita.

Veamos un ejemplo. Un uso de la mónada `State` puede ser sumar los valores de un array de números sobre el estado actual. Podríamos hacerlo eligiendo `Number` como el tipo de estado `s` y usar `traverse_` para recorrer el array, con una llamada a `modify` para cada elemento del array:
	
```haskell
import Data.Foldable (traverse_)
import Control.Monad.State
import Control.Monad.State.Class

sumArray :: Array Number -> State Number Unit
sumArray = traverse_ \n -> modify \sum -> sum + n
```

El módulo `Control.Monad.State` proporciona tres funciones para ejecutar un cálculo en la mónada `State`:

```haskell
evalState :: forall s a. State s a -> s -> a
execState :: forall s a. State s a -> s -> s
runState  :: forall s a. State s a -> s -> Tuple a s
```

Cada una de estas funciones toma un estado inicial de tipo `s` y un cálculo de tipo `State s a`. `evalState` sólo devuelve el valor de retorno, `execState` sólo devuelve el estado final, y `runState` devuelve ambos expresados como un valor de tipo `Tuple a s`.

Dada la función `sumArray` de arriba, podemos usar `execState` en PSCi para sumar los números de varios arrays como sigue:

```text
> :paste
… execState (do
…   sumArray [1, 2, 3]
…   sumArray [4, 5]
…   sumArray [6]) 0
… ^D
21
```

X> ## Ejercicios
X>
X> 1. (Fácil) ¿Cuál es el resultado de reemplazar `execState` con `runState` o `evalState` en nuestro ejemplo anterior?
X> 1. (Medio) Una cadena de paréntesis está _equilibrada_ si se obtiene o bien concatenando cero o más cadenas equilibradas, o envolviendo una cadena equilibrada más corta en un par de paréntesis.
X>     
X>     Usa la mónada `State` y la función `traverse_` para escribir una función
X>
X>     ```haskell
X>     testParens :: String -> Boolean
X>     ```
X>
X>     que comprueba si una `String` de paréntesis está equilibrada, llevando la cuenta del número de paréntesis abiertos que no han sido cerrados. Tu función debe funcionar como sigue:
X>
X>     ```text
X>     > testParens ""
X>     true
X>     
X>     > testParens "(()(())())"
X>     true
X>     
X>     > testParens ")"
X>     false
X>     
X>     > testParens "(()()"
X>     false
X>     ```
X>
X>     _Pista_: puedes querer usar la función `toCharArray` del módulo `Data.String` para convertir la cadena de entrada en un array de caracteres.

## La mónada Reader

Otra mónada suministrada por el paquete `purescript-transformers` es la mónada `Reader`. Esta mónada proporciona la capacidad de leer de una configuración global. Mientras que la mónada `State` proporciona la capacidad de leer y escribir un único fragmento de estado mutable, la mónada `Reader` sólo proporciona la capacidad de leer un único fragmento de datos.

El constructor de tipo `Reader` toma dos argumentos de tipo: un tipo `r` que representa el tipo de configuración y el tipo de retorno `a`.

El módulo `Control.Monad.Reader` provee la siguiente API:

```haskell
ask   :: forall r. Reader r r
local :: forall r a. (r -> r) -> Reader r a -> Reader r a
```

La acción `ask` se puede usar para leer la configuración actual, y la acción `local` se puede usar para ejecutar un cálculo con una configuración modificada.

Por ejemplo, supongamos que estamos desarrollando una aplicación controlada por permisos y queremos usar la mónada `Reader` para mantener el objeto con los permisos del usuario actual. Podemos elegir que el tipo `r` sea un tipo `Permissions` con la siguiente API:

```haskell
hasPermission :: String -> Permissions -> Boolean
addPermission :: String -> Permissions -> Permissions
```

Cuando queramos comprobar si el usuario tiene un permiso concreto, podemos usar `ask` para extraer el objeto de permisos actual. Por ejemplo, podemos querer que sólo los administradores puedan crear usuarios nuevos:

```haskell
createUser :: Reader Permissions (Maybe User)
createUser = do
  permissions <- ask
  if hasPermission "admin" permissions
    then map Just newUser
    else pure Nothing
```

Para elevar los permisos de usuario, podemos usar la acción `local` para modificar el objeto `Permissions` durante la ejecución de algún cálculo:

```haskell
runAsAdmin :: forall a. Reader Permissions a -> Reader Permissions a
runAsAdmin = local (addPermission "admin")
```

Podemos entonces escribir una función para crear un nuevo usuario incluso si el usuario no tiene los permisos `admin`:

```haskell
createUserAsAdmin :: Reader Permissions (Maybe User)
createUserAsAdmin = runAsAdmin createUser
```

Para ejecutar un cálculo en la mónada `Reader` se puede usar la función `runReader` para suministrar la configuración global:

```haskell
runReader :: forall r a. Reader r a -> r -> a
```

X> ## Ejercicios
X>
X> En estos ejercicios, usaremos la mónada `Reader` para construir una pequeña biblioteca para representar documentos con sangría. La "configuración global" será un número indicando el nivel de sangría actual:
X>
X>    ```haskell
X>    type Level = Int
X>    
X>    type Doc = Reader Level String
X>    ```
X>
X> 1. (Fácil) Escribe una función `line` que representa una línea con el nivel de sangría actual. Tu función debe tener el siguiente tipo:
X>
X>     ```haskell
X>     line :: String -> Doc
X>     ```
X>
X>     _Pista_: usa la función `ask` para leer el nivel de sangría actual.
X> 1. (Fácil) Usa la función `local` para escribir una función
X>
X>     ```haskell
X>     indent :: Doc -> Doc
X>     ```
X>
X>     que incrementa el nivel de sangría para un bloque de código.
X> 1. (Medio) Usa la función `sequence` definida en `Data.Traversable` para escribir una función
X>
X>     ```haskell
X>     cat :: Array Doc -> Doc
X>     ```
X>
X>     que concatena una colección de documentos, separándolos con nuevas líneas.
X> 1. (Medio) Usa la función `runReader` para escribir una función
X>
X>     ```haskell
X>     render :: Doc -> String
X>     ```
X>
X>     que representa un documento como una `String`.
X>
X> Debes ahora ser capaz de usar tu biblioteca para escribir documentos simples como sigue:
X>
X> ```haskell
X> render $ cat
X>   [ line "Here is some indented text:"
X>   , indent $ cat
X>       [ line "I am indented"
X>       , line "So am I"
X>       , indent $ line "I am even more indented"
X>       ]
X>   ]
X> ```

## La mónada Writer

La mónada `Writer` proporciona la capacidad de acumular un valor secundario además del valor de retorno de un cálculo.

Un caso de uso común es acumular un registro de mensajes de tipo `String` o `Array String`, pero la mónada `Writer` es más general que esto. Puede de hecho ser usada para acumular un valor en cualquier monoide, de manera que se puede usar para llevar registro de un total entero usando el monoide `Additive Int`, o registrar si alguno de varios valores `Boolean` intermedios era `true` usando el monoide `Disj Boolean`.

El constructor de tipo `Writer` toma dos argumentos de tipo. Un tipo `w` que debe ser una instancia de la clase de tipos `Monoid`, y el tipo de retorno `a`.

El elemento clave de la API `Writer` es la función `tell`:

```haskell
tell :: forall w a. Monoid w => w -> Writer w Unit
```

La acción `tell` añade el valor proporcionado al resultado acumulado actual.

Como ejemplo, añadamos un registro de mensajes a una función existente usando el monoide `Array String`. Considera nuestra implementación previa de la función _máximo común divisor_:

```haskell
gcd :: Int -> Int -> Int
gcd n 0 = n
gcd 0 m = m
gcd n m = if n > m
            then gcd (n - m) m
            else gcd n (m - n)
```

Podemos añadir registro de mensajes a esta función cambiando el tipo de retorno a `Writer (Array String) Int`:

```haskell
import Control.Monad.Writer
import Control.Monad.Writer.Class

gcdLog :: Int -> Int -> Writer (Array String) Int
```

Sólo tenemos que cambiar ligeramente nuestra función para que registre ambas entradas en cada paso:

```haskell
    gcdLog n 0 = pure n
    gcdLog 0 m = pure m
    gcdLog n m = do
      tell ["gcdLog " <> show n <> " " <> show m]
      if n > m
        then gcdLog (n - m) m
        else gcdLog n (m - n)
```

Podemos ejecutar un cálculo en la mónada `Writer` usando las funciones `execWriter` o `runWriter`:

```haskell
execWriter :: forall w a. Writer w a -> w
runWriter  :: forall w a. Writer w a -> Tuple a w
```

Como en el caso de la mónada `State`, `execWriter` sólo devuelve el registro de mensajes acumulado, mientras que `runWriter` devuelve tanto el registro de mensajes como el resultado.

Podemos probar nuestra función modificada en PSCi:

```text
> import Control.Monad.Writer
> import Control.Monad.Writer.Class

> runWriter (gcdLog 21 15)
Tuple 3 ["gcdLog 21 15","gcdLog 6 15","gcdLog 6 9","gcdLog 6 3","gcdLog 3 3"]
```

X> ## Ejercicios
X>
X> 1. (Medio) Reescribe la función `sumArray` de antes usando la mónada `Writer` y el monoide `Additive Int` del paquete `purescript-monoid`.
X> 1. (Medio) La función _Collatz_ está definida en los números naturales `n` como `n / 2` cuando `n` is par, y `3 * n + 1` cuando `n` es impar. Por ejemplo, la secuencia de Collatz iterada comenzando en 10 es como sigue:
X>
X>     ```text
X>     10, 5, 16, 8, 4, 2, 1, ...
X>     ```
X>
X>     Se conjetura que la función de Collatz iterada siempre llega a `1` después de un número finito de aplicaciones.
X>
X>     Escribe una función que usa recursividad para calcular cuántas iteraciones de la función Collatz son necesarias antes de que la secuencia llegue a `1`.
X>
X>     Modifica tu función para que use la mónada `Writer` para registrar mensajes en cada aplicación de la función Collatz.

## Transformadores de mónada (*monad transformers*)

Cada una de las tres mónadas anteriores, `State`, `Reader` y `Writer`, son también ejemplos de los llamados _transformadores de mónada_. Los transformadores de mónada equivalentes se llaman `StateT`, `ReaderT`, y `WriterT` respectivamente.

¿Qué es un transformador de mónada? Como hemos visto, una mónada aumenta el código PureScript con algún tipo de efecto secundario, que se puede interpretar en PureScript usando el gestor apropiado (`runState`, `runReader`, `runWriter`, etc.) Esto está bien si sólo necesitamos usar _un_ efecto secundario. Sin embargo, a menudo es útil usar más de un efecto secundario a la vez. Por ejemplo, podemos querer usar `Reader` junto a `Maybe` para expresar un _resultado opcional_ en el contexto de alguna configuración global. O podemos querer el estado mutable proporcionado por la mónada `State` junto a la capacidad pura de registrar errores de la mónada `Either`. Este es el problema resuelto por los _trasformadores de mónada_.

Fíjate en que ya hemos visto que lo mónada `Eff` proporciona una solución parcial a este problema, ya que les efectos nativos se pueden intercalar usando la estrategia de los _efectos extensibles_. Los transformadores de mónada proporcionan otra solución y cada aproximación tiene sus beneficios y limitaciones.

Un transformador de mónada es un constructor de tipo que está parametrizado no sólo por un tipo, sino también por un constructor de otro tipo. Toma una mónada y la convierte en otra mónada añadiendo su propia variedad de efectos secundarios.

Veamos un ejemplo. La version transformadora de mónada de la mónada `State` es `StateT`, definida en el módulo `Control.Monad.State.Trans`. Podemos averiguar la familia de `StateT` usando PSCi:

```text
> import Control.Monad.State.Trans
> :kind StateT
Type -> (Type -> Type) -> Type -> Type
```

Esto parece bastante confuso, pero podemos aplicar a `StateT` un argumento cada vez para entender cómo usarlo.

El primer argumento de tipo es el tipo de estado que queremos usar, como en el caso de `State`. Intentemos usar un estado de tipo `String`:

```text
> :kind StateT String
(Type -> Type) -> Type -> Type
```

El siguiente argumento es un constructor de tipo con familia `Type -> Type`. Representa la mónada subyacente a la que queremos añadir los efectos de `StateT`. Como ejemplo elijamos la mónada `Either String`:

```text
> :kind StateT String (Either String)
Type -> Type
```

Nos queda un constructor de tipo. El argumento final representa el tipo de retorno, y podemos instanciarlo a `Number` por ejemplo:

```text
> :kind StateT String (Either String) Number
Type
```

Finalmente nos queda algo con familia `Type`, que significa que ya podemos intentar buscar valores de este tipo.

La mónada que hemos construido (`StateT String (Either String)`) representa cálculos que pueden fallar con un error y que pueden usar estado mutable.

Podemos usar las acciones de la mónada externa `StateT String` (`get`, `put`, y `modify`) directamente, pero para usar los efectos de la mónada envuelta (`Either String`) necesitamos "elevarlas" sobre el transformador de mónada. El módulo `Control.Monad.Trans` define la clase de tipos `MonadTrans` que captura los constructores de tipo que son transformadores de mónada como sigue:

```haskell
class MonadTrans t where
  lift :: forall m a. Monad m => m a -> t m a
```

Esta clase contiene un único miembro, `lift`, que toma cálculos en cualquier mónada subyacente `m` y los eleva a la mónada envuelta `t m`. En nuestro caso, el constructor de tipo `t` es `StateT String`, y `m` es la mónada `Either String`, de manera que `lift` proporciona una manera de elevar cálculos de tipo `Either String a` a cálculos de tipo `StateT String (Either String) a`. Esto significa que podemos usar los efectos de `StateT String` y `Either String` uno al lado del otro, siempre y cuando usemos `lift` cada vez que usamos un cálculo de tipo `Either String a`.

Por ejemplo, el siguiente cálculo lee el estado subyacente y devuelve un error si el estado es la cadena vacía:

```haskell
import Data.String (drop, take)

split :: StateT String (Either String) String
split = do
  s <- get
  case s of
    "" -> lift $ Left "Empty string"
    _ -> do
      put (drop 1 s)
      pure (take 1 s)
```

Si el estado no está vacío, el cálculo usa `put` para actualizar el estado a `drop 1 s` (esto es, `s` quitando el primer carácter), y devuelve `take 1 s` (esto es, el primer carácter de `s`).

Probémoslo en PSCi:

```text
> runStateT split "test"
Right (Tuple "t" "est")

> runStateT split ""
Left "Empty string"
```

Esto no es muy destacable, ya que podríamos haberlo implementado sin `StateT`. Sin embargo, como estamos trabajando en una mónada, podemos usar notación do o combinadores aplicativos para construir cálculos más grandes a partir de cálculos más pequeños. Por ejemplo, podemos aplicar `split` dos veces para leer los dos primeros caracteres de una cadena:

```text
> runStateT ((<>) <$> split <*> split) "test"
(Right (Tuple "te" "st"))
```

Podemos usar la función `split` junto a otras cuantas acciones para construir una biblioteca de análisis básica. De hecho, este es el método usado por la biblioteca `purescript-parsing`. Esta es la potencia de los transformadores de mónadas; podemos crear mónadas a medida para una variedad de problemas, elegiendo los efectos secundarios que necesitamos y manteniendo la expresividad de la notación do y los combinadores aplicativos.

## El transformador de mónada ExceptT

El paquete `purescript-transformers` define también el transformador de mónada `ExceptT e`, que es el transformador correspondiente a la mónada `Either e`. Proporciona la siguiente API:

```haskell
class MonadError e m where
  throwError :: forall a. e -> m a
  catchError :: forall a. m a -> (e -> m a) -> m a

instance monadErrorExceptT :: Monad m => MonadError e (ExceptT e m)

runExceptT :: forall e m a. ExceptT e m a -> m (Either e a)
```

La clase `MonadError` captura aquellas mónadas que soportan lanzar y cazar errores de algún tipo `e`, y proporciona una instancia para el transformador de mónada `ExceptT e`. La acción `throwError` se puede usar para indicar fallo, igual que `Left` en la mónada `Either e`. La acción `catchError` nos permite continuar tras lanzar un error usando `throwError`.

El gestor `runExceptT` se usa para ejecutar cálculos de tipo `ExceptT e m a`.

Esta API es similar a la proporcionada por el paquete `purescript-exceptions` y el efecto `Exception`. Sin embargo hay diferencias importantes:

- `Exception` usa excepciones JavaScript mientras que `ExceptT` modela los errores como una estructura de datos pura
- El efecto `Exception` sólo soporta excepciones de un tipo, el tipo `Error` de JavaScript, mientras que `ExceptT` soporta errores de cualquier tipo. En particular, somos libres de definir nuevos tipos de error.

Probemos `ExceptT` usándolo para envolver la mónada `Writer`. De nuevo, somos libres de usar acciones del transformador de mónada `ExceptT e` directamente, pero los cálculos en la mónada `Writer` deben elevarse usando `lift`:

```haskell
import Control.Monad.Trans
import Control.Monad.Writer
import Control.Monad.Writer.Class
import Control.Monad.Error.Class
import Control.Monad.Except.Trans

writerAndExceptT :: ExceptT String (Writer (Array String)) String
writerAndExceptT = do
  lift $ tell ["Before the error"]
  throwError "Error!"
  lift $ tell ["After the error"]
  pure "Return value"
```

Si probamos esta función en PSCi, podemos ver cómo los dos efectos de acumular un registro de mensajes y lanzar errores interactúan. Primero, podemos ejecutar el cálculo externo `ExceptT` usando `runExceptT`, devolviendo un resultado de tipo `Writer String (Either String String)`. Podemos entonces usar `runWriter` para ejecutar el cálculo interno `Writer`:

```text
> runWriter $ runExceptT writerAndExceptT
Tuple (Left "Error!") ["Before the error"]
```

Fíjate en que sólo los mensajes escritos antes de que se lanzase el error han sido añadidos al registro.

## Pilas de transformadores de mónada (*monad transformer stacks*)

Como hemos visto, los transformadores de mónada se pueden usar para construir nuevas mónadas sobre mónadas existentes. Para algún transformador de mónada `t1` y alguna mónada `m`, la aplicación `t1 m` es también una mónada. Esto significa que podemos aplicar un _segundo_ transformador de mónada `t2` al resultado `t1 m` para construir una tercera mónada `t2 (t1 m)`. De esta forma, podemos construir una _pila_ de transformadores de mónada que pueden combinar los efectos secundarios proporcionados por sus mónadas constituyentes.

En la práctica, la mónada subyacente `m` es o bien la mónada `Eff` si se requieren efectos secundarios nativos, o la mónada `Identity` definida en el módulo `Data.Identity`. La mónada `Identity` no añade efectos secundarios nuevos, de manera que transformar la mónada `Identity` sólo proporciona los efectos del transformador de mónada. De hecho, las mónadas `State`, `Reader` y `Writer` se implementan transformando la mónada `Identity` con `StateT`, `ReaderT` y `WriterT` respectivamente.

Veamos un ejemplo en el que combinamos tres efectos secundarios. Usaremos los efectos `StateT`, `WriterT` y `ExceptT`, con la mónada `Identity` en la base de la pila. Esta pila de transformadores de mónada proporcionará los efectos secundarios de estado mutable, llevar un registro de mensajes y errores puros.

Podemos usar esta pila de transformadores de mónada para reproducir nuestra acción `split` con la capacidad añadida de registrar mensajes.

```haskell
type Errors = Array String

type Log = Array String

type Parser = StateT String (WriterT Log (ExceptT Errors Identity))

split :: Parser String
split = do
  s <- get
  lift $ tell ["The state is " <> show s]
  case s of
    "" -> lift $ lift $ throwError ["Empty string"]
    _ -> do
      put (drop 1 s)
      pure (take 1 s)
```

Si probamos este cálculo en PSCi vemos que el estado se añade al registro de mensajes para cada invocación de `split`.

Date cuenta de que tenemos que eliminar los efectos secundarios en el orden en que aparecen en la pila de transformadores de mónada: primero usamos `runStateT` para quitar el constructor de tipo `StateT`, luego `runWriterT`, y a continuación `runExceptT`. Finalmente, ejecutamos el cálculo en la mónada `Identity` usando `runIdentity`.

```text
> runParser p s = runIdentity $ runExceptT $ runWriterT $ runStateT p s

> runParser split "test"
(Right (Tuple (Tuple "t" "est") ["The state is test"]))

> runParser ((<>) <$> split <*> split) "test"
(Right (Tuple (Tuple "te" "st") ["The state is test", "The state is est"]))
```

Sin embargo, si el análisis no tiene éxito porque el estado está vacío no se registra ningún mensaje:

```text
> runParser split ""
(Left ["Empty string"])
```

Esto se debe a la forma en que los efectos secundarios suministrados por el transformador de mónada `ExceptT` interactúa con los efectos secundarios proporcionados por el transformador de mónada `WriterT`. Podemos abordar esto cambiando el orden en que componemos la pila de transformadores de mónada. Si movemos el transformador `ExceptT` al tope de la pila, el registro de mensajes contendrá todos los mensajes escritos hasta el primer error, como vimos previamente cuando transformamos `Writer` con `ExceptT`.

Un problema con este código es que tenemos que usar la función `lift` múltiples veces para elevar cálculos sobre múltiples transformadores de mónada: por ejemplo, la llamada a `throwError` tiene que elevarse dos veces, una vez sobre `WriterT` y una segunda vez sobre `StateT`. Esto está bien para pequeñas pilas de transformadores de mónada, pero se vuelve molesto rápidamente.

Afortunadamente, como veremos, podemos usar la generación automática de código proporcionada por la inferencia de clases de tipo para hacer la mayor parte de este "levantamiento pesado" por nosotros.

X> ## Ejercicios
X>
X> 1. (Fácil) Usa el transformador de mónada `ExceptT` sobre el funtor `Identity` para escribir una función `safeDivide` que divide dos números, lanzando un error si el denominador es cero.
X> 1. (Medio) Escribe un analizador
X>
X>     ```haskell
X>     string :: String -> Parser String
X>     ```
X>
X>     que se ajuste a una cadena como prefijo del estado actual o falle con un mensaje de error.
X>
X>     Tu analizador debe funcionar como sigue:
X>
X>     ```text
X>     > runParser (string "abc") "abcdef"
X>     (Right (Tuple (Tuple "abc" "def") ["The state is abcdef"]))
X>     ```
X>
X>     _Pista_: puedes usar la implementación de `split` como punto de partida. Puede que encuentres útil la función `stripPrefix`.
X> 1. (Difícil) Usa los transformadores de mónada `ReaderT` y `WriterT` para reimplementar la biblioteca de impresión de documentos que escribimos previamente usando la mónada `Reader`.
X>
X>     En lugar de usar `line` para emitir cadenas y `cat` para concatenar cadenas, usa el monoide `Array String` con el transformador de mónada `WriterT`, y usa `tell` para añadir una línea al resultado.

## ¡Clases de tipos al rescate!

Cuando vimos la mónada `State` al comienzo del capítulo, di los siguientes tipos a las acciones de la mónada `State`:

```haskell
get    :: forall s.             State s s
put    :: forall s. s        -> State s Unit
modify :: forall s. (s -> s) -> State s Unit
```

En realidad, los tipos dados en el módulo `Control.Monad.State.Class` son más generales que eso:

```haskell
get    :: forall m s. MonadState s m =>             m s
put    :: forall m s. MonadState s m => s        -> m Unit
modify :: forall m s. MonadState s m => (s -> s) -> m Unit
```

El módulo `Control.Monad.State.Class` define la clase de tipos (multiparámetro) `MonadState`, que nos permite abstraer sobre "mónadas que soportan estado puro mutable". Como cabe esperar, el constructor de tipo `State s` es una instancia de la clase de tipos `MonadState s`, pero hay más instancias interesantes de esta clase.

En particular, hay instancias de `MonadState` para los transformadores de mónada `WriterT`, `ReaderT` y `ExceptT` proporcionados en el paquete `purescript-transformers`. Cada uno de estos transformadores de mónada tiene una instancia de `MonadState` cuando la `Monad` subyacente la tenga. En la práctica, esto significa que siempre y cuando `StateT` aparezca _en algún sitio_ de la pila de transformadores de mónada, y todo lo que haya por encima de `StateT` sea una instancia de `MonadState`, somos libres de usar `get`, `put` y `modify` directamente sin usar `lift`.

De hecho, lo mismo es cierto para las acciones que hemos visto para los transformadores `ReaderT`, `WriterT` y `ExceptT`. `purescript-transformers` define una clase de tipos para cada uno de los transformadores principales, permitiéndonos abstraer sobre mónadas que soportan sus operaciones.

En el caso de la función `split` de arriba, la pila de mónadas que construimos es una instancia de cada una de las clases de tipos `MonadState`, `MonadWriter` y `MonadError`. ¡Esto significa que no necesitamos llamar a `lift` para nada! Podemos simplemente usar las acciones `get`, `put`, `tell` y `throwError` como si estuviesen definidas en la misma pila de mónadas:

```haskell
split :: Parser String
split = do
  s <- get
  tell ["The state is " <> show s]
  case s of
    "" -> throwError "Empty string"
    _ -> do
      put (drop 1 s)
      pure (take 1 s)
```

Parece como si realmente hubiésemos extendido nuestro lenguaje de programación para soportar los tres efectos secundarios de estado mutable, registro de mensajes y gestión de errores. Sin embargo, todo está implementado usando funciones puras y estado inmutable bajo el capó.

## Alternativas

El paquete `purescript-control` define un número de abstracciones para trabajar con cálculos que pueden fallar. Una de estas es la clase de tipos `Alternative`:

```haskell
class Functor f <= Alt f where
  alt :: forall a. f a -> f a -> f a

class Alt f <= Plus f where
  empty :: forall a. f a

class (Applicative f, Plus f) <= Alternative f
```

`Alternative` proporciona dos nuevos combinadores: el valor `empty` que proporciona un prototipo para un cálculo fallido, y la función `alt` (y su sinónimo `<|>`) que proporciona la capacidad de recurrir a un cálculo _alternativo_ en caso de error.

El módulo `Data.List` proporciona dos funciones útiles para trabajar con constructores de tipo de la clase de tipos `Alternative`:

```haskell
many :: forall f a. Alternative f => Lazy (f (List a)) => f a -> f (List a)
some :: forall f a. Alternative f => Lazy (f (List a)) => f a -> f (List a)
```

El combinador `many` usa la clase de tipos `Alternative` para ejecutar repetidamente un cálculo _cero o más_ veces. El combinador `some` es similar, pero requiere que al menos el primer cálculo tenga éxito.

En el caso de nuestra pila de transformadores de mónada `Parser`, hay una instancia de `Alternative` inducida por la componente `ExceptT`, que soporta fallo mediante composición de errores en diferentes ramas usando una instancia `Monoid` (esto es por lo que elegimos `Array String` para nuestro tipo `Errors`). Esto significa que podemos usar las funciones `many` y `some` para ejecutar un analizador múltiples veces:

```text
> import Split
> import Control.Alternative

> runParser (many split) "test"
(Right (Tuple (Tuple ["t", "e", "s", "t"] "")
              [ "The state is \"test\""
              , "The state is \"est\""
              , "The state is \"st\""
              , "The state is \"t\""
              ]))
```

Aquí, la cadena de entrada `"test"` se ha partido repetidamente para devolver un array de cuatro cadenas con un único carácter, el estado resultante está vacío, y el registro muestra que hemos aplicado el combinador `split` cuatro veces.

Otros ejemplos de constructores de tipo `Alternative` son `Maybe` y `Array`.

## Mónadas por comprensión

El módulo `Control.MonadPlus` define una subclase de la clase de tipos `Alternative` llamada `MonadPlus`. `MonadPlus` captura los constructores de tipo que son mónadas e instancias de `Alternative`:

```haskell
class (Monad m, Alternative m) <= MonadZero m

class MonadZero m <= MonadPlus m
```

En particular, nuestra mónada `Parser` es instancia de `MonadPlus`.

Cuando vimos los arrays por comprensión presentamos la función `guard`, que se puede usar para eliminar resultados no deseados. De hecho, la función `guard` es más general y se puede usar para cualquier mónada que sea una instancia de `MonadPlus`:

```haskell
guard :: forall m. MonadZero m => Boolean -> m Unit
```

El operador `<|>` nos permite volver hacia atrás en caso de fallo. Para ver cómo es útil esto, definamos una variante del combinador `split` que sólo detecta mayúsculas:

```haskell
upper :: Parser String
upper = do
  s <- split
  guard $ toUpper s == s
  pure s
```

Aquí hemos usado `guard` para fallar si la cadena no está en mayúsculas. Fíjate en que este código es muy similar a los arrays por comprensión que vimos antes. Cuando usamos `MonadPlus` de esta forma, decimos que estamos construyendo _mónadas por comprensión_.

## Retroceso (*backtracking*)

Podemos usar el operador `<|>` para retroceder a otra alternativa en caso de fallo. Para demostrarlo, definamos otro analizador que busca caracteres minúsculos:

```haskell
lower :: Parser String
lower = do
  s <- split
  guard $ toLower s == s
  pure s
```

Con esto, podemos definir un analizador que ajusta ávidamente muchas mayúsculas si el primer carácter es mayúscula, o muchas minúsculas si el primer carácter es minúscula:

```text
> upperOrLower = some upper <|> some lower
```

Este analizador encontrará coincidencias hasta que cambiemos de mayúsculas a minúsculas o viceversa:

```text
> runParser upperOrLower "abcDEF"
(Right (Tuple (Tuple ["a","b","c"] ("DEF"))
              [ "The state is \"abcDEF\""
              , "The state is \"bcDEF\""
              , "The state is \"cDEF\""
              ]))
```

Podemos incluso usar `many` para partir una cadena por completo en sus componentes mayúsculas y minúsculas:

```text
> components = many upperOrLower

> runParser components "abCDeFgh"
(Right (Tuple (Tuple [["a","b"],["C","D"],["e"],["F"],["g","h"]] "")
              [ "The state is \"abCDeFgh\""
              , "The state is \"bCDeFgh\""
              , "The state is \"CDeFgh\""
              , "The state is \"DeFgh\""
              , "The state is \"eFgh\""
              , "The state is \"Fgh\""
              , "The state is \"gh\""
              , "The state is \"h\""
              ]))
```

De nuevo, esto ilustra la potencia de reutilización que proporcionan los transformadores de mónada. ¡Hemos sido capaces de escribir un analizador con retroceso en estilo declarativo con sólo unas pocas líneas de código reutilizando abstracciones estándar!

X> ## Ejercicios
X>
X> 1. (Fácil) Elimina las llamadas a `lift` de tu implementación del analizador `string`. Verifica que la nueva implementación pasa la comprobación de tipos y convéncete de que debe.
X> 1. (Medio) Usa tu analizador `string` con el combinador `many` para escribir un analizador que reconozca cadenas consistentes en varias copias de la cadena `"a"` seguidas de varias copias de la cadena `"b"`.
X> 1. (Medio) Usa el operador `<|>` para escribir un analizador que reconozca cadenas con las letras `a` o `b` en cualquier orden.
X> 1. (Difícil) La mónada `Parser` se puede definir también como sigue:
X>
X>     ```haskell
X>     type Parser = ExceptT Errors (StateT String (WriterT Log Identity))
X>     ```
X>
X>     ¿Qué efecto tiene este cambio en nuestras funciones analizadoras?

## La mónada RWS

Hay una combinación concreta de transformadores de mónada tan común que se proporciona como un transformador único en el paquete `purescript-transformers`. Las mónadas `Reader`, `Writer` y `State` se combinan para formar la mónada _reader-writer-state_, o simplemente la mónada `RWS`. Esta mónada tiene un transformador de mónada correspondiente llamado transformador de mónada `RWST`.

Usaremos la mónada `RWS` para modelar la lógica del juego de nuestro juego de aventuras textual.

La mónada `RWS` está definida por tres parámetros de tipo (además de su valor de retorno):

```haskell
type RWS r w s = RWST r w s Identity
```

Fíjate en que la mónada `RWS` está definida en términos de su propio transformador de mónada, fijando la mónada base a `Identity` que no tiene efectos secundarios.

El primer parámetro de tipo, `r`, representa el tipo de la configuración global. El segundo, `w`, representa el monoide que usaremos para acumular un registro de mensajes, y el tercero, `s`, es el tipo de nuestro estado mutable.

En el caso de nuestro juego, nuestra configuración global está definida en un tipo llamada `GameEnvironment` en el módulo `Data.GameEnvironment`:

```haskell
type PlayerName = String

newtype GameEnvironment = GameEnvironment
  { playerName    :: PlayerName
  , debugMode     :: Boolean
  }
```

Define el nombre del jugador y un indicador de si estamos ejecutando el juego en modo de depuración. Estas opciones se fijarán desde la línea de comandos cuando vayamos a ejecutar nuestro transformador de mónada.

El estado mutable está definido en un tipo llamado `GameState` del módulo `Data.GameState`:

```haskell
import qualified Data.Map as M
import qualified Data.Set as S

newtype GameState = GameState
  { items       :: M.Map Coords (S.Set GameItem)
  , player      :: Coords
  , inventory   :: S.Set GameItem
  }
```

El tipo de datos `Coords` representa puntos en una rejilla bidimensional, y el tipo de datos `GameItem` es una enumeración de los objetos del juego:

```haskell
data GameItem = Candle | Matches
```

El tipo `GameState` usa dos nuevas estructuras de datos: `Map` y `Set`, que representan asociaciones ordenadas y conjuntos ordenados respectivamente. La propiedad `items` es un mapeo de coordenadas en la rejilla del juego a conjuntos de objetos del juego en esa ubicación. La propiedad `player` almacena la ubicación actual del jugador, y la propiedad `inventory` almacena el conjunto de objetos del juego acarreados por el jugador.

Las estructuras de datos `Map` y `Set` están ordenadas por sus claves, que pueden ser cualquier tipo en la clase de tipos `Ord`. Esto significa que las claves de nuestras estructuras de datos deben estar completamente ordenadas.

Veremos cómo las estructuras `Map` y `Set` se usan para escribir las acciones de nuestro juego.

Para nuestro registro de mensajes usaremos el monoide `List String`. Podemos definir un sinónimo de tipo para nuestra mónada `Game` implementada mediante `RWS`:

```haskell
type Log = L.List String

type Game = RWS GameEnvironment Log GameState
```

## Implementando la lógica del juego

Vamos a construir nuestro juego a partir de acciones simples definidas en la mónada `Game`, reutilizando acciones de las mónadas `Reader`, `Writer` y `State`. En el nivel superior de nuestra aplicación, ejecutaremos los cálculos puros en la mónada `Game` y usaremos la mónada `Eff` para convertir los resultados en efectos secundarios observables como imprimir texto en la consola.

Una de las acciones más simples de nuestro juego es la acción `has`. Esta acción comprueba si el inventario del jugador contiene un objeto del juego concreto. Se define como sigue:

```haskell
has :: GameItem -> Game Boolean
has item = do
  GameState state <- get
  pure $ item `S.member` state.inventory
```

La función usa la acción `get` de la clase de tipos `MonadState` para leer el estado actual del juego y luego usa la función `member` definida en `Data.Set` para comprobar si el `GameItem` especificado aparece en el `Set` de objetos del inventario.

Otra acción es `pickUp`. Añade un objeto del juego al inventario del jugador si está en la habitación actual. Usa acciones de las clases de tipos `MonadWriter` y `MonadState`. Primero lee el estado actual del juego:

```haskell
pickUp :: GameItem -> Game Unit
pickUp item = do
  GameState state <- get
```

A continuación, `pickUp` busca el conjunto de objetos en la habitación actual. Lo hace usando la función `lookup` definida en `Data.Map`:

```haskell
  case state.player `M.lookup` state.items of
```

La función `lookup` devuelve un resultado opcional indicado por el constructor de tipo `Maybe`. Si la clave no aparece en el mapa, la función `lookup` devuelve `Nothing`, en caso contrario devuelve el valor correspondiente envuelto en el constructor `Just`.

Estamos interesados en el caso en que el conjunto de objetos correspondiente contiene el objeto del juego especificado. De nuevo, podemos comprobar esto usando la función `member`:

```haskell
    Just items | item `S.member` items -> do
```

En este caso, podemos usar `put` para actualizar el estado del juego y `tell` para añadir un mensaje al registro:

```haskell
      let newItems = M.update (Just <<< S.delete item) state.player state.items
          newInventory = S.insert item state.inventory
      put $ GameState state { items     = newItems
                            , inventory = newInventory
                            }
      tell (L.singleton ("You now have the " <> show item))
```

Fíjate en que no necesitamos usar `lift` en ninguno de los dos cálculos aquí porque hay instancias apropiadas de `MonadState` y `MonadWriter` en nuestra pila de transformadores de mónada `Game`.

El argumento a `put` usa una actualización de registro para modificar los campos `items` e `inventory` del estado del juego. Usamos la función `update` de `Data.Map` que modifica un valor en una clave particular. En este caso, modificamos el conjunto de objetos en la ubicación actual del jugador, usando la función `delete` para eliminar el elemento especificado del conjunto. Actualizamos también `inventory` usando `insert` para añadir el nuevo objeto al conjunto de inventario del jugador.

Finalmente, la función `pickUp` gestiona el resto de casos notificando al usuario mediante `tell`:

```haskell
    _ -> tell (L.singleton "I don't see that item here.")
```

Como ejemplo de uso de la mónada `Reader`, podemos mirar el código del comando `debug`. Este comando permite al usuario inspeccionar el estado del juego en tiempo de ejecución si el juego está ejecutándose en modo de depuración:

```haskell
  GameEnvironment env <- ask
  if env.debugMode
    then do
      state <- get
      tell (L.singleton (show state))
    else tell (L.singleton "Not running in debug mode.")
```

Aquí usamos la acción `ask` para leer la configuración del juego. De nuevo, fíjate en que no hemos necesitado usar `lift` en ningún cálculo, y que podemos usar acciones definidas en las clases de tipos `MonadState`, `MonadReader` y `MonadWriter` en el mismo bloque en notación do.

Si el indicador `debugMode` está activo se usa la acción `tell` para escribir el estado en el registro de mensajes. En otro caso, añadimos un mensaje de error.

El resto del módulo `Game` define un conjunto similar de acciones, usando cada una únicamente las acciones definidas por las clases de tipos `MonadState`, `MonadReader` y `MonadWriter`.

## Ejecutando el cálculo

Como nuestra lógica de juego se ejecuta en la mónada `RWS`, es necesario ejecutar el cálculo para responder a los comandos del usuario.

La interfaz de usuario de nuestro juego se construye usando dos paquetes: `purescript-yargs`, que proporciona una interfaz aplicativa a la biblioteca de análisis de línea de comandos `yargs`, y `purescript-node-readline`, que envuelve el módulo NodeJS `readline`, permitiéndonos escribir aplicaciones interactivas basadas en consola.

La interfaz para la lógica de juego está proporcionada por la función `game` del módulo `Game`:

```haskell
game :: Array String -> Game Unit
```

Para ejecutar este cálculo, pasamos una lista de palabras introducidas por el usuario como un array de cadenas, y ejecutamos el cálculo `RWS` resultante usando `runRWS`:

```haskell
data RWSResult state result writer = RWSResult state result writer

runRWS :: forall r w s a. RWS r w s a -> r -> s -> RWSResult s a w
```

`runRWS` parece una combinación de `runReader`, `runWriter` y `runState`. Toma una configuración global y un estado inicial como argumentos y devuelve una estructura de datos que contiene el registro de mensajes, el resultado y el estado final.

La interfaz de usuario de nuestra aplicación está definida por una función `runGame` con la siguiente firma de tipo:

```haskell
runGame
  :: forall eff
   . GameEnvironment
  -> Eff ( exception :: EXCEPTION
         , readline :: RL.READLINE
         , console :: CONSOLE
         | eff
         ) Unit
```

El efecto `CONSOLE` indica que esta función interactúa con el usuario mediante la consola (usando los paquetes `purescript-node-readline` y `purescript-console`). `runGame` toma la configuración del juego como argumento a la función.

El paquete `purescript-node-readline` proporciona el tipo `LineHandler`, que representa acciones en la mónada `Eff` que gestionan entrada de usuario de la terminal. Aquí esta la API correspondiente:

```haskell
type LineHandler eff a = String -> Eff eff a

setLineHandler
  :: forall eff a
   . Interface
  -> LineHandler (readline :: READLINE | eff) a
  -> Eff (readline :: READLINE | eff) Unit
```

El tipo `Interface` representa un descriptor de la consola, y se pasa como argumento a la función que interactúa con ella. Se puede crear un `Interface` usando la función`createConsoleInterface`:

```haskell
runGame env = do
  interface <- createConsoleInterface noCompletion
```

El primer paso es establecer el símbolo de espera de comandos de la consola. Pasamos el descriptor `interface` y proporcionamos la cadena de espera y el nivel de sangría:

```haskell
  setPrompt "> " 2 interface
```

En nuestro caso, estamos interesados en implementar la función gestora de línea. Nuestro gestor de línea se define usando una función auxiliar en una declaración `let` como sigue:

```haskell
lineHandler
  :: GameState
  -> String
  -> Eff ( exception :: EXCEPTION
         , console :: CONSOLE
         , readline :: RL.READLINE
         | eff
         ) Unit
lineHandler currentState input = do
  case runRWS (game (split " " input)) env currentState of
    RWSResult state _ written -> do
      for_ written log
      setLineHandler interface $ lineHandler state
  prompt interface
  pure unit
```

La ligadura let está cerrada tanto sobre la configuración del juego, llamada `env`, como sobre el descriptor de consola, llamado `interface`.

Nuestro gestor toma un primer argumento adicional, el estado del juego. Esto es necesario porque necesitamos pasar el estado del juego a `runRWS` para ejecutar la lógica del juego.

Lo primero que hace esta acción es partir la entrada del usuario en palabras usando la función `split` del módulo `Data.String`. Usa entonces `runRWS` para ejecutar la acción del juego `game` (en la mónada `RWS`), pasando el entorno y estado actual del juego.

Habiendo ejecutado la lógica del juego, que es un cálculo puro, necesitamos imprimir cualquier mensaje registrado a la pantalla y mostrar al usuario la cadena de espera para el siguiente comando. La acción `for_` se usa para recorrer el registro de mensajes (de tipo `List String`) e imprimir sus entradas por consola. Finalmente, `setLineHandler` se usa para actualizar la función gestora de línea para que use el estado actualizado del juego, y se muestra la cadena de espera de nuevo usando la acción `prompt`.

La función `runGame` finalmente engancha el gestor de línea inicial a la interfaz de consola y muestra la cadena de espera inicial:

```haskell
  setLineHandler interface $ lineHandler initialGameState
  prompt interface
```

X> ## Ejercicios
X>
X> 1. (Medio) Implementa un nuevo comando `cheat` que mueve todos los objetos de la rejilla de juego al inventario del usuario.
X> 1. (Difícil) La componente `Writer` de la mónada `RWS` se usa actualmente para dos tipos de mensajes: mensajes de error y mensajes informativos. Por esto, varias partes del código usan sentencias case para gestionar casos de error.
X>
X>     Refactoriza el código para usar el transformador de mónada `ExceptT` para gestionar mensajes de error, y `RWS` para gestionar mensajes informativos.

## Gestionando opciones de línea de comandos

La última parte de la aplicación es responsable de analizar las opciones de línea de comandos y crear el registro de configuración `GameEnvironment`. Para esto usamos el paquete `purescript-yargs`.

`purescript-yargs` es un ejemplo de _análisis de línea de comandos aplicativo_. Recuerda que un funtor aplicativo nos permite elevar funciones de aridad arbitraria sobre un constructor de tipo que representa algunos efectos secundarios. En el caso del paquete `purescript-yargs`, el funtor en que estamos interesados es el funtor `Y`, que añade los efectos secundarios de leer de las opciones de línea de comandos. Proporciona el siguiente gestor:

```haskell
runY :: forall a eff.
          YargsSetup ->
          Y (Eff (exception :: EXCEPTION, console :: CONSOLE | eff) a) ->
             Eff (exception :: EXCEPTION, console :: CONSOLE | eff) a
```

Esto se ilustra mejor con un ejemplo. La función `main` de la aplicación se define usando `runY` como sigue:

```haskell
main = runY (usage "$0 -p <player name>") $ map runGame env
```

El primer argumento se usa para configurar la biblioteca `yargs`. En nuestro caso simplemente proporcionamos un mensaje de uso, pero el módulo `Node.Yargs.Setup` tiene varias opciones más.

El segundo ejemplo usa la función `map` para elevar la función `runGame` sobre el constructor de tipo `Y`. El argumento `env` se construye en una declaración `where` usando los operadores aplicativos `<$>` y `<*>`:

```haskell
  where
  env :: Y GameEnvironment
  env = gameEnvironment
          <$> yarg "p" ["player"]
                   (Just "Player name")
                   (Right "The player name is required")
                   false
          <*> flag "d" ["debug"]
                   (Just "Use debug mode")
```

Aquí, la función `gameEnvironment`, que tiene el tipo `PlayerName -> Boolean -> GameEnvironment`, se eleva sobre `Y`. Los dos argumentos especifican cómo leer el nombre de usuario y el indicador de depuración de la línea de comandos. El primer argumento describe la opción del nombre de jugador, que se especifica mediante las opciones `p` o `--player`, y el segundo describe el indicador de depuración, que se activa usando las opciones `-d` o `--debug`.

Esto demuestra dos funciones básicas definidas en el módulo `Node.Yargs.Applicative`: `yarg`, que define una opción de línea de comandos que toma un argumento opcional (de tipo `String`, `Number` o `Boolean`) y `flag`, que define un indicador de línea de comando de tipo `Boolean`.

Fíjate en que hemos podido usar la notación otorgada por los operadores aplicativos para dar una especificación compacta y declarativa de nuestra interfaz de línea de comandos. Además, es simple añadir nuevos argumentos de línea de comando añadiendo un nuevo argumento de función a `runGame` y usando `<*>` para elevar `runGame` sobre un argumento adicional en la definición de `env`.

X> ## Ejercicios
X>
X> 1. (Medio) Añade una nueva propiedad `cheatMode` con valor Boolean al registro `GameEnvironment`. Añade una nueva posibilidad `-c` a la configuración de `yargs` que habilite el modo trampa. El comando `cheat` del ejercicio anterior debe estar deshabilitado si el modo trampa no está habilitado.

## Conclusión

Este capítulo ha sido una demostración práctica de las técnicas que hemos aprendido hasta ahora, usando transformadores de mónadas para construir una especificación pura de nuestro juego, y la mónada `Eff` para construir una interfaz de usuario usando la consola.

Como hemos separado nuestra implementación de la interfaz de usuario, debería ser posible crear otras interfaces para nuestro juego. Por ejemplo, podríamos usar la mónada `Eff` para representar nuestro juego en el navegador usando la API Canvas o el DOM.

Hemos visto cómo los transformadores de mónadas nos permiten escribir código seguro en un estilo imperativo, donde los efectos están registrados en el sistema de tipos. Además, las clases de tipos proporcionan una manera potente de abstraer sobre las acciones proporcionadas por una mónada, permitiendo la reutilización de código. Hemos podido usar abstracciones estándar como `Alternative` y `MonadPlus` para construir mónadas útiles combinando transformadores de mónadas estándar.

Los transformadores de mónadas son una demostración excelente de la clase de código expresivo que se puede escribir basándose en capacidades avanzadas del sistema de tipos como polimorfismo de orden mayor y clases de tipos multiparamétricas.

En el siguiente capítulo veremos como los transformadores de mónadas se pueden usar para dar una solución a una queja común cuando se trabaja con código JavaScript asíncrono; el problema del _infierno de retrollamadas_.
