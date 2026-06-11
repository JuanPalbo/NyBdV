# NyBdV — Documentación técnica

Novela visual construida sobre **Twine / Harlowe 3.3.9**. El motor de VN es código propio que vive en el **Story JavaScript** y el **Story Stylesheet** de la historia; todo lo demás es Harlowe estándar.

---

## Archivos y entorno

| Archivo | Rol |
|---|---|
| `Stories/NyBdV.html` | Historia publicada desde Twine — **es la fuente real**. Twine la abre y la sirve. |
| `Stories/NyBdV.twee` | Fuente `.twee` exportada — **está desactualizada** respecto al HTML. El HTML manda. |
| `Stories/NyBdV.html.bak` … `.bak5` | Puntos de restauración de etapas anteriores. |
| `Stories/serve.py` | Servidor HTTP local multi-thread para desarrollo. |
| `Stories/Imagenes/` | Copia local de los assets (fondos y sprites). |
| `Imagenes/` | Assets del repo, servidos por CDN. |

---

## Servidor local (`serve.py`)

```python
from http.server import ThreadingHTTPServer, SimpleHTTPRequestHandler
import functools
H = functools.partial(SimpleHTTPRequestHandler, directory=r'C:\Users\juanc\Documents\Twine\Stories')
print("Servidor iniciado en http://localhost:8000")
ThreadingHTTPServer(('', 8000), H).serve_forever()
```

**Por qué `ThreadingHTTPServer` y no `HTTPServer`:** el servidor single-thread de Python se cuelga en deadlock cuando el navegador mantiene conexiones keep-alive abiertas (bloquean la carga de imágenes y los screenshots). La versión multi-thread resuelve el problema.

Levantar el servidor:
```
python serve.py
```
URL del juego: `http://localhost:8000/NyBdV.html`

> Hay que volver a levantarlo después de cada reinicio del equipo.

---

## Imágenes y `BASE`

La constante `BASE` en el Story JavaScript define de dónde se cargan todos los assets:

```js
var BASE = "https://cdn.jsdelivr.net/gh/JuanPalbo/NyBdV@main/Imagenes/";
// Para usar local: var BASE = "Imagenes/";
```

- **CDN (jsDelivr):** producción y uso habitual. Sin dependencia del servidor local.
- **Local (`"Imagenes/"`):** útil para probar assets nuevos antes de hacer push. Requiere el servidor local levantado.

### Lista de preload (`PRELOAD`)

Array declarado junto a `BASE`. Listar aquí **todos** los archivos que usa el juego para que el preloader los cachée antes de mostrar el primer pasaje:

```js
var PRELOAD = [
    "CentralElectrica.jpg",
    "FondoLiving.jpg",
    "FondoCuartoGabriel.jpg",
    "FondoCuartoGabrielNoche.png",
    "FondoCafe.png",
    "FondoCafeTarde.png",
    "Gabriel.png",
    "Martina.png",
    "MartinaEsceptica.png",
    "MartinaEyeroll.png",
    "Nico.png"
];
```

Si se agrega un asset nuevo al juego, hay que añadirlo aquí.

---

## Story JavaScript — módulos

Todo el código vive en una IIFE `(function () { ... })()` que se ejecuta una sola vez al cargar la página. Usa `var $ = window.jQuery` para acceder a jQuery de Harlowe.

### 1. Scaffold del DOM

Se crea en el `<body>`, **fuera de `tw-story`**, para que persista entre pasajes:

| ID | Tipo | Descripción |
|---|---|---|
| `#vn-bg` | `<div>` | Capa de fondo. Su `background-image` se actualiza por JS. |
| `#vn-stage` | `<div>` | Contenedor de sprites. |
| `#vn-left` | `<img.vn-sprite>` | Sprite izquierdo. |
| `#vn-right` | `<img.vn-sprite>` | Sprite derecho. |
| `#vn-name` | `<div>` | Placa de nombre del hablante. Contiene `.vn-name-chip`. |
| `#vn-bar` | `<div>` | Barra de controles (↶ ↷ ☰). |
| `#vn-log-overlay` | `<div>` | Overlay del registro de diálogo. |
| `#vn-blackout` | `<div>` | Capa negra para transiciones fade. |
| `#vn-loading` | `<div>` | Overlay del preloader. |

### 2. Constantes de navegación con fade (`FADE_PASSAGES`)

