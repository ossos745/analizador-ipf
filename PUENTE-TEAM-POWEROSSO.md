# Puente entre Powerosso App y el Analizador IPF

Esto es lo que hay que pegarle al **otro** chat (el de Powerosso App). El lado
del analizador ya está hecho y funcionando en `analizador-ipf.html` (v24).

Está escrito contra el `CLAUDE.md` de Powerosso App: una sola página sin
framework, `src/app.src.html` como fuente y `index.html` generado por
`src/build.py`, `Store` con sus tres modos, Supabase por funciones `pw_*`,
acceso por `?k=<token>`. Donde menciono funciones de la app (`LIFT`, `basicOf`,
`softRender`, `vDash`) es a partir de esa guía, no de haber leído el código: hay
que confirmar las firmas exactas antes de escribir nada.

## Por qué así y no con una API

Las dos apps corren enteras en el navegador. El analizador no tiene backend y el
video no se sube a ningún lado —eso es un principio del proyecto, no un
detalle—. Entonces el enlace no puede ser una API: es un **traspaso de ida y
vuelta** entre dos pestañas del mismo dispositivo.

- **Ida:** Powerosso abre el analizador con el contexto de la serie en la URL.
- **Vuelta:** el analizador devuelve el veredicto y las métricas, y —si está
  marcada la casilla— el archivo de video.

Nada se manda solo. El envío lo dispara un botón en el analizador, y el botón
dice a qué sitio va.

---

## Decisión 1: el analizador vive en los dos lados

| Copia | URL | Para qué |
|---|---|---|
| **Interna** | `https://ossos745.github.io/powerosso-app/analizador/` | La que abre el botón "Evaluar en video" desde una serie. |
| **Pública** | `https://ossos745.github.io/analizador-ipf/` | Usarlo solo: un celular ajeno, un video que te mandan por WhatsApp, una consulta rápida. |

**Las dos están en el mismo origen.** `https://ossos745.github.io` es el origen
de las dos, porque el origen es el host, no la carpeta. Eso simplifica el puente
de verdad: `postMessage` entre ellas es same-origin, el archivo de video pasa sin
pelear con las restricciones de terceros de Safari, y el almacenamiento del
dispositivo es **el mismo** para las dos.

Ese "el mismo" tiene las dos caras:

- A favor: lo que guarde Powerosso en IndexedDB lo puede leer el analizador, y
  al revés. Hoy no hace falta, pero es la puerta para, más adelante, que el
  analizador abra solo un video ya guardado.
- En contra: **comparten `localStorage`**. Revisé las claves y hoy no chocan —el
  analizador usa `ipf.pref` y Powerosso usa `powerosso.v3`, `powerosso.k`,
  `powerosso.kc`—, pero esto hay que tenerlo presente: una clave nueva sin
  prefijo en cualquiera de las dos pisa a la otra. **Toda clave nueva lleva su
  prefijo**, `ipf.` o `powerosso.`.

Si algún día Powerosso se muda a dominio propio, las dos copias dejan de ser del
mismo origen y el puente sigue funcionando igual, pero hay que listar los dos
orígenes a mano:

```js
// hoy alcanza con uno; se deja como lista para el día que cambie el dominio
const ORIGENES_OK = ['https://ossos745.github.io'];
const ANALIZADOR  = location.origin + '/powerosso-app/analizador/';
```

### Cómo se publica la copia interna

`analizador-ipf.html` es un archivo solo, sin dependencias ni build. Va al repo
`powerosso-app` como `analizador/index.html`, **copiado tal cual**. No pasa por
`src/build.py` y no se edita ahí: la fuente vive en el otro proyecto y acá sólo
se copia el archivo ya hecho. Conviene dejarlo dicho en el `CLAUDE.md` de
Powerosso, porque la convención de ese repo es "nunca editar `index.html` a
mano" y este es la excepción.

Dos cosas que revisar al publicarla:

