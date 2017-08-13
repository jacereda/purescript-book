# El infierno de retrollamadas (*callback hell*)

## Objetivos del capítulo

En este capítulo veremos cómo las herramientas que hemos visto hasta ahora (a saber, transformadores de mónada y funtores aplicativos) se pueden usar para resolver algunos problemas del mundo real. En particular, veremos cómo podemos resolver el problema del infierno de retrollamadas.

## Preparación del proyecto

El código fuente de este capítulo se puede compilar y ejecutar usando `pulp run`. También es necesario instalar el módulo `request` usando NPM:

```text
npm install
```

## El problema

El código asíncrono en JavaScript usa normalmente _retrollamadas_ (*callbacks*) para estructurar el flujo del programa. Por ejemplo, para leer texto de un fichero, el método preferido es usar la función `readFile` y pasar una retrollamada (una función que se llamará cuando el texto esté disponible):

```javascript
function readText(onSuccess, onFailure) {
  var fs = require('fs');
  fs.readFile('file1.txt', { encoding: 'utf-8' }, function (error, data) {
    if (error) {
      onFailure(error.code);
    } else {
      onSuccess(data);
    }   
  });
}
```

Sin embargo, si hay involucradas múltiples operaciones, esto puede llevar rápidamente a retrollamadas anidadas, lo que puede acabar en código difícil de leer:

```javascript
function copyFile(onSuccess, onFailure) {
  var fs = require('fs');
  fs.readFile('file1.txt', { encoding: 'utf-8' }, function (error, data1) {
    if (error) {
      onFailure(error.code);
    } else {
      fs.writeFile('file2.txt', data, { encoding: 'utf-8' }, function (error) {
        if (error) {
          onFailure(error.code);
        } else {
          onSuccess();
        }
      });
    }   
  });
}
```

Una solución a este problema es descomponer las llamadas asíncronas individuales en sus propias funciones:

```javascript
function writeCopy(data, onSuccess, onFailure) {
  var fs = require('fs');
  fs.writeFile('file2.txt', data, { encoding: 'utf-8' }, function (error) {
    if (error) {
      onFailure(error.code);
    } else {
      onSuccess();
    }
  });
}

function copyFile(onSuccess, onFailure) {
  var fs = require('fs');
  fs.readFile('file1.txt', { encoding: 'utf-8' }, function (error, data) {
    if (error) {
      onFailure(error.code);
    } else {
      writeCopy(data, onSuccess, onFailure);
    }   
  });
}
```

Esta solución funciona, pero tiene algunos problemas:

- Es necesario pasar resultados intermedios a funciones asíncronas como argumentos de función, de la misma manera que hemos pasado `data` a `writeCopy` arriba. Esto está bien para funciones pequeñas, pero si hay muchas retrollamadas involucradas, las dependencias de datos pueden volverse complejas, resultando en muchos argumentos de función adicionales.
- Hay un patrón común. Las retrollamadas `onSuccess` y `onFailure` se especifican normalmente como argumentos a cada función asíncrona. Pero este patrón se tiene que documentar en la documentación del módulo que acompaña el código fuente. Es mejor capturar este patrón en el sistema de tipos y usarlo para forzar su uso.

A continuación veremos cómo usar las técnicas que hemos aprendido hasta ahora para resolver estos problemas.

## La mónada de continuación

Traduzcamos el ejemplo `copyFile` de arriba a PureScript usando la FFI. Al hacerlo, la estructura del cálculo resultará evidente, y seremos conducidos de manera natural a un transformador de mónada definido en el paquete `purescript-transformers`; el transformador de mónada de continuación `ContT`.

_Nota_: en la práctica no es necesario escribir estas funciones a mano cada vez. Puedes encontrar funciones de entrada/salida asíncrona en las bibliotecas `purescript-node-fs` y `purescript-node-fs-aff`.

Primero tenemos que dar tipos a `readFile` y `writeFile` usando la FFI. Comencemos definiendo algunos sinónimos de tipo y un nuevo efecto para entrada/salida de fichero:

```haskell
foreign import data FS :: Effect

type ErrorCode = String
type FilePath = String
```

`readFile` toma un nombre de fichero y una retrollamada que toma dos argumentos. Si el fichero ha sido leído con éxito, el segundo argumento tendrá el contenido del fichero, y si no, el primer argumento se usará para indicar el error.

