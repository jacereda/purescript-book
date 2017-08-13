# Gráficos con Canvas

## Objetivos del capítulo

Este capítulo será un ejemplo extendido enfocado en el paquete `purescript-canvas`, que proporciona una forma de generar gráficos 2D desde Purescript usando la API Canvas de HTML5.

## Preparación del proyecto

El módulo de este proyecto introduce las siguientes dependencias de Bower nuevas:

- `purescript-canvas`, que da tipos a los métodos de la API Canvas de HTML5
- `purescript-refs`, que proporciona un efecto secundario para usar _referencias globales mutables_

El código fuente para el capítulo está dividido en un conjunto de módulos, cada uno de los cuales define un método `main`. Las distintas secciones de este capítulo están implementadas en ficheros diferentes, y el módulo `Main` se puede cambiar modificando el comando de construcción de Pulp para ejecutar el método `main` del fichero adecuado en cada momento.

El fichero HTML `html/index.html` contiene un único elemento `canvas` que se usará en cada ejemplo, y un elemento `script` para cargar el código PureScript compilado. Para probar el código de cada sección, abre el fichero HTML en tu navegador.

## Formas simples

El fichero `Example/Rectangle.purs` contiene un ejemplo introductorio simple que dibuja un único rectángulo azul en el centro del lienzo. El módulo importa `Control.Monad.Eff`, y también el módulo `Graphics.Canvas` que contiene acciones en la mónada `Eff` para trabajar con la API Canvas.

La acción `main` comienza, como los otros módulos, usando la acción `getCanvasElementById` para obtener una referencia al objeto lienzo, y la acción `getContext2D` para acceder al contexto de representación 2D del canvas:

```haskell
main = void $ unsafePartial do
  Just canvas <- getCanvasElementById "canvas"
  ctx <- getContext2D canvas
```

_Nota_: la llamada a `unsafePartial` es necesaria ya que el ajuste de patrón sobre el resultado de `getCanvasElementById` es parcial, coincidiendo sólo con el constructor `Just`. Para nuestros propósitos es suficiente, pero en código de producción querremos probablemente ajustarnos también al constructor `Nothing` y proporcionar un mensaje de error adecuado.

Los tipos de estas acciones se pueden averiguar usando PSCi o mirando la documentación:

```haskell
getCanvasElementById :: forall eff. String -> Eff (canvas :: CANVAS | eff) (Maybe CanvasElement)

getContext2D :: forall eff. CanvasElement -> Eff (canvas :: CANVAS | eff) Context2D
```

`CanvasElement` y `Context2D` son tipos definidos en el módulo `Graphics.Canvas`. El mismo módulo define también el efecto `CANVAS` usado por todas las acciones del módulo.

El contexto gráfico `ctx` gestiona el estado del canvas y proporciona métodos para dibujar formas primitivas, fijar estilos y colores, y aplicar transformaciones.

Continuamos fijando el estilo de relleno para que sea azul mediante la acción `setFillStyle`:

```haskell
  setFillStyle "#0000FF" ctx
```

Fíjate en que la acción `setFillStyle` toma el contexto gráfico como argumento. Este es un patrón común en el módulo `Graphics.Canvas`.

Finalmente, usamos la acción `fillPath` para rellenar el rectángulo. `fillPath` tiene el siguiente tipo:

```haskell
fillPath :: forall eff a. Context2D ->
                          Eff (canvas :: CANVAS | eff) a ->
                          Eff (canvas :: CANVAS | eff) a
```

`fillPath` toma un contexto gráfico y otra acción que construye la trayectoria a dibujar. Para construir una trayectoria podemos usar la acción `rect`. `rect` toma un contexto gráfico y un registro que proporciona la posición y tamaño del rectángulo. 

```haskell
  fillPath ctx $ rect ctx
    { x: 250.0
    , y: 250.0
    , w: 100.0
    , h: 100.0
    }
```

Construye el ejemplo del rectángulo proporcionando `Example.Rectangle` como nombre del módulo principal:

```text
$ mkdir dist/
$ pulp build -O --main Example.Rectangle --to dist/Main.js
```

Ahora abre el fichero `html/index.html` y verifica que este código dibuja un rectángulo azul en el centro del lienzo.