1. **El service worker la va a tomar.** `sw.js` tiene alcance `/powerosso-app/`,
   así que `/powerosso-app/analizador/` queda adentro. Siendo *network first*,
   eso termina siendo bueno: el analizador queda disponible sin señal. Sólo hay
   que confirmar que el SW no intente cachear los `blob:` de los videos.
2. **Las dos copias, siempre la misma versión.** El paquete trae `version`:
   guárdala en el registro y, si no coincide con la que la app espera, muéstralo
   como aviso en la serie —no rechaces el paquete—. Un dato viejo bien
   etiquetado sirve; un dato viejo disfrazado de nuevo, no.

---

## Ida: abrir el analizador desde una serie

```js
// en la sesión, al tocar "Evaluar en video" en una serie de un básico
const token = crypto.randomUUID();
sessionStorage.setItem('ipf.token', token);          // con prefijo: origen compartido

const p = new URLSearchParams({
  origen:   'powerosso',                  // obligatorio: lo que activa el puente
  alumno:   aid,
  nombre:   DATA.athletes[aid].data.name, // sólo para mostrarlo en pantalla
  ejercicio: String(exIndex),             // el índice del ejercicio en el día
  ejNombre: ex.name,
  serie:    String(nSerie),
  sesion:   aid + '~' + meso + '~' + w + '-' + d,   // el id del log
  lift:     liftDe(ex),                   // squat | bench | deadlift
  token:    token,
  ret:      location.origin + location.pathname,
  cerrar:   '1'
});

const w2 = window.open(ANALIZADOR + '#' + p.toString(), 'ipf');
```

Tres cuidados:

- **Los parámetros van en el fragmento, después del `#`.** El fragmento no se
  manda a ningún servidor ni queda en logs de acceso. Importa porque ahí va el
  nombre del alumno.
- **El `?k=` del alumno no se pasa nunca.** El analizador no necesita el token de
  acceso y no tiene por qué verlo: lleva su propio token de un solo uso, que
  sólo sirve para emparejar la vuelta con la serie que lo pidió.
- **`sesion` es el id del log tal como ya lo usas** (`<aid>~<MESO>~<w>-<d>`). Eso
  hace que la vuelta sepa exactamente dónde escribir, sin inventar un id nuevo.

---

## Decisión 2: la vuelta trae el video, y el video queda en el dispositivo

### Cómo viaja

`postMessage` no manda sólo texto: el navegador clona estructuras, y un `File`
sobrevive el clonado con su nombre, su tamaño y su tipo. Está probado en v24: un
`File` de 3 MB llega al otro lado como `[object File]`, intacto. No se convierte
a base64, no se copia a ningún disco por el camino, no pasa por Supabase: es la
misma referencia al archivo, pasada de una pestaña a la otra.

El mensaje es el paquete JSON **más una propiedad**:

```js
{ ...paquete, videoArchivo: File }   // sólo si paquete.video.adjunto === true
```

- **`paquete.video.adjunto` dice si el archivo viene o no.** La casilla del
  analizador ("Devolver también el video") viene marcada y se puede desmarcar: un
  video de teléfono pesa fácil 100–300 MB.
- **El archivo va completo y tal como está**, sin recomprimir y sin recortar.
  Recortarlo en el navegador significa recodificarlo: lento, calienta el
  teléfono, pierde calidad. El paquete trae `video.rangoFrames` y `video.fps`,
  así que Powerosso reproduce **sólo la parte analizada** sin tocar el archivo
  (`video.currentTime = rangoFrames[0] / fps`, y corta en el otro).
- **Por el camino de reserva el video no puede ir.** Si la ventana se perdió y la
  vuelta es por navegación con el paquete en el fragmento, viajan sólo los
  números y `adjunto` viene en `false`. Lo mismo al pegar el paquete a mano.

### Recibirlo

```js
window.addEventListener('message', async ev => {
  if (ORIGENES_OK.indexOf(ev.origin) < 0) return;                  // de dónde viene
  const d = ev.data;
  if (!d || d.app !== 'analizador-ipf' || d.formato > 2) return;   // qué es
  if (!d.contexto || d.contexto.token !== sessionStorage.getItem('ipf.token')) return;
  sessionStorage.removeItem('ipf.token');                          // una sola vez

  let vid = false;
  if (d.video && d.video.adjunto && d.videoArchivo) {
    await ipfGuardarVideo(d, d.videoArchivo);
    vid = true;
  }
  await ipfGuardarMetricas(d, vid);
  softRender();
});
```