En nuestro caso, envolveremos `readFile` con una función que toma dos retrollamadas: una retrollamada de error (`onFailure`) y una retrollamada de resultado (`onSuccess`), igual que hicimos con `copyFile` y `writeCopy` arriba. Usando el soporte de funciones de múltiples argumentos de `Data.Function` por simplicidad, nuestra función envuelta `readFileImpl` puede tener esta pinta:

```haskell
foreign import readFileImpl
  :: forall eff
   . Fn3 FilePath
         (String -> Eff (fs :: FS | eff) Unit)
         (ErrorCode -> Eff (fs :: FS | eff) Unit)
         (Eff (fs :: FS | eff) Unit)
```

En el módulo externo JavaScript, `readFileImpl` estaría definida como:

```javascript
exports.readFileImpl = function(path, onSuccess, onFailure) {
  return function() {
    require('fs').readFile(path, {
      encoding: 'utf-8'
    }, function(error, data) {
      if (error) {
        onFailure(error.code)();
      } else {
        onSuccess(data)();
      }
    });
  };
};
```

Esta firma de tipo indica que `readFileImpl` toma tres argumentos: una ruta de fichero, una retrollamada de éxito y una retrollamada de error, y devuelve un cálculo con efectos secundarios que devuelve un resultado vacío (`Unit`). Fíjate en que a las retrollamadas les damos tipos que usan la mónada `Eff` para registrar sus efectos.

Debes intentar entender por qué esta implementación tiene la representación correcta en tiempo de ejecución para su tipo.

`writeFileImpl` es muy similar; difiere únicamente en que el contenido del fichero se pasa a la función, no a la retrollamada. Su implementación tiene esta pinta:

```haskell
foreign import writeFileImpl
  :: forall eff
   . Fn4 FilePath
         String
         (Eff (fs :: FS | eff) Unit)
         (ErrorCode -> Eff (fs :: FS | eff) Unit)
         (Eff (fs :: FS | eff) Unit)
```

```javascript
exports.writeFileImpl = function(path, data, onSuccess, onFailure) {
  return function() {
    require('fs').writeFile(path, data, {
      encoding: 'utf-8'
    }, function(error) {
      if (error) {
        onFailure(error.code)();
      } else {
        onSuccess();
      }
    });
  };
};
```

Dadas estas declaraciones FFI, podemos escribir las implementaciones de `readFile` y `writeFile`. Estas usarán el módulo `Data.Function.Uncurried` para convertir las ligaduras FFI de múltiples argumentos en funciones currificadas normales de PureScript, y que por lo tanto tienen tipos algo más legibles.

Además, en lugar de requerir dos retrollamadas, una para éxitos y otra para fallos, podemos requerir una única retrollamada que responde a ambas cosas. Esto es, la nueva retrollamda toma un valor en la mónada `Either ErrorCode` como argumento:

```haskell
readFile
  :: forall eff
   . FilePath
  -> (Either ErrorCode String -> Eff (fs :: FS | eff) Unit)
  -> Eff (fs :: FS | eff) Unit
readFile path k =
  runFn3 readFileImpl
         path
         (k <<< Right)
         (k <<< Left)

writeFile
  :: forall eff
   . FilePath
  -> String
  -> (Either ErrorCode Unit -> Eff (fs :: FS | eff) Unit)
  -> Eff (fs :: FS | eff) Unit
writeFile path text k =
  runFn4 writeFileImpl
         path
         text
         (k $ Right unit)
         (k <<< Left)
```

Ahora podemos identificar un patrón importante. Cada una de estas funciones toma una retrollamada que devuelve un valor en alguna mónada (en este caso, `Eff (fs :: FS | eff)`) y devuelve un valor en _la misma mónada_. Esto significa que cuando la primera retrollamada devuelve un resultado, esa mónada se puede usar para ligar el resultado a la entrada de la siguiente función asíncrona. De hecho, eso es exactamente lo que hicimos a mano en el ejemplo `copyFile`.

Esto es la base del _transformador de mónada de continuación_, que está definido en el módulo `Control.Monad.Cont.Trans` de `purescript-transformers`.

`ContT` se define como un newtype como sigue:

```haskell
newtype ContT r m a = ContT ((a -> m r) -> m r)
```

Una _continuación_ es simplemente otro nombre para una retrollamada. Una continuación captura el _resto_ de un cálculo; en nuestro caso, qué sucede después de que el resultado sea proporcionado tras una llamada asíncrona.

