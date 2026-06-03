# CLAUDE.md — DSE Selector Tool

## 1. Descripción general

**DSE Selector Tool** es una herramienta interna de ventas para Kodland, una escuela de programación para niños. Ayuda a los TCMs (vendedores) a recomendar el curso y grupo correcto a cada estudiante, priorizando grupos activos cercanos en fecha y grupos posibles (por demanda) que coincidan con la disponibilidad horaria del cliente.

- **Usuarios:** TCMs de LatAm, Brazil y MENA
- **Despliegue:** Sitio estático en GitHub Pages — push a `main` dispara el workflow `.github/workflows/static.yml` y publica automáticamente
- **Sin build system:** No hay npm, webpack ni dependencias locales. La app entera vive en `index.html`

---

## 2. Arquitectura

Un solo archivo `index.html` con tres bloques:

| Bloque | Líneas aprox. | Contenido |
|---|---|---|
| `<style>` | 14–214 | CSS propio + Tailwind CDN |
| `<body>` HTML | 217–512 | Pantalla de región, layout de 3 columnas, modales |
| `<script>` | 513–2680 | Toda la lógica JS |

**CDNs externas (cargadas desde el HTML):**
- Tailwind CSS (CDN)
- Inter font (Google Fonts)
- Font Awesome 6.5.1

**Desarrollo local:**
```bash
# Opción 1: abrir directo en navegador
start index.html

# Opción 2: servidor local (evita problemas CORS con fetch)
python -m http.server 8080
# o
npx serve .
```

---

## 3. Regiones soportadas

La app comienza mostrando una pantalla de selección de región. Al elegir, se llama `selectRegion(regionKey)` que configura todas las fuentes de datos y traducciones.

### LatAm
- **Idioma:** Español
- **Zona horaria del sheet:** GMT-5 (`sheetTimezoneOffset: -5`)
- **Zona horaria default del cliente:** UTC-5
- **Grupos activos:** Sí (única región con grupos activos)
- **Grupos posibles:** Sí
- **Hoja activos:** `SHEET_ID` / `SHEET_GID = "2084392899"`
- **Hoja posibles:** `POSSIBLE_GROUPS_SHEET_ID` / `POSSIBLE_GROUPS_GID = "1601359885"`
- **TCMs:** Hoja central `gid=1478651946`
- **Cursos:** Hoja central `gid=0`
- **Apps Script:** deployment propio (ver sección 5)
- **SEN:** "Estudiante SEN"

### Brazil
- **Idioma:** Portugués
- **Zona horaria del sheet:** GMT-3 (`sheetTimezoneOffset: -3`)
- **Zona horaria default del cliente:** UTC-3
- **Grupos activos:** No (`active: null`)
- **Grupos posibles:** Sí (sheet propio `13Ex-2PA9374...`, `gid=711101762`)
- **TCMs:** Hoja central `gid=2096614143`
- **Cursos:** Hoja central `gid=567922396`
- **Apps Script:** deployment compartido con MENA
- **SEN:** "Aluno NEE" (necesidades educativas especiales — terminología distinta)

### MENA (Turkey)
- **Idioma:** Turco
- **Zona horaria del sheet:** GMT+3 (`sheetTimezoneOffset: +3`)
- **Zona horaria default del cliente:** UTC+3
- **Grupos activos:** No (`active: null`)
- **Grupos posibles:** Sí (sheet propio `1Z5b3A5lzFp4...`, `gid=1962452144`)
- **TCMs:** Hoja central `gid=971800836`
- **Cursos:** Hoja central `gid=732223650`
- **Apps Script:** deployment compartido con Brazil
- **SEN:** "SEN Öğrencisi"

---

## 4. Fuente de datos central

**Sheet ID (editable, no publicado):** `1YNeyHpv12xZYU4ml1bzczoJN1myydZLBXGyWJtJbKBs`

Cada región tiene su propia pestaña de configuración de cursos. La estructura de columnas es:

| Col | Índice | Contenido |
|-----|--------|-----------|
| A | 0 | (ignorada) |
| B | 1 | (ignorada) |
| C | 2 | ID interno entre corchetes, ej: `[1116]` |
| D | 3 | Capacidad PRM (ej: `4`) |
| E | 4 | Capacidad Standard (ej: `14`) |
| F | 5 | (ignorada) |
| G | 6 | Nombre corto para el dropdown ("ShortCut"), ej: `Python (8-13)` |