## Haciendo uso del polimorfismo de fila

Hay otras formas de representar trayectorias. La función `arc` dibuja un segmento de arco, y las funciones `moveTo`, `lineTo` y `closePath` se pueden usar para dibujar trayectorias lineales por tramos.

El fichero `Shapes.purs` dibuja tres formas: un rectángulo, un segmento de arco y un triángulo.

Hemos visto que la función `rect` toma un registro como argumento. De hecho, las propiedades del rectángulo se definen en un sinónimo de tipo:

```haskell
type Rectangle =
  { x :: Number
  , y :: Number
  , w :: Number
  , h :: Number
  }
```

Las propiedades `x` e `y` representan la ubicación de la esquina superior izquierda, mientras que las propiedades `w` y `h` representan el ancho y alto respectivamente.

Para dibujar un segmento de arco podemos usar la función `arc` pasando un registro con el siguiente tipo:

```haskell
type Arc =
  { x     :: Number
  , y     :: Number
  , r     :: Number
  , start :: Number
  , end   :: Number
  }
```

Aquí, las propiedades `x` e `y` representan el punto central, `r` es el radio, y `start` y `end` representan los extremos del arco en radianes.

Por ejemplo, este código rellena un segmento de arco centrado en `(300, 300)` con radio `50`:

```haskell
  fillPath ctx $ arc ctx
    { x      : 300.0
    , y      : 300.0
    , r      : 50.0
    , start  : Math.pi * 5.0 / 8.0
    , end    : Math.pi * 2.0
    }
```

Date cuenta de que tanto `Rectangle` como `Arc` contienen propiedades `x` e `y` de tipo `Number`. En ambos casos, este par representa un punto. Significa que podemos escribir funciones polimórficas por fila que actúan en cualquier tipo de registro.

Por ejemplo, el módulo `Shapes` define una función `translate` que traslada una forma modificando sus propiedades `x` e `y`:

```haskell
translate
  :: forall r
   . Number
  -> Number
  -> { x :: Number, y :: Number | r }
  -> { x :: Number, y :: Number | r }
translate dx dy shape = shape
  { x = shape.x + dx
  , y = shape.y + dy
  }
```

Fíjate en el tipo polimórfico por fila. Dice que `translate` acepta cualquier registro con propiedades `x` e `y` _y otras propiedades cualesquiera_, y devuelve el mismo tipo de registro. Los campos `x` e `y` se actualizan, pero el resto de campos permanecen intactos.

Esto es un ejemplo de _sintaxis de actualización de registro_ (*record update syntax*). La expresión `shape { ... }` crea un nuevo registro basado en el registro `shape`, actualizando los campos entre llaves a los valores especificados. Date cuenta de que las expresiones entre llaves se separan de sus etiquetas por símbolos igual, no con dos puntos como en los registros literales.

La función `translate` se puede usar tanto con `Rectangle` como con `Arc`, como veremos en el ejemplo `Shapes`.

El tercer tipo de trayectoria dibujada en el ejemplo `Shapes` es una trayectoria lineal por tramos. Aquí está el código correspondiente:

```haskell
  setFillStyle "#FF0000" ctx

  fillPath ctx $ do
    moveTo ctx 300.0 260.0
    lineTo ctx 260.0 340.0
    lineTo ctx 340.0 340.0
    closePath ctx
```

Usamos tres funciones aquí:

- `moveTo` mueve la posición actual de la trayectoria a las coordenadas especificadas
- `lineTo` dibuja un segmento de línea entre la posición actual y las coordenadas especificadas, y actualiza la posición actual
- `closePath` completa la trayectoria dibujando un segmento de línea uniendo la posición actual y la posición inicial

El resultado de este fragmento de código es rellenar un triángulo isosceles.

Construye el ejemplo especificando `Example.Shapes` como módulo principal:

```text
$ pulp build -O --main Example.Shapes --to dist/Main.js
```

y abre `html/index.html` de nuevo para ver el resultado. Debes ver tres tipos de formas dibujadas en el lienzo.