El argumento al constructor de datos `ContT` se parece notablemente a los tipos de `readFile` y `writeFile`. De hecho, si hacemos que el tipo `a` sea el tipo `Either ErrorCode String`, `r` sea `Unit` y `m` la mónada `Eff (fs :: FS | eff)`, podemos recuperar la parte derecha del tipo de `readFile`.

Esto motiva el siguiente sinónimo de tipo, definiendo una mónada `Async`, que usaremos para componer acciones asíncronas como `readFile` y `writeFile`:

```haskell
type Async eff = ContT Unit (Eff eff)
```

Para nuestros propósitos, siempre usaremos `ContT` para transformar la mónada `Eff`, y el tipo `r` será siempre `Unit`, pero esto no es obligatorio.

Podemos tratar `readFile` y `writeFile` como cálculos en la mónada `Async` aplicando simplemente el constructor de datos `ContT`:

```haskell
readFileCont
  :: forall eff
   . FilePath
  -> Async (fs :: FS | eff) (Either ErrorCode String)
readFileCont path = ContT $ readFile path

writeFileCont
  :: forall eff
   . FilePath
  -> String
  -> Async (fs :: FS | eff) (Either ErrorCode Unit)
writeFileCont path text = ContT $ writeFile path text
```

Con eso, podemos escribir nuestra rutina de copia de ficheros usando notación do para el transformador de mónada `ContT`:

```haskell
copyFileCont
  :: forall eff
   . FilePath
  -> FilePath
  -> Async (fs :: FS | eff) (Either ErrorCode Unit)
copyFileCont src dest = do
  e <- readFileCont src
  case e of
    Left err -> return $ Left err
    Right content -> writeFileCont dest content
```

Fíjate en cómo la naturaleza asíncrona de `readFileCont` queda oculta por la ligadura monádica expresada usando notación do; parece código síncrono, pero la mónada `ContT` es la que se ocupa de hilar nuestras funciones asíncronas.

Podemos ejecutar este cálculo usando el gestor `runContT` suministrando una continuación. La continuación representa _qué hacer a continuación_, es decir, qué hacer cuando la rutina asíncrona de copia de ficheros finaliza. Para nuestro ejemplo simple, podemos elegir `logShow` como función de continuación, que imprimirá el resultado de tipo `Either ErrorCode Unit` a la consola:

```haskell
import Prelude

import Control.Monad.Eff.Console (logShow)
import Control.Monad.Cont.Trans (runContT)

main =
  runContT
    (copyFileCont "/tmp/1.txt" "/tmp/2.txt")
    logShow
```

X> ## Ejercicios
X>
X> 1. (Fácil) Usa `readFileCont` y `writeFileCont` para escribir una función que concatena dos ficheros de texto.
X> 1. (Medio) Usa la FFI para dar un tipo apropiado a la función `setTimeout`. Escribe una función envoltorio usando la mónada `Async`:
X>
X>     ```haskell
X>     type Milliseconds = Int
X>
X>     foreign import data TIMEOUT :: Effect
X>
X>     setTimeoutCont
X>       :: forall eff
X>        . Milliseconds
X>       -> Async (timeout :: TIMEOUT | eff) Unit
X>     ```

## Poniendo ExceptT a trabajar

Esta solución funciona, pero puede mejorarse.

En la implementación de `copyFileCont` tuvimos que usar ajuste de patrones para analizar el resultado del cálculo `readFileCont` (de tipo `Either ErrorCode String`) para determinar qué hacer a continuación. Sin embargo, sabemos que la mónada `Either` tiene un transformador de mónada correspondiente, `ExceptT`, de forma que es razonable esperar que podamos usar `ExceptT` con `ContT` para combinar los dos efectos de cálculo asíncrono y gestión de errores.

De hecho es posible, y podemos ver por qué si miramos la definición de `ExceptT`:

```haskell
newtype ExceptT e m a = ExceptT (m (Either e a))
```

`ExceptT` simplemente cambia el resultado de la mónada subyacente de `a` a `Either e a`. Esto significa que podemos reescribir `copyFileCont` transformando nuestra pila de mónadas actual con el transformador `ExceptT ErrorCode`. Es tan simple como aplicar el constructor `ExceptT` a nuestra solución existente:

```haskell
readFileContEx
  :: forall eff
   . FilePath
  -> ExceptT ErrorCode (Async (fs :: FS | eff)) String
readFileContEx path = ExceptT $ readFileCont path

writeFileContEx
  :: forall eff
   . FilePath
  -> String
  -> ExceptT ErrorCode (Async (fs :: FS | eff)) Unit
writeFileContEx path text = ExceptT $ writeFileCont path text
```