```js
var FADE_PASSAGES = ["Primera noche", "La Mañana", "El Cafe", "El Cafe Noche"];
```

Pasajes que entran con un fade a negro en lugar de la transición estándar de Harlowe. Para agregar más, añadir el nombre exacto del pasaje a este array.

**`fadeGoto(destino, duracion)`** — hace fade out → navega → fade in.  
**`advanceTo(adv)`** — wrapper que decide si usar `fadeGoto` o el click estándar según si el destino está en `FADE_PASSAGES`.

### 3. `commit()` — ciclo de actualización por pasaje

Es la función central. Se llama cada vez que Harlowe renderiza un pasaje nuevo y aplica todos los cambios de escena:

```
commit()
  ├── applyScene(p)   → lee el hook |vn> y actualiza fondo, sprites y nombre
  ├── decorate(p)     → layoutChoices + trimEdgeBrs
  ├── reveal(p)       → fade-in del pasaje si es nuevo
  ├── syncNamePos(p)  → posiciona la placa de nombre sobre la caja
  ├── watchSidebar()  → inicia el MutationObserver del sidebar si no estaba activo
  └── refreshNav()    → muestra/oculta ↶ y ↷ según el estado del sidebar de Harlowe
```

**Temporización (40 / 250 / 650 ms):** Harlowe renderiza pasajes en varias pasadas asíncronas. Disparar `commit` tres veces garantiza capturar el estado final sin importar la carga del navegador.

```js
function schedule() {
    commitTimers.forEach(clearTimeout);
    commitTimers = [40, 250, 650].map(function (ms) {
        return setTimeout(commit, ms);
    });
}
```

Un `MutationObserver` sobre `document.body` dispara `schedule()` cada vez que Harlowe inserta un `<tw-passage>` nuevo.

### 4. `applyScene(p)` — escena por pasaje

Lee el hook `|vn>` del pasaje activo, que tiene el formato:

```
fondo | sprite_izq | sprite_der | hablante | nombre_visible
```

- Si el slot está vacío, ese elemento no cambia (fondo) o se oculta (sprites/nombre).
- `hablante` puede ser `"izq"`, `"der"` o vacío (narración).
- Llama a `setSprite()` para cada lado y a `recordLog()` para añadir la línea al registro.

### 5. `setSprite(img, file, speaking, anySpeaker)` — gestión de sprites

- Si `file` está vacío: oculta el sprite (`display:none`, `opacity:0`, quita `src`).
- Si el `src` cambió: hace fade opacity 0→1 al cargar (o de inmediato si ya está en caché).
- `anySpeaker && !speaking` activa `.vn-dim` (atenúa el sprite que no habla).
- La clase `sprite-<nombre>` permite overrides de posición por personaje en el CSS.

### 6. `layoutChoices(p)` — detección de decisión vs. avance

Determina el comportamiento de la caja según la cantidad de `<tw-link>` en el pasaje:

| Links | Comportamiento |
|---|---|
| 2 o más | **Decisión:** links en fila horizontal, visibles. La caja **no** es clickeable. |
| 1 | **Avance:** el link se oculta (`display:none`). La caja **entera** es clickeable. |
| 0 | La caja no hace nada. |

**Por qué no se reparentan los links:** Harlowe guarda el binding de cada `[[link]]` en el elemento `<tw-expression>` que lo envuelve. Moverlo en el DOM rompe ese binding. En cambio, se eliminan los `<br>` y nodos de texto entre los `<tw-expression>` para que queden en línea.

**Re-entrancia:** `layoutChoices` resetea el estado al inicio de cada llamada para evitar que una corrida anterior (con el pasaje a medio renderizar) deje artefactos.

### 7. Undo / Redo (`#nybdv-back` / `#nybdv-redo`)

Los botones propios delegan en los íconos nativos del sidebar de Harlowe:

```js
back.addEventListener("click", function () {
    var u = document.querySelector('tw-sidebar tw-icon[title="Undo"]');
    if (u && u.style.visibility !== "hidden") jqTrigger(u);
});
```

**Por qué `jqTrigger` y no `click()` nativo:** Harlowe usa jQuery internamente. `$(el).trigger("click")` activa los handlers de jQuery; un `el.click()` nativo no los activa.

