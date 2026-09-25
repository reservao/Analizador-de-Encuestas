# Analizador de Encuestas — mapa del proyecto

Este repo tiene **un solo archivo de aplicación**: `analizador_encuestas_v2.html`
(~9300+ líneas). Es una app JDR (Job Demands-Resources) de análisis de encuestas
de clima/engagement, 100% cliente (sin backend): todo el procesamiento del Excel
subido ocurre en el navegador y nunca sale de ahí. Se publica vía GitHub Pages.

**Antes de tocar el archivo**: usa este mapa para ubicarte por número de línea
en vez de leerlo completo o grepear a ciegas — ahorra muchísimo contexto. Los
números de línea pueden haberse corrido unas pocas líneas por ediciones
posteriores; si un grep no encuentra algo exactamente donde se indica, buscar
por el nombre de función/comentario de sección es más confiable que el número.

## Stack y libs (cargadas por CDN en el `<head>`)

- **xlsx-js-style** (fork de SheetJS con estilos) — lectura/escritura de Excel
- **D3.js 7.8.5** — organigrama (Sección 1)
- **Chart.js 4.4.1** — todos los gráficos
- **JSZip** — usado por exports
- **html2canvas** y **PptxGenJS** — cargados de forma perezosa (lazy, inyectados
  por JS cuando se necesitan) para exportar PNG/PPT del One Page

Para probar cambios sin depender de la red real, se puede copiar el archivo,
reemplazar esos `<script src="https://...">` por copias locales descargadas una
vez al scratchpad, y correr con Playwright headless
(`chromium.launch({executablePath:'/opt/pw-browsers/chromium'})`). Ver sección
"Metodología de verificación" más abajo.

## Navegación de la UI (3 niveles anidados)

La UI tiene pestañas de nivel superior controladas por `goSection(n)` que
togglean `sec1-content` / `sec2-content` / `sec3-content`:

| # | Botón UI | div contenedor | Qué hace |
|---|----------|-----------------|----------|
| 1 | "① Previo Medición" | `sec1-content` (línea ~423) | Módulo **S1** — valida el archivo de carga ANTES de lanzar la encuesta: columnas requeridas, IDs/correos duplicados o inválidos, organigrama (D3) con detección de ciclos/jefaturas huérfanas |
| 2 | "② Revisión Resultados" | `sec2-content` (línea ~643) | Módulos **G** y **C** — valida el archivo de resultados DESCARGADO: detección de dimensiones, rangos de respuesta, recalculo de dimensiones vs. lo que entregó la plataforma, consistencia carga-vs-descarga, "Nuevas Variables" (Alto Engagement/Agotamiento, eNPS) |
| 3 | "③ Reportería" | `sec3-content` (línea ~974) | Sub-pestañas via `rGoSub('tables'\|'charts'\|'onepage')` — ver abajo |

Dentro de Sección 3 (`rGoSub`, línea 4231), tres sub-tabs adicionales:

| sub-tab | div | Módulo | Qué hace |
|---------|-----|--------|----------|
| "📊 Tablas Excel" | `r-sub-tables` (~984) | **R** | Genera tablas resumen por dimensión/clasificación exportables a Excel |
| "📈 Gráficos" | `r-sub-charts` (~1057) | **RC** (gráficos libres) + **RCP** (gráficos predeterminados, dentro del mismo div) | RC = constructor de gráficos a medida. RCP = 7 gráficos predefinidos (Niveles de Engagement/Agotamiento, Histórico, Recursos y Demandas, Ranking Criticidad, Óptimo vs Crítico, eNPS, Demográficos) |
| "📄 One Page" | `r-sub-onepage` (~1265) | **ROP** | Reporte de una lámina (Engagement + Agotamiento + Recursos y Demandas + eNPS) por segmento, exportable a PNG o en lote a PPT |