Ahora, nuestra rutina de copia de fichero es mucho más simple, ya que la gestión de errores asíncrona queda oculta dentro del transformador de mónada `ExceptT`:

```haskell
copyFileContEx
  :: forall eff
   . FilePath
  -> FilePath
  -> ExceptT ErrorCode (Async (fs :: FS | eff)) Unit
copyFileContEx src dest = do
  content <- readFileContEx src
  writeFileContEx dest content
```

X> ## Ejercicios
X>
X> 1. (Medio) Modifica tu solución a la concatenación de ficheros para que use `ExceptT` para gestionar cualquier error.
X> 1. (Medio) Escribe una función `concatenateMany` para concatenar múltiples ficheros de texto dado un array de rutas de fichero de entrada. _Pista_: usa `traverse`.

## Un cliente HTTP

Como otro ejemplo de uso de `ContT` para gestionar funciones asíncronas, vamos a ver el módulo `Network.HTTP.Client` del código fuente de este capítulo. Este módulo usa la mónada `Async` para soportar peticiones HTTP asíncronas usando el módulo `request`, disponible via NPM.

El módulo `request` suministra una función que toma un URL y una retrollamada, hace una petición HTTP(S) e invoca la retrollamada cuando la respuesta está disponible, o cuando ocurre un error. Aquí tenemos un ejemplo de petición:

```javascript
require('request')('http://purescript.org'), function(err, _, body) {
  if (err) {
    console.error(err);
  } else {
    console.log(body);
  }
});
```

Vamos a recrear este ejemplo simple en PureScript usando la mónada `Async`.

En el módulo `Network.HTTP.Client`, el método `request` está envuelto con una función `getImpl`:

```haskell
foreign import data HTTP :: Effect

type URI = String

foreign import getImpl
  :: forall eff
   . Fn3 URI
         (String -> Eff (http :: HTTP | eff) Unit)
         (String -> Eff (http :: HTTP | eff) Unit)
         (Eff (http :: HTTP | eff) Unit)
```

```javascript
exports.getImpl = function(uri, done, fail) {
  return function() {
    require('request')(uri, function(err, _, body) {
      if (err) {
        fail(err)();
      } else {
        done(body)();
      }
    });
  };
};
```

De nuevo, podemos usar el módulo `Data.Function.Uncurried` para convertir esto en una función currificada PureScript normal. Como antes, combinamos las dos retrollamadas en una, esta vez aceptando un valor de tipo `Either String String`, y aplicamos el constructor `ContT` para construir una acción en nuestra mónada `Async`:

```haskell
get :: forall eff.
  URI ->
  Async (http :: HTTP | eff) (Either String String)
get req = ContT \k ->
  runFn3 getImpl req (k <<< Right) (k <<< Left)
```

X> ## Ejercicios
X>
X> 1. (Fácil) Usa `runContT` para probar `get` en PSCi, imprimiendo los resultados en la consola.
X> 1. (Medio) Usa `ExceptT` para escribir una función `getEx` que envuelve `get`, como hicimos previamente para `readFileCont` y `writeFileCont`.
X> 1. (Difícil) Escribe una función que salva a fichero el cuerpo de la respuesta a una petición usando `getEx` y `writeFileContEx`.

## Cálculos paralelos

Hemos visto cómo usar la mónada `ContT` y la notación do para componer cálculos asíncronos en secuencia. Sería también útil poder componer cálculos asíncronos _en paralelo_.

Si usamos `ContT` para transformar la mónada `Eff`, podemos realizar cálculos en paralelo simplemente iniciando nuestros dos cálculos uno tras otro.

El paquete `purescript-parallel` define una clase de tipos `MonadPar` para mónadas como `Async` que soportan ejecución paralela. Cuando nos encontramos los funtores aplicativos en un capítulo previo, observamos cómo los funtores aplicativos pueden ser útiles para combinar cálculos paralelos. De hecho, una instancia de `Parallel` define una correspondencia entre una mónada `m` (como `Async`) y un funtor aplicativo `f` que se puede usar para combinar cálculos en paralelo:

```haskell
class (Monad m, Applicative f) <= Parallel f m | m -> f, f -> m where
  sequential :: forall a. f a -> m a
  parallel :: forall a. m a -> f a
```

La clase define dos funciones:

- `parallel`, que toma un cálculo en la mónada `m` y lo convierte en cálculos en el funtor aplicativo `f`, y
- `sequential`, que realiza una conversión en sentido inverso.

