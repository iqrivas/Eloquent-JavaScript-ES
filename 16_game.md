{{meta {load_files: ["code/chapter/16_game.js", "code/levels.js"], zip: "html include=[\"css/game.css\"]"}}}

# Proyecto: Un Juego de Plataforma

{{quote {author: "Iain Banks", title: "El Jugador de Juegos", chapter: true}

Toda realidad es un juego.

quote}}

{{index "Banks, Ian", "project chapter", simulation}}

{{figure {url: "img/chapter_picture_16.jpg", alt: "Imagen de un personaje de juego brincando sobre lava.", chapter: "framed"}}}

Mucha de mi fascinación inicial con computadoras, como la de muchos niños nerd, tiene que ver con ((juego))s de computadora. Fui atraído a los pequeños ((mundo))s simulados que podía manipular y en los que las historias (algo así) revelaban más, yo supongo, por la manera en que yo proyectaba mi ((imaginación)) en ellas más que por las posibilidades que realmente ofrecían.

No le deseo a nadie una ((carrera)) en programación de juegos. Tal como la industria ((musical)), la discrepancia entre el número de personas jóvenes ansiosas queriendo trabajar en ella y la actual demanda de tales personas crea un ambiente bastante malsano.

{{index "jump-and-run game", dimensions}}

Este capítulo recorrerá la implementación de un ((juego de plataforma)) pequeño. Los juegos de plataforma (o juegos de "brincar y correr") son juegos que esperan que el ((jugador)) mueva una figura a través de un ((mundo)), el cual usualmente es bidimensional y visto de lado, mientras brinca encima y sobre cosas.

## El juego

{{index minimalism, "Palef, Thomas", "Dark Blue (game)"}}

Nuestro ((juego)) estará ligeramente basado en [Dark Blue](http://www.lessmilk.com/games/10)[
(_www.lessmilk.com/games/10_)]{if book} de Thomas Palef. Escogí ese juego porque es tanto entretenido como minimalista y porque puede ser construido sin demasiado ((código)). Se ve así:

{{figure {url: "img/darkblue.png", alt: "El juego Dark Blue"}}}

{{index coin, lava}}

La ((caja)) oscura representa al ((jugador)), cuya tarea es recolectar las cajas amarillas (monedas) mientras evita la cosa roja (lava). Un ((nivel)) es completado cuando todas las monedas han sido recolectadas.

{{index keyboard, jumping}}

El jugador puede caminar alrededor con las teclas de las flechas izquierda y derecha y pueden brincar con la flecha arriba. Brincar es una especilidad de este personaje de juego. Puede alcanzar varias veces su propia altura y puede cambiar de dirección en el aire. Esto puede no ser completamente realista, pero ayuda a darle al jugador el sentimiento de estar en control directo del ((avatar)).

{{index "fractional number", discretization, "artificial life", "electronic life"}}

El ((juego)) consiste en un ((fondo)) estático, expuesto como una ((cuadrícula)), con los elementos móviles superpuestos sobre ese fondo. Cada campo en la cuadrícula está ya sea vacío, sólido o ((lava)). Las posiciones de estos elementos no están restringidas a la cuadrícula-sus coordenadas pueden ser fraccionales, permitiendo un ((movimiento)) suave.

## La tecnología

{{index "event handling", keyboard, [DOM, graphics]}}

Usaremos el DOM del ((navegador)) para mostrar el juego, y leeremos la entrada del usuario por medio del manejo los eventos de teclas.

{{index rectangle, "background (CSS)", "position (CSS)", graphics}}

El código relacionado con la pantalla y el teclado es sólo una pequeña parte del trabajo que necesitamos hacer para construir este ((juego)). Ya que todo parece como ((caja))s coloreadas, dibujar es no es complicado: creamos los elementos del DOM y usamos estilos para darlos un color de fondo, tamaño y posición.

{{index "table (HTML tag)"}}

Podemos representar el fondo como una tabla ya que es una ((cuadrícula)) invariable de cuadrados. Los elementos libres de moverse pueden ser superpuestos usando elementos posicionados absolutamente.

{{index performance, [DOM, graphics]}}