**Importante para pruebas headless**: si vas a invocar funciones de un módulo
directamente por JS (sin clickear la UI), primero hay que activar la pestaña
correspondiente (`goSection(3); rGoSub('onepage');` etc.) — si no, los
contenedores quedan con `display:none` y todo lo que dependa de layout (medidas
de canvas, html2canvas) sale con tamaño 0×0. Ya se pisó este error varias veces
en esta sesión.

## Objetos de estado global (uno por módulo, todos `let` a nivel de script)

| Objeto | Línea decl. | Módulo |
|--------|-------------|--------|
| `G` | 1491 | Sección 2 — validador de resultados descargados |
| `C` | 1668 | Sección 2 — consistencia carga vs. descargado |
| `S1` | 2858 | Sección 1 — previo medición / organigrama |
| `R` | 4238 | Tablas Excel |
| `RC` | 4498 | Gráficos libres |
| `RCP` | 5044 | Gráficos predeterminados |
| `ROP` | 8598 | One Page |

Cada módulo tiene su propio `headers`/`rows` cargados independientemente (subes
el Excel por separado en cada sección) y sus propias funciones de detección de
columnas — **no están unificadas**, así que un bug de detección de columnas
casi siempre hay que replicarlo/arreglarlo en varios módulos a la vez (ver
"Patrones y trampas" abajo).

## Mapa de secciones por comentario (`// ─── NOMBRE ───`)

```
1420  MAESTRO (catálogo de dimensiones JDR para Sección 2/G — preguntas, textos, rangos)
1493  Normalización (norm())
1558  Análisis (detectDimensions, checkRanges)
1670  Navegación / Render Paso 1/2/3 (S2: validación de resultados)
1882  Descargas
1958  Utilidades UI
2092  Paso 5: Consistencia (módulo C)
2416  Módulo 6: Nuevas Variables (Alto Engagement/Agotamiento, eNPS — sobre G)
2694  Sección 1: Previo Medición (arranca módulo S1)
2696  Verificación universal de columnas (helper UI reusado por varios módulos)
2793  Mapeo manual de columnas
3148  Organigrama D3 (S1)
3817  Descarga HTML interactivo (organigrama)
4033  Reporte de validación mejorado
4230  Sección 3: Reportería (arranca R — Tablas Excel)
4240  Shared utility functions (normalize(), findCol(), escHtml(), ITEM_CODE_TEXTS — ver abajo)
4497  Sección 3B: Gráficos (arranca RC — gráficos libres)
5043  Predeterminados (arranca RCP — gráficos predeterminados, RCP_DEFS)
8597  One Page (arranca ROP)
```

## Funciones/constantes clave compartidas (línea ~4212 en adelante, aprox.)

- `normalize(s)` / `norm(s)` — dos normalizadores de texto ligeramente distintos
  (uno quita espacios, el otro los conserva) usados por distintos módulos para
  comparar nombres de columna sin tildes/mayúsculas. `rNorm`/`rcpNorm` son alias
  de `normalize` dentro de R/RCP respectivamente.
- `findCol(q, headers)` — busca una columna por código corto o texto exacto
  (usado por G/ROP). Devuelve el **nombre de columna** (string) o `null`.
- `rColIdx(name)` (R) / `rcpColIdxFuzzy(col)` (RCP) — buscan por código, alias
  conocidos (`RCP_COL_ALIASES`) y coincidencia difusa. Devuelven **índice**
  (number) o `-1`. Los tres (`findCol`, `rColIdx`, `rcpColIdxFuzzy`) son
  independientes entre sí — no comparten código, así que un fix de detección en
  uno NO se propaga a los otros automáticamente.
- `escHtml(s)` / `escJsAttr(s)` (~línea 3777) — escapan texto antes de
  insertarlo en `innerHTML`. **Usar siempre que se inserte un valor que venga
  del Excel del usuario** (encabezado, valor de celda, clasificación, ID,
  correo) en un template literal de `innerHTML`. Ver "Seguridad" abajo.