X> ## Ejercicios
X>
X> 1. (Fácil) Experimenta con las funciones `strokePath` y `setStrokeStyle` en cada uno de los ejemplos vistos hasta ahora.
X> 1. (Fácil) Las funciones `fillPath` y `strokePath` se pueden usar para representar trayectorias complejas con un estilo común usando un bloque de notación do dentro del argumento a la función. Intenta cambiar el ejemplo `Rectangle` para que dibuje dos rectángulos, uno al lado del otro, usando la misma llamada a `fillPath`. Intenta dibujar un sector de un círculo usando una combinación de trayectoria lineal por tramos y un segmento de arco.
X> 1. (Medio) Dado el siguiente tipo de registro:
X>
X>     ```haskell
X>     type Point = { x :: Number, y :: Number }
X>     ```
X>
X>     que representa un punto 2D, escribe una función `renderPath` que traza una trayectoria cerrada construida a partir de un número de puntos:
X>
X>     ```haskell
X>     renderPath
X>       :: forall eff
X>        . Context2D
X>       -> Array Point
X>       -> Eff (canvas :: Canvas | eff) Unit
X>     ```
X>
X>     Dada una función
X>
X>     ```haskell
X>     f :: Number -> Point
X>     ```
X>
X>     que toma un `Number` entre `0` y `1` como argumento y devuelve un `Point`, escribe una acción que dibuja `f` usando tu función `renderPath`. Tu acción debe aproximar la trayectoria muestreando `f` en un conjunto finito de puntos.
X>
X>     Experimenta dibujando diferentes trayectorias alterando la función `f`.

## Dibujando círculos aleatorios

El fichero `Example/Random.purs` contiene un ejemplo que usa la mónada `Eff` para intercalar dos tipos diferentes de efectos secundarios: generación de números aleatorios y manipulación del lienzo. El ejemplo dibuja cien círculos generados aleatoriamente en el lienzo.

La acción `main` obtiene una referencia al contexto gráfico como antes, y fija los estilos de trazo y relleno:

```haskell
  setFillStyle "#FF0000" ctx
  setStrokeStyle "#000000" ctx
```

A continuación, el código usa la función `for_` para iterar por los enteros entre `0` y `100`:

```haskell
  for_ (1 .. 100) \_ -> do
```

En cada iteración, el bloque de notación do comienza generando tres números aleatorios distribuidos entre `0` y `1`. Estos números representan las coordenadas `x` e `y` y el radio de un círculo.

```haskell
    x <- random
    y <- random
    r <- random
```

A continuación, para cada círculo, el código crea un `Arc` basándose en estos parámetros y finalmente rellena y perfila el arco con los estilos actuales:

```haskell
    let path = arc ctx
         { x     : x * 600.0
         , y     : y * 600.0
         , r     : r * 50.0
         , start : 0.0
         , end   : Math.pi * 2.0
         }
    fillPath ctx path
    strokePath ctx path
```

Construye este ejemplo especificando el módulo `Example.Random` como módulo principal:

```text
$ pulp build -O --main Example.Random --to dist/Main.js
```

y mira el resultado abriendo `html/index.html`.

## Transformaciones

Podemos hacer más cosas en el lienzo que dibujar formas simples. Todos los lienzos mantienen una transformación que se usa para transformar las formas antes de dibujarse. Las formas pueden ser trasladadas, rotadas, escaladas y distorsionadas.

La biblioteca `purescript-canvas` soporta estas transformaciones usando las siguientes funciones:

```haskell
translate :: forall eff
           . TranslateTransform
          -> Context2D
          -> Eff (canvas :: Canvas | eff) Context2D

rotate    :: forall eff
           . Number
          -> Context2D
          -> Eff (canvas :: Canvas | eff) Context2D

scale     :: forall eff
           . ScaleTransform
          -> Context2D
          -> Eff (canvas :: CANVAS | eff) Context2D

transform :: forall eff
           . Transform
          -> Context2D
          -> Eff (canvas :: CANVAS | eff) Context2D
```

La acción `translate` realiza una traslación cuyas componentes se especifican en las propiedades del registro `TranslateTransform`.

La acción `rotate` realiza una rotación respecto al origen de un número de radianes especificado como primer argumento.

La acción `scale` realiza un escalado con origen en el centro. El registro `ScaleTransform` especifica los factores de escala junto a los ejes `x` e `y`.