En los juegos y otros programas que deben animar ((gráficos)) y responder a la ((entrada)) del usuario sin retraso notable, la ((eficiencia)) es importante. Aunque el DOM no fue diseñado originalmente para gráficos de alto rendimiento, es realmente mejor en ello de lo que se esperaría. Viste algunas ((animaciones)) en [Capítulo ?](dom#animation). En una máquina moderna, un simple juego como este desempeña bien, aún si no nos preocupamos mucho de la ((optimización)).

{{index canvas, [DOM, graphics]}}

En el [siguiente capítulo](canvas), exploraremos otra tecnología del ((navegador)), la etiqueta `<canvas>`, la cual provee un forma más tradicional de dibujar gráficos, trabajando en término de formas y ((pixel))es más que elementos del DOM.

## Niveles

{{index dimensions}}

Vamos a querer una forma de especificar niveles que sea fácilmente leíble y editable por humanos. Ya que está bien que todo empiece sobre una cuadrícula, podríamos usar cadenas de caracteres largas en las que cada caracter representa un elemento-ya sea una parte de la cuadrícula de fondo o un elemento móvil.

El plano para un nivel pequeño podría lucir como esto:

```{includeCode: true}
let simplePlanoDeNivel = `
......................
..#................#..
..#..............=.#..
..#.........o.o....#..
..#.@......#####...#..
..#####............#..
......#++++++++++++#..
......##############..
......................`;
```

{{index level}}

Los puntos son espacios vacíos, los signos de número (`#`) son muros y los signos de suma son lava. La posición inicial del ((jugador)) es el ((arroba)) (`@`). Cada caracter O es una moneda y el signo de igual (`=`) en la parte superior es un bloque de lava que se mueve de un lado a otro horizontalmente.

{{index bouncing}}

Vamos a soportar dos tipos adicionales de ((lava)) en movimento: el caracter de la barra vertical (`|`) crea gotas que mueven verticalmente y la `v` indica lava _goteando_-la lava que se mueve verticalmente que no rebota de un lado a otro sino que sólo se mueve hacia abajo, brincando a su posición inicial cuando toca el suelo.

Un ((juego)) completo consiste en múltiples ((nivel))es que el ((jugador)) debe completar. Un nivel es completado cuando todas las ((monedas))s han sido recolectadas. Si el jugador toca la ((lava)), el nivel actual se restaura a su posición inicial y el juego puede intentar de nuevo.

{{id level}}

## Leyendo un nivel

{{index "Level class"}}

La siguiente ((clase)) guarda un objeto de ((nivel)). Su argumento debe ser la cadena de carateres que define el nivel.

```{includeCode: true}
class Nivel {
    constructor(plano) {
      let filas = plano.trim().split("\n").map(l => [...l]);
      this.height = filas.length;
      this.width = filas[0].length;
      this.iniciarActores = [];
  
      this.filas = filas.map((fila, y) => {
        return fila.map((car, x) => {
          let tipo = caracteresDelNivel[car];
          if (typeof tipo == "string") return tipo;
          this.iniciarActores.push(
            tipo.create(new Vector(x, y), car));
          return "vacío";
        });
      });
    }
}
```

{{index "trim method", "split method", [whitespace, trimming]}}

El método `trim` es usado para remover el espacio en blanco al inicio y al final de la cadena de caracteres del plano. Esto permite a nuestro plano de ejemplo empezar con una nueva línea de manera que todas las líneas esten directamente una debajo de la otra. La cadena restante es dividida en ((caracteres de nueva línea)) y cada línea es propagada en un arreglo, produciendo un arreglo de caracteres.

{{index [array, "as matrix"]}}

Entonces `filas` guarda un arreglo de arreglos de caracteres, las filas del plano. Podemos derivar el ancho y alto del nivel de ellas. Pero aún debemos separar los elementos móviles de la cuadrícula de fondo. Llamaremos a los elementos móviles _actores_. Ellos se guardaran en un arreglo de objetos. El fondo será un arreglo de arreglos de caracteres, que tendrán tipos de campos como `"vacío"`, `"muro"` o `"lava"`.

{{index "map method"}}

Para crear estos arreglos, mapearemos sobre las filas y luego sobre su contenido. Recuerda que `map` pasa el índice del arreglo como un segundo argumento a la función de mapeo, que nos dice las coordenadas x y las coordenadas y de un caracter dado. Las posiciones en el juego serán guardadas como pares de coordenadas, con la izquierda superior siendo 0,0 y cada cuadrado del fondo siendo 1 unidad de alto y ancho.

{{index "static method"}}

Para interpretar los caracteres en el plano, el constructor del `Nivel` usa el objeto de `caracteresDelNivel`, el cual mapea los elementos a cadenas de caracteres y caracteres de actores a clases. Cuando `tipo` está en la clase actor, su método estático `create` es usado para crear un objeto, el cual es agregado a `iniciarActores` y la función de mapeo regresa `"vacío"` para este cuadrado de fondo.

{{index "Vec class"}}

La posición del actor es guardada como un objeto `Vector`. Este es un vector bidimensional, un objeto con propiedades `x` y `y`, como se vió en los ejercicios de [Capítulo ?](object#exercise_vector).

{{index [state, in objects]}}

Mientras el juego se ejecuta, los actores terminarán en diferentes lugares o incluso desaparecerán completamente (como las monedas lo hacen cuando son recolectadas). Usaremos una clase `Estado`
para dar seguimiento al estado de un juego que en ejecución.

```{includeCode: true}
class Estado {
    constructor(nivel, actores, estatus) {
      this.nivel = nivel;
      this.actores = actores;
      this.estatus = estatus;
    }
  
    static start(nivel) {
      return new Estado(nivel, nivel.iniciarActores, "jugando");
    }
  
    get jugador() {
      return this.actores.find(a => a.tipo == "jugador");
    }
}
```

La propiedad `estatus` cambiará a `"perdido"` or `"ganado"` cuando el juego haya terminado.

Esto es de nuevo una estructura de datos persistente-actualizar el estado del juego crea un nuevo estado y deja el anterior intacto.

## Actores

{{index actor, "Vec class", [interface, object]}}

Los objetos Actor representan la posición actual y el estado de un elemento móvil dado en nuestro juego. Todos los objetos Actor se ajustan a la misma interface. Su propiedad `posicion` tiene las coordenadas de la esquina superior izquierda del elemento y su propiedad `tamano` tiene su tamaño.

Luego tienen un método `update`, que es usado para computar su nuevo estado y posición después de un paso de tiempo. Simula lo que hace el actor -moviéndose en respuesta a las teclas de flecha para el jugador y rebotar de un lado a otro para la lava-y regresa un objeto actor nuevo y actualizado.

Una propiedad `tipo` contiene la cadena de caracteres que identifica el tipo de actor `"jugador"`, `"moneda"` o `"lava"`. Esto es útil cuando se dibuja el juego-la apariencia del rectángulo dibujado para un actor está basado en su tipo.

Las clases actor tienen un método estático `create` que es usado por el constructor de `Nivel` para crear un actor a partir de un caracter en el plano de nivel. Se le dan las coordenadas del caracter y el caracter mismo, el cuál es necesario debido a que la clase `Lava` maneja diferentes caracteres.

{{id vector}}

Esta es la clase `Vector` que usaremos para nuestros valores bidimensionales, tales como la posición y el tamaño de los actores.

```{includeCode: true}
class Vector {
    constructor(x, y) {
      this.x = x; this.y = y;
    }
    plus(otro) {
      return new Vector(this.x + otro.x, this.y + otro.y);
    }
    times(factor) {
      return new Vector(this.x * factor, this.y * factor);
    }
}
```

{{index "times method", multiplication}}

El método `times` escala un vector por un número dado. Será útil cuando necesitemos multiplicar un vector de velocidad por un intervalo de tiempo para obtener la distancia recorrida durante ese tiempo.

Los diferentes tipos de actores tienen sus propias clases ya que su comportamiento es diferente. Vamos a definir estas clases. Veremos sus métodos `update` después.

{{index simulation, "Player class"}}

La clase jugador tiene una propiedad `velocidad` que guarda su velocidad actual para simular momento y gravedad.

```{includeCode: true}
class Jugador {
    constructor(posicion, velocidad) {
      this.posicion = posicion;
      this.velocidad = velocidad;
    }
  
    get tipo() { return "jugador"; }
  
    static create(posicion) {
      return new Jugador(posicion.plus(new Vector(0, -0.5)),
                        new Vector(0, 0));
    }
}

Jugador.prototype.tamano = new Vector(0.8, 1.5);
```

Debido a que un jugador tiene una altura de un cuadrado y medio, su posición inicial está establecida a ser medio cuadrado arriba de la posición donde el caracter `@` apareció. De esta manera, su parte inferior se alinea con la parte inferior del cuadrado en el que apareció.

La propiedad `tamano` es la misma para todas las instancias de `Jugador`, así que la guardamos en el prototipo en vez de en la instancia misma. Podríamos haber usado un ((getter)) como `type`, pero eso crearía y regresaría un nuevo objeto `Vector` cada vez que la propiedad sea leída, lo cual sería desperdicio. (Las cadenas de caracteres, siendo ((inmutables)), no tienen que ser creadas de nuevo cada vez que son evaluadas.)

{{index "Lava class", bouncing}}

Al construir un actor `Lava`, necesitamos inicializar el objeto de manera diferente dependiendo del caracter en el que está basado. La lava dinámica se mueve a su velocidad actual hasta que pega con un obstáculo. En ese punto, si tiene una propiedad `reiniciar`, brincará de regreso a su posición inicial (goteando). Si no lo hace, invertirá su velocidad y continuará en otra dirección (rebotando).

El método `create` mira al caracter que el constructor de `Level` pasa y crea el actor de lava apropiado.

```{includeCode: true}
class Lava {
    constructor(posicion, velocidad, reiniciar) {
      this.posicion = posicion;
      this.velocidad = velocidad;
      this.reiniciar = reiniciar;
    }
  
    get tipo() { return "lava"; }
  
    static create(posicion, car) {
      if (car == "=") {
        return new Lava(posicion, new Vector(2, 0));
      } else if (car == "|") {
        return new Lava(posicion, new Vector(0, 2));
      } else if (car == "v") {
        return new Lava(posicion, new Vector(0, 3), posicion);
      }
    }
  }
  
Lava.prototype.tamano = new Vector(1, 1);
```

{{index "Coin class", animation}}

Los actores `Moneda` son relativamente simples. Mayormente se quedan en su lugar. Pero para avivar un poco el juego, se les da un "bamboleo", un leve movimiento vertical de arriba a abajo. Para seguir esto, un objeto moneda guarda una posición base así como una propiedad `wobble` que registra la ((fase)) del movimiento de rebote. Juntos, estos determinan la posición real de la moneda (guardada en la propiedad `posicion`).

```{includeCode: true}
class Moneda {
    constructor(posicion, posBase, bamboleo) {
      this.posicion = posicion;
      this.posBase = posBase;
      this.bamboleo = bamboleo;
    }
  
    get tipo() { return "moneda"; }
  
    static create(posicion) {
      let posBase = posicion.plus(new Vector(0.2, 0.1));
      return new Moneda(posBase, posBase,
                      Math.random() * Math.PI * 2);
    }
}
  
Moneda.prototype.tamano = new Vector(0.6, 0.6);
```

{{index "Math.random function", "random number", "Math.sin function", sine, wave}}

En el [Capítulo ?](dom#sin_cos), vimos que `Math.sin` nos da la coordenada-y de un punto en un círculo. Esa coordenada va de lado a otro en una forma de onda suave mientras nos movemos por el círculo, lo cual hace útil a la función seno para modelar movimiento ondulado.

{{index pi}}

Para evitar una situación donde todas las monedas se mueven arriba y abajo sincrónicamente, la fase inicial de cada moneda es aleatoria. La _((fase))_ de la onda de `Math.sin`, el ancho de la onda que produce, es 2π. Multiplicamos el valor retornado por `Math.random` por ese número para dar a la moneda una posición inicial aleatoria en la onda.

{{index map, [object, "as map"]}}

Ahora podemos definir el objeto `caracteresDeNivel` que mapea los caracteres del plano ya sea a cuadr{icula de fondo o clases de actores.

```{includeCode: true}
const caracteresDelNivel = {
    ".": "vacío", "#": "muro", "+": "lava",
    "@": Jugador, "o": Moneda,
    "=": Lava, "|": Lava, "v": Lava
};
```

Eso nos da todas las partes que necesitamos para crear una instancia de `Nivel`.

```{includeCode: strip_log}
let simpleNivel = new Nivel(simplePlanoDeNivel);
console.log(`${simpleNivel.width} by ${simpleNivel.height}`);
// → 22 by 9
```

La tarea adelante es mostrar dichos niveles en la pantalla y modelar tiempo y movimiento dentro de ellos.

## Encapsulación como una carga

{{index "programming style", "program size", complexity}}

La mayoría del código en este capítulo no se preocupa mucho de la ((encapsulación)) por dos razones. Primero, encapsular necesita esfuerzo extra. Hace a los programas más grandes y requiere que conceptos e interfaces adicionales sean introducidos. Dado que sólo hay una cantidad de código que le puedes mostrar a un lector antes de que sus ojos pierdan la atención, he hecho un esfuerzo por mantener el programa pequeño.

{{index [interface, design]}}

Segundo, los elementos varios en este juego están tan estrechamente vinculados que si el comportamiento de uno de ellos cambia, es poco probable que alguno de los otros logre quedarse igual. Las interfaces entre los elementos terminarían codificando muchas de los supuestos acerca de la forma que el juego funciona. Esto las hace mucho menos efectivas-cuando sea que cambies una parte del sistema, todavía tienes que precouparte de la forma en que impacta a las otras partes porque sus interfaces no cubrirían la nueva situación.

Algunos _((puntos de corte))_ en un sistema se prestan bien a la separacion mediante interfaces rigurosas, pero otros no. Intentando encapsular algo que no es un límite adecuado es una manera segura de deserdiciar mucha energía. Cuando cometes este error, te darás cuenta que tus interfaces se hacen incómodamente grandes y detalladas y que necesitan cambiarse frecuentemente, mientras el programa evoluciona.

{{index graphics, encapsulation, graphics}}

Hay una cosa que sí _vamos_ a encapsular, y eso es el subsistema de ((dibujo)). La razón para esto es que ((mostraremos)) el mismo juego en una manera diferente en el [próximo capítulo](canvas#canvasdisplay). Poniendo el dibujo detrás de una interface, podemos cargar el mismo programa de juego ahí y conectar un nuevo ((módulo)) de pantalla.

{{id domdisplay}}

## Dibujar

{{index "DOMDisplay class", [DOM, graphics]}}

La encapsulación del código de ((dibujo)) se hace definiendo un objeto de _((visualización))_,que muestra un ((nivel)) y estado dado. El tipo de visualización que definimos en este capítulo se llama `DOMDisplay` porque usa elemento del DOM para mostrar el nivel.

{{index "style attribute", CSS}}

Estaremos usando una hoja de estilos para definir los colores y otras propiedades fijas de los elementos que conforman el juego. También sería posible asignar directamente a los elementos la propiedad `style` cuando los creemos, pero eso produciría programas más verbosos.

{{index "class attribute"}}

La siguiente función auxiliar provee una forma precisa de crear un elemento y darle algunos atributos y nodos hijos:

```{includeCode: true}
function elt(name, attrs, ...children) {
    let dom = document.createElement(name);
    for (let attr of Object.keys(attrs)) {
      dom.setAttribute(attr, attrs[attr]);
    }
    for (let child of children) {
      dom.appendChild(child);
    }
    return dom;
}
```

Una visualización es creada dándole un elemento padre al cual debe agregarse y un objeto de ((nivel)).

```{includeCode: true}
class DOMDisplay {
    constructor(padre, nivel) {
      this.dom = elt("div", {class: "game"}, dibujarCuadricula(nivel));
      this.capaActor = null;
      padre.appendChild(this.dom);
    }
  
    clear() { this.dom.remove(); }
}
```

{{index level}}

El ((fondo)) del nivel, que nunca cambia, es dibujado una sola vez. Los actores son redibujados cada vez que la visualización es actualizada con un estado dado. La propiedad `capaActor` será usada para seguir al elemento que tiene a los actores para que puedan ser fácilmente removidos y reemplazados.

{{index scaling, "DOMDisplay class"}}

Nuestras ((coordenadas)) y tamaños son registrados en unidades de ((cuadrícula)), donde un tamaño o distancia de 1 significa un bloque de cuadrícula. Cuando se establezcan los tamaños de ((pixel)), tendremos que escalar estas coordenadas hacia arriba -todo en el juego sería ridiculamente pequeño a un sólo pixel por cuadrado. La constante `escala` da el número de pixeles que una sola unidad toma en la pantalla.

```{includeCode: true}
const escala = 20;

function dibujarCuadricula(nivel) {
  return elt("table", {
    class: "fondo",
    style: `width: ${nivel.width * escala}px`
  }, ...nivel.filas.map(fila =>
    elt("tr", {style: `height: ${escala}px`},
        ...fila.map(tipo => elt("td", {class: tipo})))
  ));
}
```

{{index "table (HTML tag)", "tr (HTML tag)", "td (HTML tag)", "spread operator"}}

Como mencionamos, el fondo es dibujado como un elemento `<table>`. Esto corresponde con la a la estructura de la propiedad `filas` del nivel -cada fila de la cuadrícula se convierte en una fila de la tabla ( elemento `<tr>`). Las cadenas de caracteres en la cuadrícula son usadas como nombres de clases para los elementos celdas de tabla (`<td>`). El operador de propagación (tres puntos) se usa para pasar arreglos de nodos hijos a `elt` como argumentos separados.

{{id game_css}}

El siguiente ((CSS)) hace que la tabla luzca como el fondo que queremos:

```{lang: "text/css"}
.fondo    { background: rgb(52, 166, 251);
                 table-layout: fixed;
                 border-spacing: 0;              }
.fondo td { padding: 0;                     }
.lava          { background: rgb(255, 100, 100); }
.muro          { background: white;              }
```

{{index "padding (CSS)"}}

Algunos de estos (`table-layout`, `border-spacing`, and `padding`) son usados para suprimir comportamientos predeterminados no deseados. No queremos que el layout de la ((tabla)) dependa de los contenidos de sus celdas, y no queremos espacio entre las celdas de la ((tabla)) o padding dentro de ellas.

{{index "background (CSS)", "rgb (CSS)", CSS}}

La regla `background` define el color del fondo. El CSS permite que los colores sean específicados tanto como palabras (`white`) o con un formato como `rgb(R, G, B)`, donde los componentes rojo, verde y azul del color son separados en tres números de 0 a 255. Entonces, en `rgb(52, 166, 251)`, el componente rojo es 52, el verde es 166 y el azul es 251. Dado que el componente azul es el más grande, el color resultante será azulado. Puedes ver que en la regla `.lava`, el primer número (rojo) es el más grande.

{{index [DOM, graphics]}}

Dibujamos cada ((actor)) creándole un elemento del DOM y definiendo la posición y tamaño de ese elemento basados en las propiedades del actor. Los valores tienen que ser multiplicados por la `escala` que van de unidades de juego a pixeles.

```{includeCode: true}
function dibujarActores(actores) {
    return elt("div", {}, ...actores.map(actor => {
      let rect = elt("div", {class: `actor ${actor.tipo}`});
      rect.style.width = `${actor.tamano.x * escala}px`;
      rect.style.height = `${actor.tamano.y * escala}px`;
      rect.style.left = `${actor.posicion.x * escala}px`;
      rect.style.top = `${actor.posicion.y * escala}px`;
      return rect;
    }));
}
```

{{index "position (CSS)", "class attribute"}}

Para darle más de una clase a un elemento, separamos los nombres de clase por espacios. En el código ((CSS)) que se muestra a continuación, la clase `actor` da a los actores su posición absoluta. Su nombre de tipo es usado como una clase extra para darles un color. No tenemos que definir la clase `lava` de nuevo porque estamos reusando la clase para los cuadrados de lava en la cuadrícula que definimos anteriormente.

```{lang: "text/css"}
.actor  { position: absolute;            }
.moneda   { background: rgb(241, 229, 89); }
.jugador { background: rgb(64, 64, 64);   }
```

{{index graphics, optimization, efficiency, [state, "of application"], [DOM, graphics]}}

El método `syncState` es usado para hacer que la pantalla muestre un estado dado. Primero quita los gráficos de los viejos actores, si hay alguno, y luego redibuja a los actores en sus nuevas posiciones. Puede ser tentador intentar reusar los elementos del DOM para los actores, pero para hacer que eso funcione, necesitaríamos muchos registros adicionales para asociar a los actores con los elementos del DOM y para asegurarse que quitemmos elementos cuando sus actores se desvanezcan. Ya que típicamente sólo tendremos un puñado de actores en el juego, redibujarlos no es costoso.

```{includeCode: true}
DOMDisplay.prototype.syncState = function(estado) {
  if (this.capaActor) this.capaActor.remove();
  this.capaActor = dibujarActores(estado.actores);
  this.dom.appendChild(this.capaActor);
  this.dom.className = `game ${estado.estatus}`;
  this.scrollPlayerIntoView(estado);
};
```

{{index level, "class attribute"}}

Añadiendo el estatus del nivel actual como un nombre de clase al wrapper, podemos estilizar al actor jugador ligeramente diferente cuando el juego es ganado o perdido añadiendo un regla de ((CSS)) que toma efecto sólo cuando el jugador tiene un  ((elemento ancestro)) con una clase dada.

```{lang: "text/css"}
.perdido .jugador {
  background: rgb(160, 64, 64);
}
.ganado .jugador {
  box-shadow: -4px -7px 8px white, 4px -7px 8px white;
}
```

{{index player, "box shadow (CSS)"}}

Después de tocar la ((lava)), el color del jugador se vuelve rojo oscuro, sugiriendo que se quema. Cuando la última moneda ha sido recolectada, añadimos dos sombras blancas difuminadas —una arriba a la izquierda y una arriba a la derecha -para crear un efecto blanco de aura.

{{id viewport}}

{{index "position (CSS)", "max-width (CSS)", "overflow (CSS)", "max-height (CSS)", viewport, scrolling, [DOM, graphics]}}

No podemos asumir que el nivel siempre encaja en el _viewport_—el elemento en el que dibujamos el juego. Es por esto que se requiere la llamada a `scrollPlayerIntoView`. Esta asegura que si el nivel sobresale fuera del viewport, desplazaremos ese viewport para asegurarnos que el jugador esté cerca de su centro. El siguiente ((CSS)) da al elemento del DOM que envuelve el juego un tamaño máximo y se asegura que cualquier cosa que salga de la caja del elemento no sea visible. También le damos una posición relativa para que el actor que está dentro esté posicionado relativo a la esquina superior izquierda del nivel.

```{lang: "text/css"}
.juego {
  overflow: hidden;
  max-width: 600px;
  max-height: 450px;
  position: relative;
}
```

{{index scrolling}}

En el método `scrollPlayerIntoView`, encontramos la posición del jugador y actualizamos la posición de desplazamiento del elemento que envuelve. Cambiamos la posición de desplazamiento manipulando las propiedades `scrollLeft` y `scrollTop` del elemento cuando el jugador está muy cerca del borde.

```{includeCode: true}
DOMDisplay.prototype.scrollPlayerIntoView = function(estado) {
  let ancho = this.dom.clientWidth;
  let alto = this.dom.clientHeight;
  let margen = ancho / 3;

  // The viewport
  let izquierda = this.dom.scrollLeft, derecha = izquierda + ancho;
  let arriba = this.dom.scrollTop, abajo = arriba + alto;

  let jugador = estado.jugador;
  let centro = jugador.posicion.plus(jugador.tamano.times(0.5))
                         .times(escala);

  if (centro.x < izquierda + margen) {
    this.dom.scrollLeft = centro.x - margen;
  } else if (centro.x > derecha - margen) {
    this.dom.scrollLeft = centro.x + margen - ancho;
  }
  if (centro.y < arriba + margen) {
    this.dom.scrollTop = centro.y - margen;
  } else if (centro.y > abajo - margen) {
    this.dom.scrollTop = centro.y + margen - alto;
  }
};
```

{{index center, coordinates, readability}}

La manera en que se encuentra el centro del jugador muestra como los métodos en nuestro tipo `Vector` permiten que las computaciones con objetos sean escritas de una manera relativamente leíble. Para encontrar el centro del jugador, añadimos su posición (su esquina superior izquierda) y la mitad de su tamaño. Ese es el centro en las coordenadas de nivel, pero necesitamos en coordenadas pixel, para que podamos multiplicar el vector resultante por nuestra escala.

{{index validation}}

A continuación, una serie de validaciones verifican que la posición del jugador no esté fuera del rango permitido. Hay que notar que algunas veces esto fijará coordenadas de desplazamiento sin sentido que están debajo de cero o más allá del área de desplazamiento del elemento. Esto está bien —el DOM los limitará a valores aceptables. Fijar `scrollLeft` a -10 causará que se convierta en 0.

Hubiese sido ligeramente más sencillo siempre tratar de desplazar al jugador al centro del ((viewport)). Pero esto crea un efecto discordante. Mientras brincas, la vista constantemente cambiará arriba y abajo. Es más agradable tener un area "neutral" en el medio de la pantalla donde puedas moverte alrededor sin causar ningún desplazamiento.

{{index [game, screenshot]}}

Ahora podemos mostrar nuestro pequeño nivel.

```{lang: "text/html"}
<link rel="stylesheet" href="css/game.css">

<script>
  let simpleNivel = new Nivel(simplePlanoDeNivel);
  let display = new DOMDisplay(document.body, simpleNivel);
  display.syncState(Estado.start(simpleNivel));
</script>
```

{{if book

{{figure {url: "img/game_simpleLevel.png", alt: "Our level rendered",width: "7cm"}}}

if}}

{{index "link (HTML tag)", CSS}}

La etiqueta `<link>`, cuando se usa con `rel="stylesheet"`, es una forma de cargar un archivo de CSS en una página. El archivo `juego.css` contiene los estilos necesarios para nuestro juego. 

## Motion and collision

{{index physics, [animation, "platform game"]}}

Now we're at the point where we can start adding motion—the most
interesting aspect of the game. The basic approach, taken by most
games like this, is to split ((time)) into small steps and, for each
step, move the actors by a distance corresponding to their speed
multiplied by the size of the time step. We'll measure time in
seconds, so speeds are expressed in units per second.

{{index obstacle, "collision detection"}}

Moving things is easy. The difficult part is dealing with the
interactions between the elements. When the player hits a wall or
floor, they should not simply move through it. The game must notice
when a given motion causes an object to hit another object and respond
accordingly. For walls, the motion must be stopped. When hitting a
coin, it must be collected. When touching lava, the game should be lost.

Solving this for the general case is a big task. You can find
libraries, usually called _((physics engine))s_, that simulate
interaction between physical objects in two or three ((dimensions)).
We'll take a more modest approach in this chapter, handling only
collisions between rectangular objects and handling them in a rather
simplistic way.

{{index bouncing, "collision detection", [animation, "platform game"]}}

Before moving the ((player)) or a block of ((lava)), we test whether
the motion would take it inside of a wall. If it does, we simply
cancel the motion altogether. The response to such a collision depends
on the type of actor—the player will stop, whereas a lava block will
bounce back.

{{index discretization}}

This approach requires our ((time)) steps to be rather small since it
will cause motion to stop before the objects actually touch. If the
time steps (and thus the motion steps) are too big, the player would
end up hovering a noticeable distance above the ground. Another
approach, arguably better but more complicated, would be to find the
exact collision spot and move there. We will take the simple approach
and hide its problems by ensuring the animation proceeds in small
steps.

{{index obstacle, "touches method", "collision detection"}}

{{id touches}}

This method tells us whether a ((rectangle)) (specified by a position
and a size) touches a grid element of the given type.

```{includeCode: true}
Level.prototype.touches = function(pos, size, type) {
  var xStart = Math.floor(pos.x);
  var xEnd = Math.ceil(pos.x + size.x);
  var yStart = Math.floor(pos.y);
  var yEnd = Math.ceil(pos.y + size.y);

  for (var y = yStart; y < yEnd; y++) {
    for (var x = xStart; x < xEnd; x++) {
      let isOutside = x < 0 || x >= this.width ||
                      y < 0 || y >= this.height;
      let here = isOutside ? "wall" : this.rows[y][x];
      if (here == type) return true;
    }
  }
  return false;
};
```

{{index "Math.floor function", "Math.ceil function"}}

The method computes the set of grid squares that the body ((overlap))s
with by using `Math.floor` and `Math.ceil` on its ((coordinates)).
Remember that ((grid)) squares are 1 by 1 units in size. By
((rounding)) the sides of a box up and down, we get the range of
((background)) squares that the box touches.

{{figure {url: "img/game-grid.svg", alt: "Finding collisions on a grid",width: "3cm"}}}

We loop over the block of ((grid)) squares found by ((rounding)) the
((coordinates)) and return `true` when a matching square is found.
Squares outside of the level are always treated as `"wall"` to ensure
that the player can't leave the world and that we won't accidentally
try to read outside of the bounds of our `rows` array.

The state `update` method uses `touches` to figure out whether the player
is touching lava.

```{includeCode: true}
State.prototype.update = function(time, keys) {
  let actors = this.actors
    .map(actor => actor.update(time, this, keys));
  let newState = new State(this.level, actors, this.status);

  if (newState.status != "playing") return newState;

  let player = newState.player;
  if (this.level.touches(player.pos, player.size, "lava")) {
    return new State(this.level, actors, "lost");
  }

  for (let actor of actors) {
    if (actor != player && overlap(actor, player)) {
      newState = actor.collide(newState);
    }
  }
  return newState;
};
```

The method is passed a time step and a data structure that tells it which keys
are being held down. The first thing it does is call the `update`
method on all actors, producing an array of updated actors. The actors
also get the time step, the keys, and the state, so that they can base
their update on those. Only the player will actually read keys, since
that's the only actor that's controlled by the keyboard.

If the game is already over, no further processing has to be done (the
game can't be won after being lost, or vice versa). Otherwise, the
method tests whether the player is touching background lava. If so,
the game is lost, and we're done. Finally, if the game really is still
going on, it sees whether any other actors overlap the player.

Overlap between actors is detected with the `overlap` function. It
takes two actor objects and returns true when they touch—which is the
case when they overlap both along the x-axis and along the y-axis.

```{includeCode: true}
function overlap(actor1, actor2) {
  return actor1.pos.x + actor1.size.x > actor2.pos.x &&
         actor1.pos.x < actor2.pos.x + actor2.size.x &&
         actor1.pos.y + actor1.size.y > actor2.pos.y &&
         actor1.pos.y < actor2.pos.y + actor2.size.y;
}
```

If any actor does overlap, its `collide` method gets a chance to
update the state. Touching a lava actor sets the game status to
`"lost"`. Coins vanish when you touch them and set the status to
`"won"` when they are the last coin of the level.

```{includeCode: true}
Lava.prototype.collide = function(state) {
  return new State(state.level, state.actors, "lost");
};

Coin.prototype.collide = function(state) {
  let filtered = state.actors.filter(a => a != this);
  let status = state.status;
  if (!filtered.some(a => a.type == "coin")) status = "won";
  return new State(state.level, filtered, status);
};
```

{{id actors}}

## Actor updates

{{index actor, "Lava class", lava}}

Actor objects' `update` methods take as arguments the time step, the
state object, and a `keys` object. The one for the `Lava` actor type
ignores the `keys` object.

```{includeCode: true}
Lava.prototype.update = function(time, state) {
  let newPos = this.pos.plus(this.speed.times(time));
  if (!state.level.touches(newPos, this.size, "wall")) {
    return new Lava(newPos, this.speed, this.reset);
  } else if (this.reset) {
    return new Lava(this.reset, this.speed, this.reset);
  } else {
    return new Lava(this.pos, this.speed.times(-1));
  }
};
```

{{index bouncing, multiplication, "Vec class", "collision detection"}}

This `update` method computes a new position by adding the product of the ((time)) step
and the current speed to its old position. If no obstacle blocks that
new position, it moves there. If there is an obstacle, the behavior
depends on the type of the ((lava)) block—dripping lava has a `reset`
position, to which it jumps back when it hits something. Bouncing lava
inverts its speed by multiplying it by -1 so that it starts moving in
the opposite direction.

{{index "Coin class", coin, wave}}

Coins use their `update` method to wobble. They ignore collisions with
the grid since they are simply wobbling around inside of their own
square.

```{includeCode: true}
const wobbleSpeed = 8, wobbleDist = 0.07;

Coin.prototype.update = function(time) {
  let wobble = this.wobble + time * wobbleSpeed;
  let wobblePos = Math.sin(wobble) * wobbleDist;
  return new Coin(this.basePos.plus(new Vector(0, wobblePos)),
                  this.basePos, wobble);
};
```

{{index "Math.sin function", sine, phase}}

The `wobble` property is incremented to track time and then used as an
argument to `Math.sin` to find the new position on the ((wave)). The
coin's current position is then computed from its base position and an
offset based on this wave.

{{index "collision detection", "Player class"}}

That leaves the ((player)) itself. Player motion is handled separately
per ((axis)) because hitting the floor should not prevent horizontal
motion, and hitting a wall should not stop falling or jumping motion.

```{includeCode: true}
const playerXSpeed = 7;
const gravity = 30;
const jumpSpeed = 17;

Player.prototype.update = function(time, state, keys) {
  let xSpeed = 0;
  if (keys.ArrowLeft) xSpeed -= playerXSpeed;
  if (keys.ArrowRight) xSpeed += playerXSpeed;
  let pos = this.pos;
  let movedX = pos.plus(new Vector(xSpeed * time, 0));
  if (!state.level.touches(movedX, this.size, "wall")) {
    pos = movedX;
  }

  let ySpeed = this.speed.y + time * gravity;
  let movedY = pos.plus(new Vector(0, ySpeed * time));
  if (!state.level.touches(movedY, this.size, "wall")) {
    pos = movedY;
  } else if (keys.ArrowUp && ySpeed > 0) {
    ySpeed = -jumpSpeed;
  } else {
    ySpeed = 0;
  }
  return new Player(pos, new Vector(xSpeed, ySpeed));
};
```

{{index [animation, "platform game"], keyboard}}

The horizontal motion is computed based on the state of the left and
right arrow keys. When there's no wall blocking the new position
created by this motion, it is used. Otherwise, the old position is
kept.

{{index acceleration, physics}}

Vertical motion works in a similar way but has to simulate ((jumping))
and ((gravity)). The player's vertical speed (`ySpeed`) is first
accelerated to account for ((gravity)).

{{index "collision detection", keyboard, jumping}}

We check for walls again. If we don't hit any, the new position is
used. If there _is_ a wall, there are two possible outcomes. When the
up arrow is pressed _and_ we are moving down (meaning the thing we hit
is below us), the speed is set to a relatively large, negative value.
This causes the player to jump. If that is not the case, the player
simply bumped into something, and the speed is set to zero.

The gravity strength, ((jumping)) speed, and pretty much all other
((constant))s in this game have been set by ((trial and error)). I
tested values until I found a combination I liked.

## Tracking keys

{{index keyboard}}

For a ((game)) like this, we do not want keys to take effect once per
keypress. Rather, we want their effect (moving the player figure) to
stay active as long as they are held.

{{index "preventDefault method"}}

We need to set up a key handler that stores the current state of the
left, right, and up arrow keys. We will also want to call
`preventDefault` for those keys so that they don't end up
((scrolling)) the page.

{{index "trackKeys function", "key code", "event handling", "addEventListener method"}}

The following function, when given an array of key names, will return
an object that tracks the current position of those keys. It registers
event handlers for `"keydown"` and `"keyup"` events and, when the key
code in the event is present in the set of codes that it is tracking,
updates the object.

```{includeCode: true}
function trackKeys(keys) {
  let down = Object.create(null);
  function track(event) {
    if (keys.includes(event.key)) {
      down[event.key] = event.type == "keydown";
      event.preventDefault();
    }
  }
  window.addEventListener("keydown", track);
  window.addEventListener("keyup", track);
  return down;
}

const arrowKeys =
  trackKeys(["ArrowLeft", "ArrowRight", "ArrowUp"]);
```

{{index "keydown event", "keyup event"}}

The same handler function is used for both event types. It looks at
the event object's `type` property to determine whether the key state
should be updated to true (`"keydown"`) or false (`"keyup"`).

{{id runAnimation}}

## Running the game

{{index "requestAnimationFrame function", [animation, "platform game"]}}

The `requestAnimationFrame` function, which we saw in [Chapter
?](dom#animationFrame), provides a good way to animate a game. But its
interface is quite primitive—using it requires us to track the time at
which our function was called the last time around and call
`requestAnimationFrame` again after every frame.

{{index "runAnimation function", "callback function", [function, "as value"], [function, "higher-order"], [animation, "platform game"]}}

Let's define a helper function that wraps those boring parts in a
convenient interface and allows us to simply call `runAnimation`,
giving it a function that expects a time difference as an argument and
draws a single frame. When the frame function returns the value
`false`, the animation stops.

```{includeCode: true}
function runAnimation(frameFunc) {
  let lastTime = null;
  function frame(time) {
    if (lastTime != null) {
      let timeStep = Math.min(time - lastTime, 100) / 1000;
      if (frameFunc(timeStep) === false) return;
    }
    lastTime = time;
    requestAnimationFrame(frame);
  }
  requestAnimationFrame(frame);
}
```

{{index time, discretization}}

I have set a maximum frame step of 100 milliseconds (one-tenth of a
second). When the browser tab or window with our page is hidden,
`requestAnimationFrame` calls will be suspended until the tab or
window is shown again. In this case, the difference between `lastTime`
and `time` will be the entire time in which the page was hidden.
Advancing the game by that much in a single step would look silly and
might cause weird side effects, such as the player falling through the
floor.

The function also converts the time steps to seconds, which are an
easier quantity to think about than milliseconds.

{{index "callback function", "runLevel function", [animation, "platform game"]}}

The `runLevel` function takes a `Level` object and a ((display))
constructor and returns a promise. It displays the level (in
`document.body`) and lets the user play through it. When the level is
finished (lost or won), `runLevel` waits one more second (to let the
user see what happens) and then clears the display, stops the
animation, and resolves the promise to the game's end status.

```{includeCode: true}
function runLevel(level, Display) {
  let display = new Display(document.body, level);
  let state = State.start(level);
  let ending = 1;
  return new Promise(resolve => {
    runAnimation(time => {
      state = state.update(time, arrowKeys);
      display.syncState(state);
      if (state.status == "playing") {
        return true;
      } else if (ending > 0) {
        ending -= time;
        return true;
      } else {
        display.clear();
        resolve(state.status);
        return false;
      }
    });
  });
}
```

{{index "runGame function"}}

A game is a sequence of ((level))s. Whenever the ((player)) dies, the
current level is restarted. When a level is completed, we move on to
the next level. This can be expressed by the following function, which
takes an array of level plans (strings) and a ((display)) constructor:

```{includeCode: true}
async function runGame(plans, Display) {
  for (let level = 0; level < plans.length;) {
    let status = await runLevel(new Level(plans[level]),
                                Display);
    if (status == "won") level++;
  }
  console.log("You've won!");
}
```

{{index "asynchronous programming", "event handling"}}

Because we made `runLevel` return a promise, `runGame` can be written
using an `async` function, as shown in [Chapter ?](async). It returns
another promise, which resolves when the player finishes the game.

{{index game, "GAME_LEVELS data set"}}

There is a set of ((level)) plans available in the `GAME_LEVELS`
binding in [this chapter's
sandbox](https://eloquentjavascript.net/code#16)[
([_https://eloquentjavascript.net/code#16_](https://eloquentjavascript.net/code#16))]{if
book}. This page feeds them to `runGame`, starting an actual game.

```{sandbox: null, focus: yes, lang: "text/html", startCode: true}
<link rel="stylesheet" href="css/game.css">

<body>
  <script>
    runGame(GAME_LEVELS, DOMDisplay);
  </script>
</body>
```

{{if interactive

See if you can beat those. I had quite a lot of fun building them.

if}}

## Exercises

### Game over

{{index "lives (exercise)", game}}

It's traditional for ((platform game))s to have the player start with
a limited number of _lives_ and subtract one life each time they die.
When the player is out of lives, the game restarts from the beginning.

{{index "runGame function"}}

Adjust `runGame` to implement lives. Have the player start with three.
Output the current number of lives (using `console.log`) every time a
level starts.

{{if interactive

```{lang: "text/html", test: no, focus: yes}
<link rel="stylesheet" href="css/game.css">

<body>
<script>
  // The old runGame function. Modify it...
  async function runGame(plans, Display) {
    for (let level = 0; level < plans.length;) {
      let status = await runLevel(new Level(plans[level]),
                                  Display);
      if (status == "won") level++;
    }
    console.log("You've won!");
  }
  runGame(GAME_LEVELS, DOMDisplay);
</script>
</body>
```

if}}

### Pausing the game

{{index "pausing (exercise)", "escape key", keyboard}}

Make it possible to pause (suspend) and unpause the game by pressing
the Esc key.

{{index "runLevel function", "event handling"}}

This can be done by changing the `runLevel` function to use another
keyboard event handler and interrupting or resuming the animation
whenever the Esc key is hit.

{{index "runAnimation function"}}

The `runAnimation` interface may not look like it is suitable for this
at first glance, but it is if you rearrange the way `runLevel` calls
it.

{{index [binding, global], "trackKeys function"}}

When you have that working, there is something else you could try. The
way we have been registering keyboard event handlers is somewhat
problematic. The `arrowKeys` object is currently a global binding, and
its event handlers are kept around even when no game is running. You
could say they _((leak))_ out of our system. Extend `trackKeys` to
provide a way to unregister its handlers and then change `runLevel`
to register its handlers when it starts and unregister them again when
it is finished.

{{if interactive

```{lang: "text/html", focus: yes, test: no}
<link rel="stylesheet" href="css/game.css">

<body>
<script>
  // The old runLevel function. Modify this...
  function runLevel(level, Display) {
    let display = new Display(document.body, level);
    let state = State.start(level);
    let ending = 1;
    return new Promise(resolve => {
      runAnimation(time => {
        state = state.update(time, arrowKeys);
        display.syncState(state);
        if (state.status == "playing") {
          return true;
        } else if (ending > 0) {
          ending -= time;
          return true;
        } else {
          display.clear();
          resolve(state.status);
          return false;
        }
      });
    });
  }
  runGame(GAME_LEVELS, DOMDisplay);
</script>
</body>
```

if}}

{{hint

{{index "pausing (exercise)", [animation, "platform game"]}}

An animation can be interrupted by returning `false` from the
function given to `runAnimation`. It can be continued by calling
`runAnimation` again.

{{index closure}}

So we need to communicate the fact that we are pausing the game to the
function given to `runAnimation`. For that, you can use a binding that
both the event handler and that function have access to.

{{index "event handling", "removeEventListener method", [function, "as value"]}}

When finding a way to unregister the handlers registered by
`trackKeys`, remember that the _exact_ same function value that was
passed to `addEventListener` must be passed to `removeEventListener`
to successfully remove a handler. Thus, the `handler` function value
created in `trackKeys` must be available to the code that unregisters
the handlers.

You can add a property to the object returned by `trackKeys`,
containing either that function value or a method that handles the
unregistering directly.

hint}}

### A monster

{{index "monster (exercise)"}}

It is traditional for platform games to have enemies that you can jump
on top of to defeat. This exercise asks you to add such an actor type
to the game.

We'll call it a monster. Monsters move only horizontally. You can make
them move in the direction of the player, bounce back and forth
like horizontal lava, or have any movement pattern you want. The class
doesn't have to handle falling, but it should make sure the monster
doesn't walk through walls.

When a monster touches the player, the effect depends on whether the
player is jumping on top of them or not. You can approximate this by
checking whether the player's bottom is near the monster's top. If
this is the case, the monster disappears. If not, the game is lost.

{{if interactive

```{test: no, lang: "text/html", focus: yes}
<link rel="stylesheet" href="css/game.css">
<style>.monster { background: purple }</style>

<body>
  <script>
    // Complete the constructor, update, and collide methods
    class Monster {
      constructor(pos, /* ... */) {}

      get type() { return "monster"; }

      static create(pos) {
        return new Monster(pos.plus(new Vector(0, -1)));
      }

      update(time, state) {}

      collide(state) {}
    }

    Monster.prototype.size = new Vector(1.2, 2);

    levelChars["M"] = Monster;

    runLevel(new Level(`
..................................
.################################.
.#..............................#.
.#..............................#.
.#..............................#.
.#...........................o..#.
.#..@...........................#.
.##########..............########.
..........#..o..o..o..o..#........
..........#...........M..#........
..........################........
..................................
`), DOMDisplay);
  </script>
</body>
```

if}}

{{hint

{{index "monster (exercise)", "persistent data structure"}}

If you want to implement a type of motion that is stateful, such as
bouncing, make sure you store the necessary state in the actor
object—include it as constructor argument and add it as a property.

Remember that `update` returns a _new_ object, rather than changing
the old one.

{{index "collision detection"}}

When handling collision, find the player in `state.actors` and
compare its position to the monster's position. To get the _bottom_ of
the player, you have to add its vertical size to its vertical
position. The creation of an updated state will resemble either
`Coin`'s `collide` method (removing the actor) or `Lava`'s (changing
the status to `"lost"`), depending on the player position.

hint}}