- `ITEM_CODE_TEXTS` + `itemTextCol()` / `rItemIdx()` / `rcpItemIdx()`
  (agregadas en esta sesión, cerca de `findCol`/`rColIdx`/`rcpColIdxFuzzy`) —
  respaldo para cuando el Excel trae el **texto literal de la pregunta** como
  encabezado (ej. "Mientras trabajo me siento lleno de energía") en vez del
  código corto (`WE1`). Intentan primero el código, si falla prueban el texto.
- `MAESTRO` (1421) y `R_DIMS` (dentro de la sección R, buscar `const R_DIMS`)
  — dos catálogos de dimensiones JDR **separados y no siempre coincidentes**
  (MAESTRO tiene un bug conocido de indexado de ítems de Vigor — ver abajo).
  `R_DIMS` es el más confiable para agrupaciones de sub-dimensión.
- `RCP_COL_ALIASES` (dentro de sección RCP) — alias de nombres de columna por
  plataforma/idioma para el módulo RCP específicamente.
- `RCP_DEFS` — array con la definición de los 7 gráficos predeterminados
  (id, nombre, parámetros configurables).

## Patrones y trampas ya encontradas (para no repetir el diagnóstico)

1. **"Alto Engagement" ≠ promedio de Engagement.** Alto Engagement requiere que
   Vigor, Dedicación y Absorción promedien ≥5 **simultáneamente** (no solo
   Vigor ≥5 — ese es un bug clásico, buscar `VIGOR_BINARY` vs `ALTO_ENGAGEMENT`
   si vuelve a aparecer). Alto Agotamiento: promedio EX1–EX4 ≥ 3.25.
2. **Agrupación correcta de ítems de Engagement (UWES-9):**
   Vigor=[WE1,WE2,WE5], Dedicación=[WE3,WE4,WE7], Absorción=[WE6,WE8,WE9].
   `R_DIMS` la tiene bien; `MAESTRO.textos` tiene los textos en orden
   secuencial 1-9 SIN esta agrupación (no confiar en su indexado para mapear
   texto↔código de sub-dimensión de Vigor).
3. **Bases con encabezados de texto literal en vez de códigos WE/EX** (común
   en archivos "Consolidado" que juntan varios años) — usar `itemTextCol`/
   `rItemIdx`/`rcpItemIdx`/`rMeanItem`, no `findCol`/`rColIdx`/`rcpColIdxFuzzy`/
   `rMean` a secas, en cualquier cálculo que dependa de ítems individuales
   (WE1-9/EX1-4 y también oppor/coach/feedb/soc/auto/optim/wp/cogn/emo/
   rolcon/hassle — **todos** los ítems de `R_DIMS` tienen el par
   `[código, texto]`, no solo Engagement/Agotamiento).
3b. **`rColIdx`/`rcpColIdxFuzzy` pueden enganchar una columna `*_BINARY`
   por error en la búsqueda aproximada** — ej. `rColIdx('DEDICATION')` sin
   coincidencia exacta caía a "DEDICATION_BINARY" porque
   `"dedicationbinary".startsWith("dedication")` es verdadero, dando un
   promedio sin sentido (0/1 en vez de escala 1-6). Ya se excluyen columnas
   `*_binary/*_bynary/*_binario` del nivel de búsqueda aproximada en ambas
   funciones — pero si se agrega una dimensión nueva a `R_DIMS`/
   `RCP_COL_ALIASES`, conviene darle explícitamente el alias en español
   (ej. `'Dedicación'`) para que resuelva por coincidencia exacta/normalizada
   (niveles 1-2) y ni siquiera necesite pasar por la búsqueda aproximada.
4. **Chart.js: la propiedad `order` de un dataset manda sobre el orden del
   arreglo `datasets` para decidir la posición izquierda→derecha dentro de un
   grupo de barras.** Si dos datasets tienen `order` explícito y distinto,
   reordenar el arreglo `datasets` NO alcanza para cambiar su posición visual
   — hay que invertir también los valores de `order`. (Bug real encontrado y
   corregido en el gráfico "Recursos y Demandas" de RCP.)