La fila 1 es cabecera, la fila 2 es sub-cabecera (también ignorada). Los datos empiezan en fila 3 → `rows.slice(1)` en `fetchCourseMap()`.

### Cómo se construye el dropdown de cursos

```js
async function fetchCourseMap(url) {
  // 1. Lee CSV de la hoja central
  // 2. Parsea col C → ID, col D → prmCapacity, col E → standardCapacity, col G → name
  // 3. Agrupa por nombre (col G): cursos con mismo ShortCut comparten una opción
  // 4. Retorna: [{ ids: [1720, 1901], name: 'Roblox (8-9)', prmCapacity: 4, standardCapacity: 14 }]
}
```

El `value` de cada `<option>` es **una cadena de IDs separados por coma**, no un solo número:
```html
<option value="1720,1901">Roblox (8-9)</option>
<option value="1116">Python (8-13)</option>
```

### COURSE_CAPACITY_MAP

Un `Map` global indexado por **ID individual** (no por grupo). Se construye en `selectRegion()`:

```js
COURSE_CAPACITY_MAP = new Map();
courses.forEach(c =>
  c.ids.forEach(id =>
    COURSE_CAPACITY_MAP.set(id, { prmCapacity: c.prmCapacity, standardCapacity: c.standardCapacity })
  )
);
```

Se usa en `mapRowsToPossibleGroups()` para determinar la capacidad de cada grupo posible:
```js
const capEntry = COURSE_CAPACITY_MAP.get(idRef);
const capacity = isPremium
  ? (capEntry?.prmCapacity ?? 4)    // fallback: 4
  : (capEntry?.standardCapacity ?? 14); // fallback: 14
```

**No hardcodear capacidades.** Si un curso nuevo entra, solo hay que agregarlo al sheet central con sus valores D y E.

### Cursos con múltiples IDs (casos reales)

Cuando Kodland migra un curso a un nuevo ID pero mantiene grupos con el ID antiguo, ambos IDs conviven bajo el mismo nombre en el dropdown:

| ShortCut | IDs |
|---|---|
| `Roblox (8-9)` | 1720, 1901 |
| `3D Modeling (13+)` | 1219 (antiguo FWD PRO), 1936 |

Para agregar un nuevo curso: añadir fila en la hoja central con los valores en C, D, E y G. El dropdown se actualiza automáticamente en el siguiente refresh.

---

## 5. Google Apps Scripts

Hay **dos deployments**. Los URLs están en `REGION_CONFIG[region].salesScriptUrl`.

| Deployment | Regiones | Notas |
|---|---|---|
| `AKfycbxkhc55bm...` | LatAm | Deployment propio, **no modificar** — tiene historial de producción |
| `AKfycbzsh1J1Mt6...` | Brazil + MENA | Deployment compartido |

### Payload enviado (POST, modo `no-cors`)

```js
{
  timestamp: new Date().toISOString(),
  region: "latam" | "brazil" | "turkey",
  amo_crm_link: "https://kodland.amocrm.ru/leads/detail/12345",
  tcm_name: "L382 — Nombre Apellido",
  student_age: "10",
  course_type: "Python (8-13)",     // texto del <option>, no el value
  group_type: "regular" | "premium",
  is_sen: true | false,
  preferred_schedule_text: "Lun 13:00, Mié 18:00",
  selected_groups: "GRUPO1 | GRUPO2",
  pitch: "...",
  cluster_id: "ABC-123"             // vacío si no hay grupo posible seleccionado
}
```

El script registra en la pestaña correspondiente (`Data-LatAm`, `Data-Brazil`, `Data-MENA`). La estructura de columnas de esas pestañas la controla el Apps Script — si cambia, hay que actualizar el script, no el frontend.

---

## 6. Flujos principales

### 6.1 Carga de región