La biblioteca `purescript-parallel` proporciona una instancia `Parallel` para la mónada `Async`. Usa referencias mutables para combinar acciones `Async` en paralelo, llevando registro de cual de las dos continuaciones se ha llamado. Cuando ambos resultados están disponibles, podemos calcular el resultado final y pasarlo a la continuación principal.

Podemos usar la función `parallel` para crear una versión de nuestra acción `readFileCont` que se puede combinar en paralelo. Aquí hay un simple ejemplo que lee dos ficheros de texto en paralelo, los concatena e imprime el resultado:

```haskell
import Prelude
import Control.Apply (lift2)
import Control.Monad.Cont.Trans (runContT)
import Control.Monad.Eff.Console (logShow)
import Control.Monad.Parallel (parallel, sequential)

main = flip runContT logShow do
  sequential $
   lift2 append
     <$> parallel (readFileCont "/tmp/1.txt")
     <*> parallel (readFileCont "/tmp/2.txt")
```

Fíjate en que, ya que `readFileCont` devuelve un valor de tipo `Either ErrorCode String`, necesitamos elevar la función `append` sobre el constructor de tipo `Either` usando `lift2` para formar nuestra función combinadora.

Dado que los funtores aplicativos soportan elevar funciones de aridad arbitraria, podemos realizar más cálculos en paralelo usando los combinadores aplicativos. ¡Podemos también beneficiarnos de todas las funciones de la biblioteca estándar que trabajan con funtores aplicativos, como `traverse` y `sequence`!

También podemos combinar cálculos paralelos con porciones de código secuencial, usando combinadores aplicativos en un bloque en notación do, o viceversa, usando `parallel` y `sequential` para cambiar los constructores de tipo como corresponda.

X> ## Ejercicios
X>
X> 1. (Fácil) Usa `parallel` y `sequential` para hacer dos peticiones HTTP y recolectar los cuerpos de la respuesta en paralelo. Tu función combinadora debe concatenar ambos cuerpos y tu continuación debe usar `print` para imprimir el resultado a consola.
X> 1. (Medio) El funtor aplicativo que corresponde a `Async` es también una instancia de `Alternative`. El operador `<|>` definido por esta instancia ejecuta dos cálculos en paralelo y devuelve el resultado del cálculo que finaliza primero.
X>
X>     Usa esta instancia de `Alternative` junto a tu función `setTimeoutCont` para definir una función
X>
X>     ```haskell
X>     timeout :: forall a eff
X>              . Milliseconds
X>             -> Async (timeout :: TIMEOUT | eff) a
X>             -> Async (timeout :: TIMEOUT | eff) (Maybe a)
X>     ```
X>
X>     que devuelve `Nothing` si el cálculo especificado no proporciona un resultado en el número de milisegundos especificado.

X> 1. (Medio) `purescript-parallel` también proporciona instancias de la clase `Parallel` para varios transformadores de mónada, incluyendo `ExceptT`.
X>
X>     Reescribe el ejemplo de entrada/salida de fichero paralela para que use `ExceptT` para gestión de errores, en lugar de elevar `append` con `lift2`. Tu solución debe usar el transformador `ExceptT` para transformar la mónada `Async`.
X>
X>     Usa este método para modificar tu función `concatenateMany` de forma que lea múltiples ficheros de entrada en paralelo.
X> 1. (Difícil, Extendido) Supongamos que nos dan una colección de documentos JSON en disco, de forma que cada documento contiene un array de referencias a otros ficheros en disco:
X>
X>     ```javascript
X>     { references: ['/tmp/1.json', '/tmp/2.json'] }
X>     ```
X>     
X>     Escribe una utilidad que toma un único fichero como entrada y recorre todos los ficheros JSON del disco referenciados de manera transitiva por ese fichero, acumulando una lista de todos los ficheros referenciados.
X>
X>     Tu utilidad debe usar la biblioteca `purescript-foreign` para analizar los documentos JSON, y debe extraer los ficheros referenciados por un único fichero en paralelo.

## Conclusión

En este capítulo, hemos visto una demostración práctica de los transformadores de mónada:

- Vimos cómo el idioma común en JavaScript de pasar retrollamadas se puede capturar con el transformador de mónada `ContT`.
- Vimos cómo el problema del infierno de las retrollamadas se puede resolver usando notación do para expresar cálculos asíncronos secuenciales, y un funtor aplicativo para expresar paralelismo.
- Hemos usado `ExceptT` pare expresar _errores asíncronos_.