**Fix del bug de avance involuntario:** `jqTrigger` simula burbujeo. El ícono de undo vive dentro de `tw-passage`, así que el click sintético llegaba a la caja clickeable y la hacía avanzar. El handler global de avance ignora clicks que provengan de:

```js
e.target.closest("tw-link, tw-sidebar, tw-icon, #vn-bar, #vn-log-overlay, #vn-blackout, #vn-ed-panel, .vn-ed-drag-overlay, .vn-ed-handle")
```

`refreshNav()` muestra u oculta los botones según la visibilidad del ícono nativo, observando el sidebar con un `MutationObserver`.

### 8. Registro de diálogo (`#vn-log-overlay`)

`recordLog(name, text)` guarda cada línea en el array `logData`, indexado por la longitud del historial de sesión de Harlowe (`sessionStorage["Saved Session"]`). Esto hace que el log sea coherente con undo/redo: si el jugador deshace y toma otra rama, las entradas posteriores se truncan y se reescriben.

`renderLog()` es incremental: solo appendea entradas nuevas al DOM. Si `logData` se truncó (undo + nueva rama), reconstruye desde cero.

Atajos de teclado: `Space` / `Enter` avanza el pasaje; `↑` abre/cierra el registro.

### 9. Preloader

Crea un `<img>` por cada archivo en `PRELOAD`, los carga en paralelo y actualiza la barra de progreso. Al completarse, hace fade out del overlay `#vn-loading`.

---

## Story Stylesheet — IDs y clases principales

### Variables CSS

```css
:root { --dialog-h: 36vh; }
```

`--dialog-h` se usa para posicionar `#vn-name` justo encima de la caja de diálogo. Si se cambia la altura de `tw-passage`, actualizar este valor en consecuencia.

### Estructura de capas (z-index)

| z-index | Elemento | Rol |
|---|---|---|
| 0 | `#vn-bg` | Fondo de escena |
| 1 | `#vn-stage` | Sprites |
| 2 | `tw-story` / `tw-passage` | Caja de diálogo |
| 3 | `#vn-name` | Placa de nombre |
| 5 | `#vn-bar` | Controles |
| 10 | `#vn-log-overlay` | Registro |
| 20 | `#vn-loading` | Preloader |
| 9999 | `#vn-blackout` | Fade entre pasajes |

### Overrides por personaje (`.sprite-<nombre>`)

El JS añade automáticamente una clase `sprite-<nombre_de_archivo_sin_extensión>` a cada `<img>` de sprite cuando lo muestra. Esto permite ajustar posición, escala o recorte por personaje sin tocar el JS:

```css
/* Ejemplo: agrandar levemente a Gabriel y desplazarlo */
.sprite-Gabriel {
    transform: scale(1.05) translateX(3%);
}
```

Propiedades seguras: `transform`, `object-position`, `max-height`, `width`, `bottom`.  
**No usar** `opacity` ni `display` (los controla el JS con `style` inline).

---

## Editor visual de sprites (dev tool)

Herramienta integrada para ajustar posición y tamaño de sprites visualmente en el navegador, sin tener que editar números a ciegas. Cuando está activa, cada sprite se puede arrastrar y redimensionar, y el panel muestra el bloque CSS listo para copiar al Story Stylesheet.

### Activar

En el Story JavaScript, cambiar la variable `SPRITE_EDITOR` (declarada junto a `var BASE`):

```js
var SPRITE_EDITOR = true;   // activar
```

> Volver a `false` antes de publicar. No afecta el gameplay ni el CSS guardado; solo agrega el panel y habilita el arrastre.

### Uso

1. Abrir el juego con el servidor local (`python serve.py`).
2. Navegar hasta un pasaje que tenga el sprite a ajustar.
3. **Mover:** hacer click y arrastrar el sprite a la posición deseada.
4. **Escalar:** arrastrar el cuadrado azul en la esquina superior derecha del sprite (hacia arriba agranda, hacia abajo achica).
5. El panel centrado en la parte superior de la pantalla muestra en tiempo real el CSS generado:

```css
.sprite-Gabriel {
    transform: translateX(2.50vw) translateY(-3.20vh);
    max-height: 85.00vh;
}
```

6. Hacer click en **Copiar** para llevar el bloque al portapapeles.
7. Pegarlo en el Story Stylesheet, dentro del bloque `.sprite-<nombre>` correspondiente.
8. **Reset** vuelve el sprite a su posición base para empezar de nuevo.
9. El botón **−** en el título del panel minimiza el panel para ver la escena completa; **+** lo expande de nuevo.