5. **`bar.base` en un plugin de Chart.js se extrapola a un pixel fuera de
   pantalla cuando el eje Y no arranca en 0.** Para dibujar texto en la base
   de una barra, usar `yScale.getPixelForValue(yScale.min)`, no `bar.base`.
6. **`requestAnimationFrame` se pausa/limita en pestañas sin foco.** No usarlo
   para esperas fijas en flujos largos (ej. exportación en lote) — usar
   `setTimeout` puro, que sigue corriendo sin importar el foco de la ventana.
7. **Exportación en lote de One Page (`ropBatchRenderOne`)**: arma un
   `filterSet` por cada medición seleccionada (BASE + comparativos) aplicando
   la clasificación que se está iterando — si solo usa `ROP.filters[0]`,
   vuelve a perder los comparativos.
8. **Etiquetas de comparación en One Page** (`ropGenerate`): hay que distinguir
   `titleLabel` (texto completo "medición · segmento", usado en título de
   página y leyenda de RD) de `chartLabel` (solo la medición/año, usado en las
   columnas de Engagement/Agotamiento/eNPS cuando hay segmentación — evita
   repetir el nombre del segmento en cada columna).

## Seguridad

- **Nunca insertar datos que vengan del Excel del usuario en `innerHTML` sin
  pasarlos por `escHtml()`** (o `escJsAttr()` si además van dentro de un
  atributo `onclick="fn('...')"`). Se hizo una limpieza completa de esto en
  esta sesión (~35 puntos corregidos) — al agregar código nuevo que renderice
  encabezados/valores de columna, clasificaciones, IDs o correos, revisar que
  siga el mismo patrón.
- La app es 100% cliente: no hay llamadas a servidor propio ni se envían datos
  a ningún lado. Los únicos recursos externos son las libs de CDN (código, no
  datos del usuario).
- No hay API keys ni credenciales en el repo. El repo es **público**.

## Git / despliegue

- Rama de trabajo de esta sesión: `claude/access-shared-chat-s1BrA`.
- **`main` es la rama que sirve GitHub Pages** (el link que usa el cliente).
  Cualquier fix debe terminar fusionado (fast-forward) a `main` para que se
  vea reflejado en el link real — no basta con pushear solo a la rama de
  trabajo. Patrón usado en esta sesión:
  ```
  git checkout main && git merge --ff-only claude/access-shared-chat-s1BrA && git push origin main
  git checkout claude/access-shared-chat-s1BrA
  ```
- El repo se llamó antes `Agente-IA`, ahora `Analizador-de-Encuestas` (GitHub
  redirige `git push` automáticamente, pero el nombre visible cambió).

## Metodología de verificación usada en esta sesión

Para cambios de lógica de cálculo o de gráficos, antes de darlos por buenos:

1. Validar sintaxis de todo el JS inline: extraer los `<script>` con un
   regex/Python y correr `node --check` sobre el conjunto.
2. Para bugs de detección de columnas: bajar el archivo real subido por el
   usuario a `/root/.claude/uploads/...`, inspeccionar encabezados con
   `openpyxl`, y simular la función de detección en Node con los encabezados
   reales antes de tocar el HTML — mucho más rápido que iterar dentro del
   navegador.
3. Para bugs de UI/gráficos: copiar el HTML al scratchpad, reemplazar los
   `<script src="https://cdn...">` por copias locales (`curl` una vez, cachear
   en el scratchpad), cargar con Playwright headless
   (`/opt/pw-browsers/chromium`), y ejecutar las funciones relevantes
   directamente vía `page.evaluate` — recordando activar `goSection`/`rGoSub`
   primero si el test depende de layout real.
4. Para dudas puntuales sobre comportamiento de Chart.js (ej. qué controla la
   posición de una barra), aislar en un HTML mínimo de 20 líneas con solo
   Chart.js — mucho más confiable y barato que instrumentar la app completa.