```
selectRegion(regionKey)
  ├── Configura DATA_SOURCES, TCM_LIST_CSV, SHEET_OFFSET
  ├── Oculta selector de región, muestra #app-container
  ├── applyTranslations(regionKey)         → actualiza todos los textos del DOM
  ├── fetchCourseMap(courseListCsv)
  │     ├── Construye COURSE_CAPACITY_MAP
  │     └── populateCourseDropdown(courses)
  ├── populateTimezones(regionKey)          → llena el <select> de zonas horarias
  ├── fetchTCMList()                        → carga TCM_LIST con TCMs activos
  └── fetchCourseData()                     → carga LIVE_COURSE_DATA + LIVE_POSSIBLE_GROUPS
```

### 6.2 "Sugerir Grupos" vs "Ver todos los grupos"

| Botón | Función | Disponibilidad requerida |
|---|---|---|
| `🔍 Sugerir Grupos` | `diagnoseAndMatch()` | Sí — error si `preferredSchedule.length === 0` |
| `📋 Ver todos los grupos` | `diagnoseAndMatch(true)` | No — omite esa validación, muestra banner amarillo |

Ambos botones validan: AMO link, edad (5-17), curso seleccionado, TCM seleccionado.

Si se usa "Ver todos los grupos" y el TCM selecciona un **grupo posible**, el botón de guardar está bloqueado hasta que marque disponibilidad (ver sección 7).

### 6.3 Matching y scoring

```
diagnoseAndMatch()
  ├── Filtra LIVE_COURSE_DATA por courseId (allowedIds), edad, formato → filteredActive
  ├── Filtra LIVE_POSSIBLE_GROUPS por courseId, edad, formato → filteredPossible
  ├── Para cada grupo:
  │     ├── calculatePriority()
  │     │     ├── Activos: score = 1000 - días_hasta_inicio; -500 si lleno
  │     │     └── Posibles: usa clusterScore de columna M del sheet
  │     └── calculateScheduleMatchScore()
  │           ├── Convierte preferencia del cliente a zona del sheet (SHEET_OFFSET)
  │           ├── Exacta (mismo día + ≤30 min): 1000 pts
  │           ├── Misma hora distinto día: 800 pts
  │           ├── Mismo día distinta hora (≤120 min): 600 pts
  │           └── Si está lleno y hay coincidencia: fija en 100 pts
  └── Ordena: si hay disponibilidad → por scheduleScore; si no → por priority
```

### 6.4 Vista de tarjetas vs vista matriz

Los grupos posibles tienen dos vistas alternadas con botones toggle:

- **Tarjetas** (default): top 3 con botón "Ver todos" → modal scrollable
- **Matriz**: modal fullscreen (90vw × 85vh), tabla días × horas en zona horaria del cliente

El badge animado `#matrix-hint` ("↗ ¡Ver todos!") aparece cuando hay resultados posibles. Se oculta al primer clic en el botón de matriz y se resetea al empezar un nuevo estudiante.

La vista matriz muestra **todos** los grupos posibles del curso/edad/formato, sin filtro de horario. Las celdas ocupadas tienen escala de calor por ocupación:

| Clase CSS | Ocupación | Color |
|---|---|---|
| `.heat-0` | 0% | Gris claro |
| `.heat-11` | 1–25% | Azul Kodland |
| `.heat-26` | 26–35% | Verde |
| `.heat-36` | 36–49% | Amarillo |
| `.heat-50` | ≥50% | Naranja |

Las celdas que coinciden con la disponibilidad del cliente muestran rayas diagonales azules (`.has-availability`). Clic en una celda ocupada → modal de detalle con lista de grupos en ese horario.

### 6.5 Guardar venta → reset

```
submitSale()
  ├── Valida: grupo seleccionado, TCM seleccionado
  ├── Valida: si hay grupo posible seleccionado → preferredSchedule.length > 0
  ├── POST a salesScriptUrl (no-cors, sin esperar respuesta)
  ├── Si hay grupo posible con semanticId:
  │     ├── Copia "Selector Tool - Cluster [ID]" al portapapeles
  │     ├── Inyecta Cluster ID en el DOM del resumen
  │     └── alert() con el texto copiado
  └── setTimeout(resetForNextStudent, 2000)
        └── Limpia todo: selección, preferredSchedule, formulario, toasts, hint badge
```

---

## 7. Lógica crítica — no romper

### 7.1 sheetTimezoneOffset por región