Finalmente, `transform` es la acción más general de las cuatro. Realiza una transformación afín especificada por una matriz.

Cualquier forma dibujada tras invocar a estas acciones recibirá automáticamente la transformación apropiada.

De hecho, el efecto de cada una de estas funciones es _postmultiplicar_ la transformación por la transformación actual del contexto. El resultado es que si se aplican múltiples transformaciones una tras otra, sus efectos se aplican en orden inverso:

```haskell
transformations ctx = do
  translate { translateX: 10.0, translateY: 10.0 } ctx
  scale { scaleX: 2.0, scaleY: 2.0 } ctx
  rotate (Math.pi / 2.0) ctx

  renderScene
```

El efecto de esta secuencia de acciones es que la escena se rota, se escala, y finalmente se traslada.

## Preservando el contexto

Un caso de uso común es representar un subconjunto de la escena usando una transformación y restablecer la transformación a continuación.

La API de Canvas proporciona los métodos `save` y `restore` que manipulan una _pila_ de estados asociados con el lienzo. `purescript-canvas` envuelve esta funcionalidad en las siguientes funciones:

```haskell
save
  :: forall eff
   . Context2D
  -> Eff (canvas :: CANVAS | eff) Context2D

restore
  :: forall eff
   . Context2D
  -> Eff (canvas :: CANVAS | eff) Context2D
```

La acción `save` apila el estado actual del contexto (incluyendo la transformación actual y cualquier estilo) en la pila, y la acción `restore` desapila el estado superior de la pila y lo restaura.

Esto permite salvar el estado actual, aplicar algunos estilos y transformaciones, dibujar primitivas, y finalmente restaurar la transformación y estado originales. Por ejemplo, la siguiente función dibuja algo en el canvas, pero aplica una rotación antes de hacerlo y restaura la transformación a continuación:

```haskell
rotated ctx render = do
  save ctx
  rotate Math.pi ctx
  render
  restore ctx
```

Para abstraer casos de uso comunes usando funciones de orden mayor, la biblioteca `purescript-canvas` proporciona la función `withContext`, que realiza una acción sobre el lienzo al tiempo que preserva el estado original del contexto:

```haskell
withContext
  :: forall eff a
   . Context2D
  -> Eff (canvas :: CANVAS | eff) a
  -> Eff (canvas :: CANVAS | eff) a          
```

Podríamos reescribir la función `rotated` anterior usando `withContext` como sigue:

```haskell
rotated ctx render =
  withContext ctx do
    rotate Math.pi ctx
    render
```

## Estado mutable global

En esta sección usaremos el paquete `purescript-refs` para demostrar otro efecto en la mónada `Eff`.

El módulo `Control.Monad.Eff.Ref` proporciona un constructor de tipo para referencias a estado global mutable y un efecto asociado:

```text
> import Control.Monad.Eff.Ref

> :kind Ref
Type -> Type

> :kind REF
Control.Monad.Eff.Effect
```

Un valor de tipo `Ref a` es una referencia a una celda mutable que contiene un valor de tipo `a`, bastante parecido a `STRef h a` que vimos en el capítulo anterior. La diferencia es que, mientras que el efecto `ST` se puede eliminar usando `runST`, el efecto `Ref` no proporciona un gestor. Mientras que `ST` se usa para seguir la pista a la mutación local segura, `Ref` se usa para mutaciones globales. Por lo tanto se debe usar escasamente.

El fichero `Example/Refs.purs` contiene un ejemplo que usa el efecto `REF` para detectar pulsaciones de ratón en el elemento `canvas`.

El código comienza creando una nueva referencia conteniendo el valor `0` mediante la acción `newRef`:

```haskell
  clickCount <- newRef 0
```

Dentro del gestor de pulsación de ratón, la acción `modifyRef` se usa para actualizar la cuenta de pulsaciones:

```haskell
    modifyRef clickCount (\count -> count + 1)
```

La acción `readRef` se usa para leer la nueva cuenta de pulsaciones:

```haskell
    count <- readRef clickCount
```

En la función `render`, usamos el contador de pulsaciones para determinar qué transformación aplicar a un rectángulo:

```haskell
    withContext ctx do
      let scaleX = Math.sin (toNumber count * Math.pi / 4.0) + 1.5
      let scaleY = Math.sin (toNumber count * Math.pi / 6.0) + 1.5

      translate { translateX: 300.0, translateY:  300.0 } ctx
      rotate (toNumber count * Math.pi / 18.0) ctx
      scale { scaleX: scaleX, scaleY: scaleY } ctx
      translate { translateX: -100.0, translateY: -100.0 } ctx

      fillPath ctx $ rect ctx
        { x: 0.0
        , y: 0.0
        , w: 200.0
        , h: 200.0
        }
```

Esta acción usa `withContext` para preservar la transformación original, y aplica la siguiente secuencia de transformaciones (recuerda que las transformaciones se aplican de abajo hacia arriba):

- El rectángulo se traslada a `(-100, -100)` de manera que su centro descanse en el origen
- Escalamos el rectángulo con respecto al origen
- Rotamos el rectángulo un múltiplo de `10` grados respecto al origen
- Trasladamos el rectángulo a `(300, 300)` de manera que su centro quede en el centro del lienzo

Construye el ejemplo:

```text
$ pulp build -O --main Example.Refs --to dist/Main.js
```

y abre el fichero `html/index.html`. Si pulsas sobre el lienzo repetidas veces verás un rectángulo verde rotando sobre el centro del lienzo.

X> ## Ejemplos
X>
X> 1. (Fácil) Escribe una función de orden mayor que traza y rellena una trayectoria simultáneamente. Reescribe el ejemplo `Random.purs` usando tu función.
X> 1. (Medio) Usa los efectos `RANDOM` y `DOM` para crear una aplicación que dibuja en el lienzo un círculo con posición, color y radio aleatorios cuando se pulsa el botón del ratón.
X> 1. (Medio) Escribe una función que transforme la escena rotándola alrededor de un punto dado. _Pista_: usa una traslación en primer lugar para llevar la escena al origen.

## Sistemas-L

En este último ejemplo, usaremos el paquete `purescript-canvas` para escribir una función que represente _sistemas-L_ (o _sistemas de Lindenmayer_).

Un sistema-L se define mediante un _alfabeto_, una secuencia inicial de letras del alfabeto y un conjunto de _reglas de producción_. Cada regla de producción toma una letra del alfabeto y devuelve una secuencia de letras de reemplazo. Este proceso se itera un cierto número de veces comenzando con la secuencia inicial de letras.

Si cada letra del alfabeto se asocia con alguna instrucción a realizar en el lienzo, el sistema-L se puede dibujar siguiendo las instrucciones por orden.

Por ejemplo, supongamos que el alfabeto consta de las letras `L` (gira a la izquierda), `R` (gira a la derecha) y `F` (avanza). Podemos definir la siguiente regla de producción:

```text
L -> L
R -> R
F -> FLFRRFLF
```

Si comenzamos con la secuencia inicial "FRRFRRFRR" e iteramos, obtenemos la siguiente secuencia:

```text
FRRFRRFRR
FLFRRFLFRRFLFRRFLFRRFLFRRFLFRR
FLFRRFLFLFLFRRFLFRRFLFRRFLFLFLFRRFLFRRFLFRRFLF...
```

y así sucesivamente. Dibujar una trayectoria lineal por tramos correspondiente a este conjunto de instrucciones aproxima una curva llamada la _curva de Koch_. Incrementar el número de iteraciones incrementa la resolución de la curva.

Traduzcamos esto al lenguaje de los tipos y funciones.

Podemos representar nuestro alfabeto mediante un tipo algebraico. Para nuestro ejemplo podemos usar el siguiente tipo:

```haskell
data Alphabet = L | R | F
```

Este tipo de datos define un constructor de datos para cada letra de nuestro alfabeto.

¿Cómo podemos representar la secuencia inicial de letras? Es simplemente un array de letras de nuestro alfabeto que llamaremos frase (`Sentence`):

```haskell
type Sentence = Array Alphabet

initial :: Sentence
initial = [F, R, R, F, R, R, F, R, R]
```

Nuestras reglas de producción se pueden expresar como una función de `Alphabet` a `Sentence` como sigue:

```haskell
productions :: Alphabet -> Sentence
productions L = [L]
productions R = [R]
productions F = [F, L, F, R, R, F, L, F]
```

Esto es una copia directa de la especificación de arriba.

Ahora podemos implementar una función `lsystem` que toma una especificación de esta forma y la dibuja en el lienzo. ¿Qué tipo debe tener `lsystem`? Bien, necesita tomar valores como `initial` y `productions` como argumentos, así como una función para dibujar una letra del alfabeto en el canvas.

Aquí tenemos una primera aproximación al tipo de `lsystem`:

```haskell
forall eff. Sentence
         -> (Alphabet -> Sentence)
         -> (Alphabet -> Eff (canvas :: Canvas | eff) Unit)
         -> Int
         -> Eff (canvas :: CANVAS | eff) Unit
```

Los dos primeros argumentos corresponden a los valores `initial` y `productions`.

El tercer argumento representa una función que toma una letra del alfabeto y la _interpreta_ realizando algunas acciones sobre el lienzo. En nuestro ejemplo, esto significaría girar a la izquierda en el caso de la letra `L`, girar a la derecha en el caso de la letra `R`, y avanzar en el caso de la letra `F`.

El argumento final es un número que representa el número de iteraciones de las reglas de producción que queremos realizar.

La primera observación es que la función `lsystem` no debe funcionar sólo para un único tipo `Alphabet`, sino para cualquier tipo, de manera que debemos generalizar nuestro tipo. Cambiemos `Alphabet` y `Sentence` por `a` y `Array a` para alguna variable de tipo cuantificada `a`:

```haskell
forall a eff. Array a
           -> (a -> Array a)
           -> (a -> Eff (canvas :: CANVAS | eff) Unit)
           -> Int
           -> Eff (canvas :: CANVAS | eff) Unit
```

La segunda observación es que para implementar instrucciones como "gira a la izquierda" y "gira a la derecha" necesitamos mantener algún estado, esto es, la dirección en que la trayectoria se está moviendo en todo momento. Necesitamos modificar nuestra función para pasar el estado durante el cálculo. De nuevo, la función `lsystem` debe funcionar para cualquier tipo de estado, así que lo representaremos usando la variable de tipo `s`.

Necesitamos añadir el tipo `s` en tres sitios:

```haskell
forall a s eff. Array a
             -> (a -> Array a)
             -> (s -> a -> Eff (canvas :: CANVAS | eff) s)
             -> Int
             -> s
             -> Eff (canvas :: CANVAS | eff) s
```

En primer lugar, el tipo `s` ha sido añadido como el tipo de un argumento adicional a `lsystem`. Este argumento representará el estado inicial del sistema-L.

El tipo `s` también aparece como argumento y tipo de retorno de la función de interpretación (el tercer argumento a `lsystem`). La función de interpretación recibirá ahora el estado actual del sistema-L como argumento y devolverá un nuevo estado actualizado.

En el caso de nuestro ejemplo, podemos usar el siguiente tipo para representar el estado:

```haskell
type State =
  { x :: Number
  , y :: Number
  , theta :: Number
  }
```

Las propiedades `x` e `y` representan la posición actual de la trayectoria, y la propiedad `theta` representa la dirección actual de la trayectoria especificada como el ángulo entre la dirección de la trayectoria y el eje horizontal en radianes.

El estado inicial del sistema se puede especificar como sigue:

```haskell
initialState :: State
initialState = { x: 120.0, y: 200.0, theta: 0.0 }
```

Ahora intentemos implementar la función `lsystem`. Veremos que su definición es notablemente simple.

Parece razonable que `lsystem` recurra sobre su cuarto argumento (de tipo `Int`). En cada paso de la recursividad, la frase actual cambiará siendo actualizada mediante las reglas de producción. Teniendo en cuesta esto, asignemos nombres a los argumentos de la función y deleguemos en una función auxiliar:

```haskell
lsystem :: forall a s eff
         . Array a
        -> (a -> Array a)
        -> (s -> a -> Eff (canvas :: CANVAS | eff) s)
        -> Int
        -> s
        -> Eff (canvas :: CANVAS | eff) s
lsystem init prod interpret n state = go init n
  where
```