### Dónde se guarda cada cosa

Dos lugares distintos, y la diferencia importa:

| Qué | Dónde | Por qué |
|---|---|---|
| El **video** y el **paquete completo** | IndexedDB, sólo en el teléfono | Pesa. No tiene por qué sincronizarse ni salir del aparato. |
| Un **resumen de las métricas** | dentro del set del log, en Supabase | Es lo que alimenta el dashboard y lo que el coach tiene que ver. Pesa menos de 1 kB. |

**Las métricas van dentro del set, como `pr`.** El modelo ya tiene el lugar: un
set es `{kg, reps, rpe, t, pr?}`. Se le agrega un `ipf?` opcional:

```js
sets[exIndex][nSerie-1].ipf = {
  v:    d.version,              // 'v24'
  at:   d.enviado,
  ver:  d.veredicto.estado,     // VALIDO | NO_VALIDO | NO_CONCLUYENTE
  mot:  d.veredicto.motivo,
  vSub: p.velSubida.prom,  vSubB: p.bandaVelPct,   // m/s y banda en %
  vBaj: p.velBajada.prom,
  dSub: p.durSubida.prom,  dBaj:  p.durBajada.prom, // s
  rom:  p.recorrido.prom,  romB:  p.recorrido.banda, // cm
  perd: p.perdidaPct ? p.perdidaPct.primeraUltima : null,
  cal:  d.incertidumbre.calibrado,
  vid:  vid                     // si el video quedó guardado en este aparato
};
// y después, como cualquier escritura de la app
Store.put('logs', logId, log);
```

Esto no inventa tabla, no inventa función `pw_*` y no toca permisos: el alumno
ya puede escribir sus propios `logs`, el id sigue empezando con `<aid>~`, y
`Store.put` encola la escritura como siempre. **Es importante que el resumen sea
chico**: `pw_load` corre cada 20 segundos, y meter el paquete entero en cada set
engordaría toda la sincronización para algo que casi nunca se mira completo.

El paquete completo sí se guarda, pero en el aparato:

```js
async function ipfGuardarVideo(d, archivo) {
  const db = await ipfDB();                       // store 'ipf', clave de texto
  const id = d.contexto.sesion + ':' + d.contexto.ejercicio + ':' + d.contexto.serie;
  await ipfPut(db, id, {
    blob: archivo,                                // el Blob se guarda tal cual
    nombre: archivo.name, bytes: archivo.size, tipo: archivo.type,
    rango: d.video.rangoFrames, fps: d.video.fps,
    paquete: d,                                   // sin videoArchivo: no es serializable como tal
    guardado: new Date().toISOString()
  });
}

// para verlo después
const v = await ipfGet(db, id);
el.src = URL.createObjectURL(v.blob);             // revocar al cerrar
el.currentTime = v.rango[0] / v.fps;
```

IndexedDB y no otra cosa, por tres razones: `localStorage` tiene 5 MB y además
es donde vive `powerosso.v3`, así que un video lo reventaría; Supabase no es
lugar para 200 MB de video de un alumno; y el origin private file system (OPFS),
que sería más prolijo para archivos grandes, todavía tiene soporte irregular en
Safari. Se puede migrar después sin tocar el puente.

### Tres cuidados del almacenamiento

1. **Pedir almacenamiento persistente:** `await navigator.storage.persist()`. En
   iOS, el almacenamiento de un sitio que **no** está instalado en la pantalla
   de inicio se borra a los siete días sin uso. Powerosso ya tiene banner de
   instalación como PWA, así que esto no es un problema nuevo: es un argumento
   más para insistir con que el alumno la instale, y vale decirlo en la pantalla
   donde se guarda el video.