Cada región almacena sus horarios en su propia zona horaria:
- LatAm: GMT-5 (Colombia)
- Brazil: GMT-3 (Brasília)
- MENA: GMT+3 (Turquía)

La variable global `SHEET_OFFSET` se actualiza en `selectRegion()`:
```js
SHEET_OFFSET = config.sheetTimezoneOffset;
```

Las funciones de conversión usan `SHEET_OFFSET`, no la constante `COLOMBIA_OFFSET = -5` (que se mantiene solo como referencia). Si se añade una región nueva, hay que definir su `sheetTimezoneOffset` en `REGION_CONFIG`.

```js
// Sheet → cliente (para mostrar la hora al TCM)
const diff = clientOffset - SHEET_OFFSET;

// Cliente → sheet (para calcular coincidencias con preferredSchedule)
const diff = SHEET_OFFSET - clientOffset;
```

### 7.2 COURSE_CAPACITY_MAP indexado por ID individual

El map siempre tiene una entrada por ID numérico, aunque el dropdown agrupe varios IDs bajo un mismo nombre. Esto permite que `mapRowsToPossibleGroups()` haga `COURSE_CAPACITY_MAP.get(idRef)` con el ID individual que viene del sheet.

### 7.3 allowedIds como array

El `value` del `<select id="course_type">` es una cadena CSV de IDs (`"1720,1901"`). Siempre parsear así:
```js
const allowedIds = courseType ? courseType.split(',').map(Number).filter(Boolean) : [];
// Uso: allowedIds.includes(c.courseId)
```

Nunca usar `parseInt(courseType)` — rompe el soporte multi-ID.

### 7.4 Cluster ID — solo aparece después de guardar

El Cluster ID (`semanticId`) **no se muestra** en el resumen de venta hasta que el guardado es exitoso. Se inyecta en el DOM dentro del `try` de `submitSale()`, después del POST. No moverlo a `updateSaleSummary()`.

### 7.5 Grupos posibles requieren disponibilidad para guardar

```js
// En submitSale():
if (selectedCourses.some(c => c.type === 'possible') && preferredSchedule.length === 0) {
  errors.push(T.msgPossibleNeedsSchedule);
}
```

Esto aplica incluso si el TCM usó "Ver todos los grupos" (que no requiere disponibilidad para buscar, pero sí para guardar un posible).

### 7.6 No modificar el script de LatAm

El Apps Script de LatAm (`AKfycbxkhc55bm...`) tiene historial de producción y lógica propia. Para cambios en la lógica de guardado, coordinar con quien administra el Apps Script. El URL está en `REGION_CONFIG.latam.salesScriptUrl`.

---

## 8. Sistema de traducciones

### Estructura

```js
const TRANSLATIONS = {
  latam:  { /* ~80 claves */ },
  brazil: { /* ~80 claves */ },
  turkey: { /* ~80 claves */ },
};
let T = TRANSLATIONS.latam; // shorthand global, se actualiza en applyTranslations()
```

Las claves son strings o funciones arrow:
```js
btnSuggest: '🔍 Sugerir Grupos',           // string simple
btnSeeAllActive: n => `📋 Ver todos los ${n} grupos activos`,  // función
alertSaved: clip => `✅ ¡Registro Guardado!\n\n📋 COPIADO:\n"${clip}"`,
```

### Agregar un texto nuevo

1. Añadir la clave en los tres bloques de TRANSLATIONS (latam, brazil, turkey)
2. Si se usa en el DOM inicial, actualizar `applyTranslations()`:
   ```js
   el('mi-elemento').textContent = T.miClave;
   ```
3. Si se usa solo en JS, referenciarlo directamente como `T.miClave`

### applyTranslations()

Se llama una vez al seleccionar región. Actualiza **todo el texto visible** del DOM: headers, labels, placeholders, botones, opciones de select, cabeceras de tabla de horarios, títulos de modales. Si se añade un elemento nuevo al HTML con texto traducible, agregar la línea correspondiente aquí.

---

## 9. Vista Matriz

### Cómo funciona

1. `openMatrixView()` → llama `getMatrixGroups()` → filtra `LIVE_POSSIBLE_GROUPS` por curso/edad/formato (sin filtro de horario)
2. `renderPossibleMatrix(groups)` → convierte cada grupo de zona del sheet a zona del cliente, construye la tabla días × horas

