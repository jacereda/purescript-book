# Empezando

## Objetivos del capítulo

En este capítulo, el objetivo será preparar un entorno de desarrollo para PureScript y escribir nuestro primer programa en PureScript.

Nuestro primer proyecto será una biblioteca PureScript muy simple, que proporcionará una única función para calcular la longitud de la diagonal de un triángulo rectángulo.

## Introducción

Estas son las herramientas que vamos a usar para preparar nuestro entorno de desarrollo para PureScript:

- [`purs`](http://purescript.org) - El propio compilador de PureScript.
- [`npm`](http://npmjs.org) - El gestor de paquetes de Node, que nos ayudará a instalar el resto de herramientas de desarrollo.
- [Pulp](https://github.com/bodil/pulp) - Una herramienta de línea de comandos que automatiza muchas de las tareas asociadas a la gestión de proyectos PureScript.

El resto del capítulo te guiará en la instalación y configuración de estas herramientas.

## Instalando PureScript

La manera recomendada de instalar el compilador de PureScript es descargar una distribución binaria para tu plataforma en el [sitio web de PureScript](http://purescript.org).

Debes verificar que los ejecutables del compilador de PureScript están disponibles en tu ruta de ejecutables. Intenta ejecutar el compilador de PureScript en la línea de comandos para comprobarlo:

```text
$ purs
```

Otras opciones para instalar el compilador de PureScript:

- Mediante NPM: `npm install -g PureScript`.
- Construir el compilador a partir del código fuente. Las instrucciones están disponibles en el sitio web de PureScript.

## Instalando las herramientas

Si no tienes una instalación funcional de [NodeJS](http://nodejs.org/), debes instalarlo. Esto instalará también el gestor de paquetes `npm` en tu sistema. Asegúrate de que tienes `npm` instalado y disponible en tu ruta de ejecutables.

También necesitarás instalar la herramienta de línea de comandos Pulp y el gestor de paquetes Bower usando `npm` como sigue:

```text
$ npm install -g pulp bower
```

Esto dejará las herramientas de línea de comandos `pulp` y `bower` en tu ruta de ejecutables. En este punto tienes todas las herramientas necesarias para crear tu primer proyecto PureScript.

## ¡Hola, PureScript!

Empecemos simple. Usaremos Pulp para compilar y ejecutar un simple programa "Hello World!".

Comienza creando un proyecto en un directorio vacío usando el comando `pulp init`:

```text
$ mkdir my-project
$ cd my-project
$ pulp init

* Generating project skeleton in ~/my-project

$ ls

bower.json	src		test
```

Pulp ha creado por nosotros dos directorios, `src` y `test`, y un fichero de configuración `bower.json`. El directorio `src` contendrá nuestros ficheros fuente y el directorio `test` nuestras pruebas. Usaremos el directorio `test` más adelante.

Modifica el fichero `src/Main.purs` para que contenga lo siguiente:

```haskell
module Main where

import Control.Monad.Eff.Console

main = log "Hello, World!"
```

Este pequeño ejemplo ilustra unas cuantas ideas clave:

- Todos los ficheros comienzan con una cabecera de módulo. Un nombre de módulo consiste en una o más palabras comenzando por mayúsculas y separadas por puntos. En este caso hemos usado una única palabra, pero `Mi.Primer.Modulo` seria un nombre de módulo igualmente válido.
- Los módulos se importan usando su nombre completo, incluyendo los puntos que separan las partes del nombre de módulo. Aquí, importamos el módulo `Control.Monad.Eff.Console` que proporciona la función `log`.
- El programa `main` está definido como una aplicación de función. En PureScript, la aplicación de función se indica con espacio en blanco separando el nombre de la función de sus argumentos.

Ejecutemos este código usando el siguiente comando:

```text
$ pulp run

* Building project in ~/my-project
* Build successful.
Hello, World!
```

¡Enhorabuena! Acabas de compilar y ejecutar tu primer programa PureScript.

## Compilando para el navegador

Pulp puede usarse para convertir nuestro código PureScript en JavaScript susceptible de ser usado un un navegador web mediante el uso del comando `pulp browserify`:

```text
$ pulp browserify

* Browserifying project in ~/my-project
* Building project in ~/my-project
* Build successful.
* Browserifying...
```

A continuación de esto, debes ver un montón de código JavaScript impreso en la consola. Esto es la salida de la herramienta [Browserify](http://browserify.org/) aplicada a una biblioteca estándar de PureScript llamada _Prelude_, así como el código del directorio `src`. Este código JavaScript puede redirigirse a un fichero e incluirse en un documento HTML. Si lo intentas, debes ver las palabras "Hello, World!" impresas en la consola de tu navegador.

## Quitando código no usado

Pulp proporciona un comando alternativo, `pulp build`, que puede usarse con la opción `-O` para aplicar la fase de _eliminación de código muerto_ responsable de quitar JavaScript innecesario de la salida. El resultado es mucho más pequeño:

```text
$ pulp build -O --to output.js

* Building project in ~/my-project
* Build successful.
* Bundling Javascript...
* Bundled.
```

De nuevo, el código generado se puede usar en un documento HTML. Si abres `output.js`, debes ver unos cuantos módulos compilados con esta pinta:

```javascript
(function(exports) {
  "use strict";

  var Control_Monad_Eff_Console = PS["Control.Monad.Eff.Console"];

  var main = Control_Monad_Eff_Console.log("Hello, World!");
  exports["main"] = main;
})(PS["Main"] = PS["Main"] || {});
```

Esto ilustra unos cuantos puntos sobre el modo en que el compilador de PureScript genera el código JavaScript:

- Todo módulo se convierte en un objeto, creado por una función envoltorio, que contiene los miembros exportados por el módulo.
- PureScript intenta preservar los nombres de las variables cuando sea posible.
- La aplicación de funciones en PureScript se convierte en aplicación de funciones de JavaScript.
- El método principal se ejecuta después de que todos los módulos hayan sido definidos, y es generado como una simple llamada a método sin argumentos.
- El código PureScript no depende de ninguna biblioteca de tiempo de ejecución (*runtime library*). Todo el código que genera el compilador tiene su origen en un módulo PureScript del que tu código depende.

Estos puntos son importantes, ya que significan que PureScript genera código simple e inteligible. De hecho, el proceso de generación de código es una transformación bastante superficial. No es necesario un conocimiento avanzado del lenguaje para predecir qué código JavaScript será generado para cierto código de entrada.

## Compilando módulos CommonJS

Pulp también puede usarse para generar módulos CommonJS a partir de código PureScript. Esto puede ser útil cuando usemos NodeJS o cuando estemos desarrollando un proyecto grande que usa módulos CommonJS para partir el código en componentes más pequeñas.

Para construir módulos CommonJS, usa el comando `pulp build` (sin la opción `-O`):

```text
$ pulp build

* Building project in ~/my-project
* Build successful.
```

Los módulos generados serán colocados en el directorio `output` por defecto. Cada módulo PureScript se compilará a su propio módulo CommonJS en su propio subdirectorio.

## Seguimiento de dependencias con Bower

Para escribir la función `diagonal` (el objetivo de este capítulo), necesitaremos poder calcular raíces cuadradas. El paquete `purescript-math` contiene definiciones de tipos para las funciones definidas en el objeto JavaScript `Math`, así que instalémoslo:

```text
$ bower install purescript-math --save
```

La opción `--save` hace que la dependencia se añada al fichero de configuración `bower.json`.

Los fuentes de la biblioteca `purescript-math` deben estar ahora disponibles en el subdirectorio `bower_components` y serán incluidos cuando compiles tu proyecto.

## Calculando diagonales

Escribamos la función `diagonal`, que será un ejemplo de uso de una función de una biblioteca externa.

Primero importa el módulo `Math` añadiendo la siguiente línea al principio del fichero `src/Main.purs`:

```haskell
import Math (sqrt)
```

También es necesario importar el módulo `Prelude` que define operaciones muy básicas como la suma y multiplicación de números:

```haskell
import Prelude
```

Ahora define la función `diagonal` como sigue:

```haskell
diagonal w h = sqrt (w * w + h * h)
```

Date cuenta de que no es necesario definir un tipo para nuestra función. El compilador es capaz de inferir que `diagonal` es una función que toma dos números y devuelve un número. Sin embargo, en general es una buena práctica proporcionar anotaciones de tipo como una forma de documentación.

Modifiquemos la función `main` para que use la nueva función `diagonal`:

```haskell
main = logShow (diagonal 3.0 4.0)
```

Ahora compila y ejecuta el proyecto de nuevo usando `pulp run`:

```text
$ pulp run

* Building project in ~/my-project
* Build successful.
5.0
```

## Probando el código usando el modo interactivo

El compilador PureScript viene con un REPL (*Read Eval Print Loop*) interactivo llamado PSCi. Puede ser muy util para probar tu código y experimentar con nuevas ideas. Usemos PSCi para probar la función `diagonal`.

Pulp puede cargar módulos fuente en PSCi automáticamente mediante el comando `pulp repl`.

```text
$ pulp repl
>
```

Puedes escribir `:?` para ver una lista de comandos:

```text
> :?
The following commands are available:

    :?                        Show this help menu
    :quit                     Quit PSCi
    :reset                    Reset
    :browse      <module>     Browse <module>
    :type        <expr>       Show the type of <expr>
    :kind        <type>       Show the kind of <type>
    :show        import       Show imported modules
    :show        loaded       Show loaded modules
    :paste       paste        Enter multiple lines, terminated by ^D
```

Pulsando la tecla Tab puedes ver una lista de todas las funciones disponibles en tu propio código, así como las disponibles en las dependencias Bower y el módulo Prelude.

Empieza importando el módulo `Prelude`:

```text
> import Prelude
```

Intenta evaluar unas cuantas expresiones ahora:

```text
> 1 + 2
3

> "Hello, " <> "World!"
"Hello, World!"
```

Probemos ahora nuestra nueva función `diagonal` en PSCi:

```text
> import Main
> diagonal 5.0 12.0

13.0
```

También puedes usar PSCi para definir funciones:

```text
> double x = x * 2

> double 10
20
```

No te preocupes si la sintaxis de estos ejemplos no te resulta muy clara. Tendrá más sentido según vayas leyendo el libro.

Finalmente, puedes comprobar el tipo de una expresión usando el comando `:type`:

```text
> :type true
Boolean

> :type [1, 2, 3]
Array Int
```

Prueba el modo interactivo ahora. Si te atascas en algún punto, usa el comando de Reset `:reset` para descargar cualquier módulo que pueda estar compilado en memoria.

X> ## Ejercicios
X>
X> 1. (Fácil) Usa la constante `pi` definida en el módulo `Math` para escribir una función `circleArea` que calcule el área de un círculo dado su radio. Prueba tu función usando PSCi (_Pista_: no olvides importar `pi` modificando la sentencia `import Math`).
X> 1. (Medio) Usa `bower install` para instalar el paquete `purescript-globals` como una dependencia. Prueba sus funciones en PSCi (_Pista_: puedes usar el comando `:browse` en PSCi para revisar los contenidos de un módulo).

## Conclusión

En este capítulo hemos preparado un proyecto simple en PureScript usando la herramienta Pulp.

También hemos escrito nuestra primera función en PureScript y un programa JavaScript que puede ser ejecutado tanto en el navegador como en NodeJS.

Usaremos este entorno en los siguientes capítulos para compilar, depurar y probar nuestro código, así que debes asegurarte de que estás cómodo con las herramientas y técnicas involucradas.