2. **Mostrar el espacio** antes de guardar:
   `const {usage, quota} = await navigator.storage.estimate();`. Guardar 200 MB
   sin avisar es la forma más rápida de que la app parezca rota.
3. **Decidir qué videos se quedan.** Esto se llena: 2 minutos de video son unos
   200 MB. Lo que propongo: el video se queda en las series marcadas (PR,
   intento de competencia, o las que el alumno marque) y de las demás se borra
   solo pasados N días. **Las métricas nunca se borran con el video**: el set
   queda con su `ipf` y `vid:false`, no queda vacío. Es el mismo criterio que ya
   usas con la foto del entrenamiento, que tampoco se sube.

---

## Decisión 3: sólo en los tres básicos de competencia

La opción "Evaluar en video" aparece **únicamente** en sentadilla, press de banca
y peso muerto de competencia. En lo demás no se muestra: el analizador juzga
contra el reglamento IPF, y aplicarlo a un hip thrust o a una sentadilla con
pausa no significa nada.

La app ya tiene con qué decidirlo. `parser.js` trae `LIFT`, y el dashboard usa
`basicOf` para mapear variantes al básico y `TSVAR` para distinguir las
variantes (pausa, búfalo, block). El filtro es justamente esa diferencia:

```js
// básico de competencia = basicOf lo reconoce Y no es una variante
const LIFT_IPF = { 'Sentadilla':'squat', 'Press banca':'bench', 'Peso muerto':'deadlift' };
function liftDe(ex) {
  const b = basicOf(ex.name);              // confirmar firma real
  if (!b || esVariante(ex.name)) return null;   // TSVAR -> fuera
  return LIFT_IPF[b] || null;
}
const puedeEvaluar = ex => !!liftDe(ex);
```

Filtrar por el nombre crudo se rompe el día que escribas "Sentadilla pausada
3seg" o "Banca cerrada": o te quedas sin la opción donde la querías, o te
aparece donde no corresponde. Usar lo que ya clasifica ejercicios en la app es
una función de cinco líneas y no se rompe.

Si alguna vez quieres medir velocidad en una variante **sin** juzgar reglamento,
eso es otra cosa y merece otro botón: el analizador tendría que entrar con el
veredicto apagado. Hoy no existe; se puede agregar.

### Los gráficos del dashboard

Van en `vDash`, con el mismo patrón que "Avance de topsets" y `rpeCards`: línea
por semana, filtro por levantamiento, `fmt()` con coma decimal, y todo escapado
con `esc()`.

| Gráfico | De dónde sale | Para qué sirve |
|---|---|---|
| **Carga–velocidad** (kg en X, m/s en Y) | el `kg` del set × `ipf.vSub` | El más útil de todos, y en tu app sale gratis: ya tienes el kg de cada serie. Es la recta que dice a qué velocidad mueve cada intensidad. |
| **Velocidad media por semana**, una línea por básico | `ipf.vSub` | Si la misma carga se mueve más rápido que el mes pasado, mejoró. |
| **Pérdida dentro de la serie** | `ipf.perd` | Cuánto se cae de la primera repetición a la última. Es la medida de fatiga de la serie. |
| **Tiempos de fase** (barras: bajada / subida) | `ipf.dSub`, `ipf.dBaj` | Si el tempo se mantiene o la subida se alarga con la carga. |
| **Profundidad** (sólo sentadilla) | la medida "Profundidad" del paquete | Que no se vaya acortando sin que nadie lo note. |
| **Historial de veredictos** | `ipf.ver` | Cuántas válidas, cuántas no, cuántas sin resolver. |

Y tres reglas para que no mientan, que son las mismas que rigen el analizador:

1. **La banda se dibuja.** Cada velocidad y cada medida viene con su banda
   (`vSubB`, `romB`). En una línea eso es una franja sombreada alrededor. Sin la
   franja, dos puntos que se ven distintos pueden ser el mismo número medido dos
   veces.
2. **No mezclar calibraciones.** Si `ipf.cal` es `false` no hay centímetros
   reales: ese punto puede ir en un gráfico de velocidad, pero no en uno de
   recorrido en cm junto a puntos calibrados. Separa o marca el punto.