### Coordenadas

- `SPANISH_TO_COL_IDX`: nombre de día en español → índice 0-6 (lunes=0)
- `DAY_KEY_TO_IDX`: clave del modal de horario (`mon`, `tue`…) → índice 0-6
- La celda `(dayIdx, hour)` en hora del cliente se construye: `colHour + (clientOffset - SHEET_OFFSET)`

### Mapa de datos

`matrixCellData` es un `Map<"dayIdx-hour", group[]>` que se reconstruye en cada `renderPossibleMatrix()`. Al hacer clic en una celda ocupada, se lee esta estructura para mostrar el modal de detalle.

### Coincidencias (disponibilidad del cliente)

La detección de disponibilidad en la matriz es directa (sin conversión adicional): se compara `DAY_KEY_TO_IDX[pref.day] === clientDayIdx && pref.hour === clientHour`. Las celdas que coinciden reciben `.has-availability` (rayas diagonales azules).

**Importante:** Las clases de calor usan `background-color` (no `background` shorthand) para que `.has-availability` — que usa `background-image` — pueda superponerse. Si cambias las clases de calor, mantener esa distinción.

---

## 10. Consideraciones al modificar

### Agregar un curso nuevo
Solo actualizar la **hoja central** con una fila nueva (cols C, D, E, G). El dropdown y el COURSE_CAPACITY_MAP se actualizan automáticamente al recargar. No hay nada que cambiar en el código salvo que el curso requiera lógica especial de detección en `extractCourseType()` o `inferInterest()`.

### Agregar una región nueva
1. Añadir entrada en `REGION_CONFIG` con: `sheetTimezoneOffset`, `active`, `possible`, `tcmCsv`, `courseListCsv`, `salesScriptUrl`
2. Añadir bloque en `TRANSLATIONS` (copiar estructura de latam/brazil/turkey)
3. Añadir tarjeta de región en el HTML (`#region-selector`)
4. Ajustar `populateTimezones()` si la región tiene zonas horarias distintas
5. Crear (o reutilizar) un Google Apps Script para recibir las ventas
6. Ajustar el grid de la región si no tiene grupos activos (`active: null`)

### Columnas del sheet de grupos activos
Se resuelven por nombre de cabecera en `mapRowsToCourses()` usando `findCol()`:
```js
const findCol = (name) => headers.findIndex(h => h.toLowerCase().includes(name.toLowerCase()));
const idx_capacity = findCol("capacity");
const idx_time = findCol("first_lesson_time_gmt_5");
```
Si el sheet renombra una columna, actualizar el string de búsqueda aquí. No usar índices numéricos fijos para grupos activos.

### Columnas del sheet de grupos posibles
`mapRowsToPossibleGroups()` usa índices **fijos por posición**, no por nombre de cabecera:

| Columna | Índice | Campo |
|---|---|---|
| A | 0 | Nombre del curso |
| B | 1 | Formato (regular/prm) |
| C | 2 | Día (ej: "Mon") |
| D | 3 | Hora (ej: "13:00") |
| E | 4 | Estudiantes actuales |
| G | 6 | Rango de edad |
| M | 12 | `clusterScore` (prioridad de negocio) |
| U | 20 | ID de referencia del curso |
| V | 21 | `semanticId` (Cluster ID) |

**Cualquier inserción de columna antes de la M rompe el `clusterScore`.** Si el sheet cambia estructura, actualizar los índices en `mapRowsToPossibleGroups()`.

### Sistema de notificaciones
`showSystemMessage(type, title, message)` crea toasts flotantes (esquina inferior derecha). Los tipos son `'success'`, `'warning'`, `'error'`. Éxito y advertencia se auto-descartan a los 5s; error requiere clic manual. Para limpiar todos los toasts: `document.getElementById('toast-container').innerHTML = ""`.

---

## 11. Comandos de desarrollo y despliegue

```bash
# Desarrollo local
python -m http.server 8080    # http://localhost:8080
# o
npx serve .

# Despliegue — automático vía GitHub Actions
git add index.html
git commit -m "descripción del cambio"
git push origin main
# El workflow .github/workflows/static.yml publica en GitHub Pages ~1 min después
```

No hay tests, linters ni build steps. El archivo se despliega tal cual.
