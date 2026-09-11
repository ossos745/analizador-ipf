# Analizador de validez IPF

Herramienta para analizar videos de sentadilla, press de banca y peso muerto, y evaluar
la validez del levantamiento según el reglamento técnico de la IPF. Un solo archivo HTML,
sin instalación, sin servidor: **el video se procesa en el navegador y no sale del dispositivo.**

No sustituye el criterio de un árbitro.

## Cómo se usa

Abre `index.html` en el navegador (Chrome recomendado) o, si activas GitHub Pages en este
repositorio, entra directamente desde la URL que te dé GitHub.

1. **Video.** Carga el archivo y elige el movimiento, la vista y el lado visible.
2. **Escala.** Marca el disco de la barra con el recuadro verde. De ahí salen dos cosas a la
   vez: la escala en centímetros y cuánto se aparta la cámara del perfil exacto.
3. **Seguir la barra.** Dibuja el recorrido del centro del disco frame a frame: celeste cuando
   baja, rojo cuando sube. Detecta las repeticiones y calcula velocidad, pico, duración y
   recorrido de cada una.
4. **Puntos.** Marca las articulaciones. Con marcas en varios frames la app las sigue por el
   video; donde no puede, lo dice.
5. **Reglas.** Checklist por movimiento, cada criterio con su medida y su banda de error.

## El principio: no fingir precisión

Todo veredicto tiene tres estados, nunca dos: **VÁLIDO**, **NO VÁLIDO** y **NO CONCLUYENTE**.
Cada medición se muestra con su banda de incertidumbre, y si el margen medido es menor que esa
banda, el veredicto es obligatoriamente no concluyente.

Consecuencias concretas de esa regla:

- Sin calibrar, todas las medidas van en píxeles y ningún criterio métrico puede dar VÁLIDO.
- Un punto que la app no ve en el frame que define el veredicto no se puede usar: no se dibuja
  la línea de profundidad ni se da un resultado, se pide marcarlo.
- Un punto seguido automáticamente ensancha la banda de la medida en la que entra. No vale lo
  mismo que uno colocado a mano.
- Si no se detecta rebote pero la resolución del video es peor que el umbral del reglamento,
  el resultado es no concluyente: no se puede certificar lo que no se puede ver.

Cada criterio declara además si es **AUTO** (medible geométricamente), **ASISTIDO** (la app
señala el frame y el valor, pero necesita confirmación visual) o **MANUAL** (imposible de
determinar desde el video). La app no detecta comandos del árbitro, ni contacto de los
cuidadores, ni audio, y nunca dice que los verificó.

No se calculan torques ni potencias: requieren masa segmentaria y peso de la barra, y
presentarlos junto a un veredicto daría una falsa impresión de rigor.

## Profundidad de sentadilla

Se evalúa comparando **alturas** entre el pliegue de la cadera y el borde superior de la
rodilla, que es el criterio del reglamento. El ángulo de rodilla no se usa para eso: 90° no
significa nada en la IPF.

La comparación se hace contra una horizontal de referencia configurable, porque si la cámara
está torcida la horizontal del gimnasio no es horizontal en la imagen.

## Cómo filmar

Cámara fija, a la altura de la cadera, perpendicular al atleta, a 3–5 m, 60 fps o más, sin gran
angular, el atleta completo en cuadro. La pestaña Ayuda de la app explica por qué cada cosa
importa y cuánto error introduce no cumplirla.

## Detección automática de pose

Opcional. Intenta cargar MediaPipe desde un CDN; si falla, la app funciona completa con marcado
manual y lo dice. También acepta el archivo del modelo desde el disco.

## Licencia

MIT.