3. **`NO_CONCLUYENTE` no es un número.** No va como cero, no va como "no válida",
   y tampoco se esconde: va como punto hueco o hueco marcado en la serie.
   Esconderlo es la única forma de que el dashboard mienta de verdad, porque
   convierte "no se pudo medir" en "no pasó nada".

**Sobre el 1RM:** la app ya estima con Bilbo (`peso × reps × 0,03 + peso`). La
recta de carga–velocidad da otra estimación, por otro camino. No las pongas en
la misma línea sin decir cuál es cuál: cuando difieran —y van a diferir— esa
diferencia es información, no un error que haya que promediar.

---

## El paquete

```json
{
  "app": "analizador-ipf",
  "formato": 2,
  "version": "v24",
  "enviado": "2026-11-03T12:00:00.000Z",
  "contexto": { "alumno":"a7", "nombre":"Juan Pérez", "ejercicio":"3",
                "ejNombre":"Sentadilla", "serie":"2",
                "sesion":"a7~HIPERTROFIA~2-1", "token":"…" },
  "video": { "nombre":"squat.mp4", "fps":30, "ancho":720, "alto":1280,
             "tipo":"video/mp4", "bytes":42123456, "adjunto":true,
             "rangoFrames":[60,210], "duracionRangoS":5.03 },
  "ejercicio": { "movimiento":"squat", "vista":"lateral", "lado":"left" },
  "veredicto": { "estado":"VALIDO", "motivo":"…" },
  "criterios": [ { "codigo":"S2", "criterio":"…", "tipo":"ASISTIDO", "estado":"ok" } ],
  "medidas":   [ { "nombre":"Profundidad", "valor":-2.1, "banda":0.8, "unidad":"cm", "nota":"…" } ],
  "parametros": {
    "reps": 5, "serieS": 18.4,
    "velSubida":     { "prom":0.764, "max":0.81, "min":0.62, "n":5, "unidad":"m/s" },
    "velSubidaPico": { "…": "…" },
    "durSubida":     { "…": "…" },
    "velBajada":     { "…": "…" },
    "velBajadaPico": { "…": "…" },
    "durBajada":     { "…": "…" },
    "recorrido":     { "…": "…" },
    "perdidaPct": { "primeraUltima":18.2, "mejorUltima":23.4, "mejorPeor":23.4 },
    "bandaVelPct": 5.3
  },
  "reps": [ { "n":1, "velSubida":0.81, "velSubidaBanda":0.04, "velSubidaPico":1.12,
              "velBajada":0.55, "velBajadaPico":0.9, "recorridoCm":41.2,
              "recorridoBanda":1.1, "durSubidaS":0.51, "durBajadaS":0.75,
              "frames":{ "arribaIni":61, "fondo":106, "arribaFin":149 } } ],
  "incertidumbre": { "k":1.96, "sigmaPx":1.5, "pxPorCm":4.2, "calibrado":true },
  "aviso": "Herramienta de entrenamiento. No sustituye el criterio de un árbitro IPF…"
}
```

De dónde sale cada cosa, por si importa: **todo lo general —velocidades, tiempos,
recorridos, repeticiones— se calcula sobre el centro del disco que se marca en
Escala**, no sobre los puntos del cuerpo. Los puntos del cuerpo (que son los que
a veces fallan) sólo entran en ángulos y profundidad. Por eso las velocidades
son el dato más confiable del paquete.

Tres cosas que conviene respetar al guardarlo:

1. **`veredicto.estado` tiene tres valores, no dos.** Si se guarda como booleano,
   convierte un "no se sabe" en un "no". El registro tiene que poder mostrar los
   tres.
2. **Cada medida y cada velocidad viene con su banda.** Un número sin banda no se
   puede comparar entre sesiones: no se sabe si la diferencia es del atleta o
   del método. Guarda las dos y muéstralas como `0,76 ± 0,04 m/s`.
