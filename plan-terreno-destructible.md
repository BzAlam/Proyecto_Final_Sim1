# Perlin Noise y Terreno Destructible en 2D (con HTML/JS y PixiJS)

Proyecto 2D con estilo *terraria*. Un terreno generado con Perlin noise que se puede destruir, renderizado con **PixiJS**.

```html
<script src="https://cdn.jsdelivr.net/npm/pixi.js@8/dist/pixi.min.js"></script>
```

## 1. Estructura de carpetas

```
proyecto/
├── index.html
├── style.css
├── package.json
└── src/
    ├── main.js              // arranque de Pixi, game loop
    ├── noise/
    │   ├── perlin.js        // Perlin noise
    │   └── fbm.js           // octavas (fractal Brownian motion)
    ├── terrain/
    │   ├── Terrain.js       // datos del terreno (grilla) y consulta de celdas
    │   ├── generator.js     // llena la grilla usando el ruido
    │   └── destruction.js   // irregularidades del terreno
    ├── render/
    │   └── TerrainRenderer.js // el sprite de pixi que usa una textura para el terreno
    ├── input.js             // inputs
    ├── ui.js                // panel de control
    └── utils/
        └── rng.js           // RNG con semilla (ver luego mulberry32)
```


## 2. Piezas clave

### 2.1 Perlin noise

- Tabla con 256 valores, mezclada con un RNG con semilla (si se pone la misma semilla sería el mismo mundo).
- Función `noise2D(x, y)`: da gradientes sumado a una interpolación suave.
- **fbm** (fractal Brownian motion): investigar pero parecería ser importante.

### 2.2 Representación del terreno

Tendríamos que elegir entre estas dos formas de representar el terreno: (Esto de una IA)

- **Heightmap** (una altura por columna): simple, pero no permite cuevas ni voladizos y limita la destrucción.
- **Grilla 2D de sólido/vacío o densidad** *(recomendada)*: un `Uint8Array` o `Float32Array` de `ancho × alto`. Permite cráteres, túneles y cuevas.


Generación:
1. Altura de la superficie con ruido 1D/2D (fBm).
2. Opcional: ruido 2D con umbral para excavar cuevas debajo de la superficie.

### 2.3 Destrucción

- Registrar el área destruida para actualizar solo esa región.
- Extras:
  - Detectar islas flotantes con *flood fill* y hacerlas caer.
  - Puede ser posible simular tierra/arena que cae con un autómata celular simple (falling sand).

### 2.4 Render con PixiJS

Representar todo el terreno como **una sola textura en un `Sprite`**.

Flujo recomendado por claude:

1. La grilla del terreno vive en memoria (`Uint8Array`).
2. Un `<canvas>` fuera de pantalla con su `ImageData` refleja los colores de cada celda.
3. `Texture.from(canvas)` crea la textura y un `Sprite` la muestra en el `stage`.
4. Cuando el terreno cambia: se repintan solo las celdas del dirty rect en el `ImageData`, se llama a `putImageData` con esa región y luego `texture.source.update()`.

> Aclaración: el dirty rect ahorra trabajo de CPU al repintar; la subida a GPU normalmente es de toda la textura, lo cual es rápido para tamaños típicos (por ejemplo 1024×512).

Alternativa: `BufferImageSource` de Pixi v8 permite usar directamente un `Uint8Array` RGBA como fuente de la textura, sin canvas intermedio. Verificá la API en la documentación de tu versión.

#### Ejemplo base

```js
// src/main.js
import { Application } from 'pixi.js';
import { TerrainRenderer } from './render/TerrainRenderer.js';

const app = new Application();
await app.init({ width: 1024, height: 512, background: '#87ceeb' });
document.body.appendChild(app.canvas);

// const terrain = new Terrain(1024, 512) ...
// const renderer = new TerrainRenderer(app, terrain);

app.ticker.add((ticker) => {
  // actualizar lógica con ticker.deltaTime
});
```

```js
// src/render/TerrainRenderer.js
import { Sprite, Texture } from 'pixi.js';

export class TerrainRenderer {
  constructor(app, terrain) {
    this.terrain = terrain;

    this.canvas = document.createElement('canvas');
    this.canvas.width = terrain.width;
    this.canvas.height = terrain.height;
    this.ctx = this.canvas.getContext('2d');
    this.imageData = this.ctx.createImageData(terrain.width, terrain.height);

    this.texture = Texture.from(this.canvas);
    this.texture.source.scaleMode = 'nearest'; // píxeles nítidos
    this.sprite = new Sprite(this.texture);
    app.stage.addChild(this.sprite);
  }

  // Pinta en imageData las celdas del rectángulo (x, y, w, h)
  paintRegion(x, y, w, h) {
    const { width, data } = { width: this.terrain.width, data: this.imageData.data };
    for (let j = y; j < y + h; j++) {
      for (let i = x; i < x + w; i++) {
        const [r, g, b, a] = this.terrain.colorAt(i, j);
        const o = (j * width + i) * 4;
        data[o] = r; data[o + 1] = g; data[o + 2] = b; data[o + 3] = a;
      }
    }
  }

  // Sube la región modificada a la textura
  flush(x, y, w, h) {
    this.ctx.putImageData(this.imageData, 0, 0, x, y, w, h);
    this.texture.source.update();
  }
}
```

#### Otras ventajas de Pixi para este proyecto

- **Ticker** integrado (`app.ticker`) con `deltaTime`.
- **Contenedores** (`Container`): mover/hacer zoom de una "cámara" cambiando posición y escala del contenedor.
- **Escombros y partículas**: sprites individuales o `ParticleContainer` para muchos elementos.
- **Filtros** (blur, sombras, brillo) para efectos de explosiones.
- Avanzado: renderizar el terreno con un **shader** que lee una textura de densidad (más eficiente, pero más complejo).

### 2.5 Game loop

`app.ticker` (basado en `requestAnimationFrame`): input → actualizar → Pixi renderiza automáticamente.

Para el mouse, se puede usar `app.canvas.addEventListener('pointerdown', ...)` convirtiendo las coordenadas con `getBoundingClientRect()`, o los eventos de Pixi (`eventMode = 'static'`).

---

## 3. Roadmap sugerido

1. Proyecto con Pixi funcionando (`Application`, fondo, ticker).
2. Implementar Perlin noise y visualizarlo en escala de grises sobre un Sprite.
3. fBm con parámetros ajustables y semilla.
4. Generar la grilla sólido/vacío y colorearla (pasto, tierra, roca).
5. Destrucción.
6. Optimización con dirty rects.
7. Panel de parámetros (lil-gui) y botón "regenerar con nueva semilla".
8. Extras: caída de tierra, escombros, cuevas, proyectiles, cámara con zoom.

---

## 4. Anotaciones

- Empezar con un tamaño de grilla chico (ponele 512×256 o 1024×512) y escalar el sprite si hace falta.
- Separar bien los datos (`Terrain`) de render (`TerrainRenderer`): ya que nos permite cambiar el render sin tocar la lógica.
- Guardar la semilla en pantalla para poder reproducir mundos interesantes.