### Comportamiento

| Situación | Qué hace el editor |
|---|---|
| El sprite cambia de personaje (nuevo `src`) | Resetea el estado de arrastre; el nuevo sprite empieza sin transform |
| El mismo personaje aparece en otro pasaje | Mantiene el transform acumulado hasta el siguiente reset manual |
| Soltar el handle de resize | El click que el navegador dispara al soltar es suprimido automáticamente para que no avance la escena |
| `SPRITE_EDITOR = false` | El panel y los handles no se crean; cero impacto en runtime |

### Notas de unidades

Los valores generados están en `vw`/`vh` (relativos al viewport), igual que el resto del CSS del motor. Esto garantiza que el layout se vea igual en cualquier resolución.

El editor **no** lee transforms de clase preexistentes: si `.sprite-Gabriel` ya tiene `transform: scale(1.05)`, el arrastre lo sobreescribe. En ese caso, combinar manualmente la escala con la traslación en el CSS final:

```css
.sprite-Gabriel {
    transform: scale(1.05) translateX(2.50vw) translateY(-3.20vh);
}
```

### Implementación (para referencia)

El drag no se implementa directamente sobre los sprites porque `tw-story` (`z-index:2`, `position:fixed`, `inset:0`) intercepta todos los eventos de mouse antes de que lleguen a los sprites (`z-index:1`). La solución usa divs overlay transparentes (`position:fixed; z-index:8999`) que siguen la posición del sprite frame a frame con `requestAnimationFrame`. Los listeners de drag y resize van en estos overlays, no en los sprites.

Al soltar el handle de resize, el navegador dispara un `click` que sin tratamiento especial avanzaría la escena. El fix registra un listener de captura de un solo disparo en `document` que consume ese click antes de que llegue al handler del juego y se auto-destruye; el juego responde a todos los clicks posteriores con normalidad.

---

## Convención de autoría — el hook `|vn>`

Cada pasaje que muestra una escena empieza con un hook nombrado de Harlowe, que el JS lee y oculta por CSS:

```
|vn>[fondo | sprite_izq | sprite_der | hablante | nombre_visible]
```

**Campos (en orden):**

| Campo | Valores posibles | Efecto |
|---|---|---|
| `fondo` | nombre de archivo (ej. `FondoLiving.jpg`) | Cambia el fondo |
| `sprite_izq` | nombre de archivo (ej. `Gabriel.png`) o vacío | Muestra/oculta sprite izquierdo |
| `sprite_der` | nombre de archivo (ej. `Martina.png`) o vacío | Muestra/oculta sprite derecho |
| `hablante` | `izq`, `der` o vacío | Atenúa el sprite que NO habla; vacío = narración, ninguno atenuado |
| `nombre_visible` | texto libre (ej. `Gabriel`) o vacío | Muestra/oculta la placa de nombre |

**Ejemplos:**

```
|vn>[FondoLiving.jpg|Gabriel.png|Martina.png|der|Martina]
→ Fondo: living. Gabriel izq, Martina der. Habla Martina (Gabriel atenuado). Placa: "Martina".

|vn>[CentralElectrica.jpg||||]
→ Solo cambia el fondo. Sin sprites, sin placa. Narración.

|vn>[|Gabriel.png||izq|Gabriel]
→ Fondo sin cambio. Solo sprite izquierdo. Habla Gabriel.
```

**Slots vacíos:** dejar el campo en blanco entre barras. Un campo vacío equivale a "sin cambio" para el fondo y a "ocultar" para sprites y nombre.

### Formato completo de un pasaje

```
:: Nombre del pasaje [etiqueta-de-ubicacion]
|vn>[fondo|sprite_izq|sprite_der|hablante|nombre]

"Línea de diálogo del personaje."

[[Siguiente ->Nombre del pasaje siguiente]]
```

- La caja **no tiene scroll**: si el texto no entra, hay que partir el pasaje.
- Con **un solo link**: el link se oculta y la caja entera es clickeable (avance).
- Con **dos o más links**: se muestran como botones en fila horizontal (decisión).

---

## Variables Harlowe

Declaradas en el pasaje `Startup` (tag `startup`), se inicializan antes de que comience la historia:

```
(set: $cdn to "https://raw.githubusercontent.com/JuanPalbo/NyBdV/main/Imagenes/")
(set: $local to "Imagenes/")
(set: $dir to $cdn)
(set: $fondo to "")
```

> Nota: `$dir` era usado por los pasajes antiguos para construir URLs de imagen con `(print:)`. El motor nuevo lee `BASE` directamente desde el JS, así que `$dir` ya no es necesario en pasajes nuevos.

### Flags de rama

| Variable | Se activa en |
|---|---|
| `$SaboInt` | Pasaje `Sabotaje interno` |
| `$AtaqueExterno` | Pasaje `Un ataque externo` |

---

## Pasajes especiales

| Pasaje | Tag | Función |
|---|---|---|
| `Startup` | `startup` | Inicializa variables. Se ejecuta antes de `Intro`. |
| `StoryStylesheet` | `stylesheet` | CSS del motor VN (el `<style>` que Twine inyecta). |
| `StoryData` | — | Metadatos de la historia (IFID, formato, colores de tags, pasaje inicial). |

Los pasajes con tag `startup` los ejecuta Harlowe automáticamente al inicio, antes del pasaje marcado como `start` en `StoryData`.

---

## Assets

### Fondos

| Archivo | Usado en |
|---|---|
| `CentralElectrica.jpg` | `Intro` (apertura con la noticia) |
| `FondoLiving.jpg` | Pasajes con tag `living` |
| `FondoCuartoGabriel.jpg` | Pasajes con tag `cuarto-Gabriel` (día) |
| `FondoCuartoGabrielNoche.png` | Cuarto de Gabriel (noche) |
| `FondoCafe.png` | El Café (día) |
| `FondoCafeTarde.png` | El Café (tarde) |

### Sprites

| Archivo | Estado |
|---|---|
| `Gabriel.png` | En uso |
| `Martina.png` | En uso |
| `MartinaEsceptica.png` | En uso |
| `MartinaEyeroll.png` | En uso |
| `Nico.png` | Preparado, aún no usado en pasajes |
| `GabrielPlaceholder.png` | Placeholder viejo (con fondo, no transparente) |
| `MartinaPlaceholder.png` | Placeholder viejo (con fondo, no transparente) |
| `Martina.jpg`, `MartinaEsceptica.jpg`, `MartinaEyeroll.jpg` | Retratos viejos, no usados en la UI nueva |

> Los placeholders tienen fondo sólido. Reemplazarlos por PNGs transparentes cuando estén listos.

---

## Testing y debugging

### Resetear el estado entre pruebas

Harlowe guarda la sesión en `sessionStorage` y la restaura automáticamente al recargar (mismo origen). Para empezar desde cero:

```js
sessionStorage.clear(); location.reload();
```

> Navegar a `?v=X` **no** resetea: Harlowe restaura la sesión igualmente.

### Identificar el pasaje actual

```js
// Pasaje activo (el saliente queda en tw-transition-container)
document.querySelector("tw-story > tw-passage")

// Historial de pasajes navegados
JSON.parse(sessionStorage["Saved Session"]).map(e => typeof e === "string" ? e : e.passage)
```

> No usar snippets de texto para identificar pasajes: `Intro 3`, `Sabotaje interno` y `Un ataque externo` comienzan todos con la misma frase.

### Acceso a jQuery y al motor

```js
window.jQuery        // jQuery de Harlowe
Engine.goToPassage("Nombre")  // navegar a un pasaje por JS
```

### Imágenes que tardan en cargar

El preloader espera "network idle". Si el screenshot de Twine falla con "Page still loading", verificar por JS:

```js
document.querySelectorAll("img").forEach(i => console.log(i.src, i.complete))
```

---

## Pendiente

- [ ] **Estrategia de preload progresivo:** en lugar de precargar todos los assets al inicio, cargar solo las imágenes de las primeras escenas al lanzar el juego y cargar el resto en segundo plano mientras el usuario juega. La complicación es que si el juego no arranca desde la primera escena (por ejemplo, tras restaurar una sesión o hacer undo hasta un pasaje intermedio), las imágenes de ese punto podrían no estar listas. Para implementarlo bien hay que definir explícitamente qué assets corresponden a cada momento de la historia y precargarlo de forma contextual. Por ahora el preloader carga todo de una sola vez con jsDelivr como CDN, lo que mitiga el impacto.