La función `go` trabaja recursivamente sobre su segundo argumento. Hay dos casos: cuando `n` es cero y cuando no lo es.

En el primer caso, la recursividad finaliza y simplemente necesitamos interpretar la frase actual de acuerdo a la función de interpretación. Tenemos una frase de tipo `Array a`, un estado de tipo `s` y una función de tipo `s -> a -> Eff (canvas :: CANVAS | eff) s`. Parece un trabajo para la función `foldM` que definimos antes y que está disponible en el paquete `purescript-control`:

```haskell
  go s 0 = foldM interpret state s
```

¿Que pasa en el caso en que no es cero? En ese caso, podemos simplemente aplicar las reglas de producción a cada letra de la frase actual, concatenando los resultados, y repetir llamando a `go` recursivamente:

```haskell
  go s n = go (concatMap prod s) (n - 1)
```

¡Eso es todo! Fíjate en cómo el uso de funciones de orden mayor como `foldM` y `concatMap` nos ha permitido comunicar nuestras ideas de manera concisa.

Sin embargo aún no hemos terminado. El tipo que hemos dado todavía es demasiado específico. Fíjate en que no usamos ninguna operación de canvas en ningún sitio de nuestra implementación. Tampoco usamos la estructura de la mónada `Eff` para nada. De hecho, ¡nuestra función es válida para _cualquier_ mónada `m`!

Aquí tenemos el tipo más general de `lsystem` de la manera en que se especifica en el código fuente que acompaña este capítulo:

```haskell
lsystem :: forall a m s
         . Monad m
        => Array a
        -> (a -> Array a)
        -> (s -> a -> m s)
        -> Int
        -> s
        -> m s
```

Podemos entender este tipo como si dijese que nuestra función de interpretación es libre de tener efectos secundarios, capturados por la mónada `m`. Puede dibujar en el lienzo, o imprimir información a la consola, o soportar fallos o múltiples valores de retorno. Animamos al lector a que intente escribir sistemas-L que usen estos tipos de efectos secundarios.

Esta función es un buen ejemplo de la potencia de separar los datos de la implementación. La ventaja de este enfoque es que ganamos la libertad de interpretar nuestros datos de varias maneras distintas. Podemos incluso factorizar `lsystem` en dos funciones más pequeñas: la primera construiría una frase usando aplicación repetida de `concatMap`, y la segunda interpretaría la frase usando `foldM`. Dejamos esto también como un ejercicio para el lector.

Completemos nuestro ejemplo implementando la función de interpretación. El tipo de `lsystem` nos dice que su firma debe ser `s -> a -> m s` para algunos tipos `a` y `s` y un constructor de tipo `m`. Sabemos que queremos que `a` sea `Alphabet` y `s` sea `State`, y para la mónada `m` podemos elegir `Eff (canvas :: CANVAS)`. Esto nos da el siguiente tipo:

```haskell
interpret :: State -> Alphabet -> Eff (canvas :: CANVAS) State
```

Para implementar esta función necesitamos gestionar los tres constructores del tipo `Alphabet`. Para interpretar las letras `L` (girar a la izquierda), y `R` (girar a la derecha), simplemente tenemos que actualizar el estado para cambiar el ángulo `theta` de manera apropiada:

```haskell
interpret state L = pure $ state { theta = state.theta - Math.pi / 3 }
interpret state R = pure $ state { theta = state.theta + Math.pi / 3 }
```

Para interpretar la letra `F` (avanzar), podemos calcular la nueva posición de la trayectoria, dibujar un segmento y actualizar el estado como sigue:

```haskell
interpret state F = do
  let x = state.x + Math.cos state.theta * 1.5
      y = state.y + Math.sin state.theta * 1.5
  moveTo ctx state.x state.y
  lineTo ctx x y
  pure { x, y, theta: state.theta }
```

Date cuenta de que en el código fuente de este capítulo, la función `interpret` se define usando una ligadura `let` dentro de la función `main`, de manera que el nombre `ctx` esté en ámbito. Sería posible también mover el contexto al tipo `State`, pero esto no sería apropiado porque no es una parte cambiante del estado del sistema.

Para representar este sistema-L podemos simplemente usar la acción `strokePath`:

```haskell
strokePath ctx $ lsystem initial productions interpret 5 initialState
```