3. **`parametros` puede venir en `null`**, si no se detectaron repeticiones. Es
   honesto, no es un error: no hay nada que guardar, y el set no lleva `ipf`.

### Criterios ASISTIDO

En `criterios`, `tipo` puede ser `AUTO`, `ASISTIDO` o `MANUAL`, y `estado` puede
venir en `manual`: la app midió algo pero no se anima a dictar el veredicto sola
y pide confirmación humana (el rebote en banca y en sentadilla son así desde
v23). En el registro eso se guarda como está, no como "válido".

---

## Vuelta de reserva, por navegación

Si la ventana se perdió o el navegador la bloqueó, el analizador navega a tu
`ret` con el paquete en el fragmento:
`https://ossos745.github.io/powerosso-app/#powerosso=<base64url del JSON>`

```js
(function ipfRecibirPorURL() {
  const m = /[#&]powerosso=([A-Za-z0-9_-]+)/.exec(location.hash);
  if (!m) return;
  const s = m[1].replace(/-/g, '+').replace(/_/g, '/');
  const bin = atob(s + '='.repeat((4 - s.length % 4) % 4));
  const b = new Uint8Array(bin.length);
  for (let i = 0; i < bin.length; i++) b[i] = bin.charCodeAt(i);
  const d = JSON.parse(new TextDecoder().decode(b));   // UTF-8: los acentos sobreviven

  history.replaceState(null, '', location.pathname + location.search);
  if (d.app === 'analizador-ipf' && d.contexto &&
      d.contexto.token === sessionStorage.getItem('ipf.token')) ipfGuardarMetricas(d, false);
})();
```

Ojo con esto: la app usa el hash para otras cosas (`#<aid>` para ver como
alumno), así que el lector tiene que correr **antes** del ruteo y limpiar el
hash con `replaceState`, como arriba, para no dejar a la app confundida.

Un paquete de 5 repeticiones queda en unos pocos kB de base64, muy por debajo de
lo que aguanta un fragmento. Por acá **el video no viaja** (`adjunto:false`): si
lo quieres, deja en la serie un botón para adjuntarlo a mano con un
`<input type="file">`.

### Y a mano

El analizador tiene **Copiar paquete** en la pestaña Exportar. Si en Powerosso
dejas un campo donde pegar ese JSON, el puente funciona incluso sin que las dos
apps se vean.

---

## Checklist de implementación

**Lado del analizador — listo en v24:**

- [x] Lee el contexto del fragmento y fija el movimiento según `lift`.
- [x] Devuelve el paquete por `postMessage`, y por navegación si no hay ventana.
- [x] Devuelve el archivo de video como `videoArchivo` (casilla marcada por defecto).
- [x] `video.adjunto` dice la verdad en los tres caminos (ventana, fragmento, copiado a mano).
- [x] Botón "Copiar paquete" para el camino manual.

**Lado de Powerosso App — falta:**

- [ ] `liftDe(ex)` sobre `basicOf`/`TSVAR`, y el botón en la serie sólo si devuelve algo.
- [ ] Abrir el analizador con el contexto y el token (sin pasar nunca el `?k=`).
- [ ] Recibir el paquete validando origen, formato y token; y el lector del hash
      antes del ruteo.
- [ ] `ipf` dentro del set del log, con `Store.put('logs', …)`.
- [ ] IndexedDB para el video y el paquete completo, con `navigator.storage.persist()`.
- [ ] Política de limpieza de videos, con el espacio usado a la vista.
- [ ] Los gráficos en `vDash`, con la franja de la banda.
- [ ] Copiar `analizador-ipf.html` a `analizador/index.html` del repo
      `powerosso-app`, y anotar en su `CLAUDE.md` que ese archivo se copia y no
      se genera.

**Convenciones del repo de Powerosso que aplican a todo esto:** editar
`src/app.src.html` y regenerar con `python3 src/build.py`, commitear
`index.html` junto con el cambio, todo en español, `esc()` para texto de usuario,
`fmt()` para números, y cuidado con `softRender()` y `S.dirty` al agregar campos.