Compila el ejemplo del sistema-L usando

```text
$ pulp build -O --main Example.LSystem --to dist/Main.js
```

y abre `html/index.html`. Debes ver la curva de Koch dibujada en el lienzo.

X> ## Ejercicios
X>
X> 1. (Fácil) Modifica el ejemplo del sistema-L anterior para que use `fillPath` en lugar de `strokePath`. _Pista_: necesitarás incluir una llamada a `closePath` y mover la llamada a `moveTo` fuera de la función `interpret`.
X> 1. (Fácil) Intenta cambiar las distintas constantes numéricas del código para comprender su efecto en el sistema representado.
X> 1. (Medio) Descompón la función `lsystem` en dos funciones más pequeñas. La primera debe construir la frase final usando aplicación repetida de `concatMap` y la segunda debe usar `foldM` para interpretar el resultado.
X> 1. (Medio) Añade una sombra a la forma rellena usando las acciones `setShadowOffsetX`, `setShadowOffsetY`, `setShadowBlur` y `setShadowColor`. _Pista_: usa PSCi para averiguar los tipos de estas funciones.
X> 1. (Medio) El ángulo de las esquinas es actualmente una constante (`pi/3`). Podemos moverlo al tipo de datos del alfabeto, lo que permite que sea alterado por las reglas de producción:
X>
X>     ```haskell
X>     type Angle = Number
X>     
X>     data Alphabet = L Angle | R Angle | F
X>     ```
X>     
X>     ¿Cómo se puede usar esta nueva información en las reglas de producción para crear formas interesantes?
X> 1. (Difícil) Un sistema-L viene dado por un alfabeto con cuatro letras: `L` (gira a la izquierda 60 grados), `R` (gira a la derecha 60 grados), `F` (avanza) y `M` (avanza también).
X>
X>     La frase inicial del sistema es una única letra `M`.
X>
X>     Las reglas de producción son como sigue:
X>
X>     ```text
X>     L -> L
X>     R -> R
X>     F -> FLMLFRMRFRMRFLMLF
X>     M -> MRFRMLFLMLFLMRFRM
X>     ```
X>
X>     Representa este sistema-L. _Nota_: necesitarás decrementar el número de iteraciones de las reglas de producción, ya que el tamaño de la frase final crece exponencialmente con el número de iteraciones.
X>
X>     Ahora fíjate en la simetría entre `F` y `M` en las reglas de producción. Las dos instrucciones de avance pueden ser diferenciadas mediante un valor `Boolean` usando el siguiente tipo de alfabeto:
X>
X>     ```haskell
X>     data Alphabet = L | R | F Boolean
X>     ```
X>
X>     Implementa de nuevo este sistema-L usando esta representación para el alfabeto.
X> 1. (Difícil) Usa una mónada diferente `m` para la función de interpretación. Puedes intentar usar el efecto `CONSOLE` para escribir el sistema-L a la consola, o usar el efecto `RANDOM` para aplicar mutaciones aleatorias al tipo de estado.

## Conclusión

En este capítulo hemos aprendido cómo usar la API Canvas de HTML5 desde PureScript usando la biblioteca `purescript-canvas`. Vimos también una demostración práctica de muchas de las técnicas que ya habíamos aprendido: asociaciones y pliegues, registros y polimorfismo de fila, y la mónada `Eff` para gestionar efectos secundarios.

Los ejemplos demuestran también la potencia de las funciones de orden mayor y la _separación de datos de la implementación_. Sería posible extender estas ideas para separar por completo la representación de una escena de su función de dibujado usando un tipo de datos algebraico, por ejemplo:

```haskell
data Scene
  = Rect Rectangle
  | Arc Arc
  | PiecewiseLinear (Array Point)
  | Transformed Transform Scene
  | Clipped Rectangle Scene
  | ...
```

Este es el enfoque tomado por el paquete `purescript-drawing` y aporta la flexibilidad de ser capaz de manipular la escena como datos de varias maneras antes de dibujar.

En el siguiente capítulo veremos cómo implementar bibliotecas como `purescript-canvas` que envuelven funcionalidad JavaScript existente, usando la _interfaz para funciones externas_ de PureScript.
