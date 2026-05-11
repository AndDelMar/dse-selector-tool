# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Descripción del proyecto

**DSE Selector Tool** — herramienta interna de ventas para una escuela de programación latinoamericana (Kodland). Ayuda a los TCMs (vendedores) a recomendar el curso correcto a cada estudiante. Se despliega como sitio estático en GitHub Pages.

## Comandos

No hay sistema de build, bundle ni dependencias npm. La app entera vive en `index.html`.

**Desarrollo local:**
```
# Abrir directamente en el navegador
start index.html

# O con un servidor local (evita problemas CORS en fetch)
python -m http.server 8080
npx serve .
```

**Despliegue:** automático vía GitHub Actions (`.github/workflows/static.yml`) al hacer push a `main`. El workflow publica el contenido raíz en GitHub Pages.

## Arquitectura

Todo el código está en un único archivo `index.html` con tres secciones:

### 1. CSS (líneas 14–56)
Estilos propios con clases utilitarias complementando Tailwind CDN. Las clases clave del dominio:
- `.course-option.critical` / `.almost-full` / `.selected` — estados visuales de las tarjetas de curso
- `.connection-status` — indicador flotante de conexión con Google Sheets

### 2. HTML (líneas 59–312)
Diseño de 3 columnas en la fila superior:
- **Col 1** — Diagnóstico del estudiante (edad, TCM, zona horaria, tipo de curso, horario preferido)
- **Col 2** — Top 3 grupos activos (`#course_recommendation`)
- **Col 3** — Top 3 grupos posibles (`#possible_groups_list`)

Fila inferior: resumen de venta + botón de envío.

Tres modales: ver todos los activos, ver todos los posibles, selector de horario (tabla de días × horas).

### 3. JavaScript (líneas 313–fin)
Flujo principal:

```
fetchCourseData()
  ├── fetchWithCSV() → parseCSVRobust() → mapRowsToCourses()
  └── fetchWithGViz() → fallback si CSV falla

diagnoseAndMatch()
  ├── Filtrado por courseId (COURSE_MASTER_MAP), edad, tipo de grupo
  ├── calculatePriority()     — grupos activos: 1000 - días_hasta_inicio; posibles: clusterScore de columna M del sheet
  ├── calculateScheduleMatchScore() — penaliza grupos llenos, convierte zona horaria del cliente a Colombia (GMT-5)
  └── renderCourseList() → HTML de tarjetas con hora convertida a zona horaria del cliente

submitSale() → POST a SALES_SCRIPT_URL (Google Apps Script)
```

## Fuentes de datos externas

Todas las URLs están declaradas al inicio del `<script>` (líneas 320–344):

| Constante | Propósito |
|-----------|-----------|
| `SHEET_ID` + `SHEET_GID` | Google Sheet publicado — grupos activos |
| `POSSIBLE_GROUPS_SHEET_ID` + `POSSIBLE_GROUPS_GID` | Google Sheet publicado — grupos posibles |
| `SALES_SCRIPT_URL` | Google Apps Script — recibe los datos de venta (POST) |
| `TCM_LIST_CSV` | Google Sheet — lista de TCMs activos (código + nombre) |

La función `fetchCourseData()` intenta primero CSV Export directo (sin proxy) y, si falla, cae en GViz JSON con proxy `allorigins.win`.

## Lógica de negocio crítica

**`COURSE_MASTER_MAP`** — diccionario que mapea el valor del dropdown `course_type` a IDs numéricos internos de Kodland (ej. `"python": [1116]`). Es el filtro primario de matching. Si se añade un curso nuevo en el Sheet, primero debe registrarse aquí.

**Zona horaria:** el sistema convierte horarios de Colombia (GMT-5) a la zona del cliente usando el offset del `<select id="client_timezone">`. La función `convertClientToColombia()` hace la conversión inversa para el scoring. La constante global `COLOMBIA_OFFSET = -5` está declarada una sola vez al inicio del bloque de zona horaria.

**Grupos activos vs posibles:** la prioridad se calcula de forma distinta:
- Activos → urgencia por fecha de inicio (más próximo = más puntos), penalización si están llenos
- Posibles → `clusterScore` leído de la columna M (índice 12) del Sheet de grupos posibles

**Detección de curso:** `extractCourseType()` usa listas de sinónimos con exclusiones explícitas para evitar falsos positivos entre niveles (ej. "Python" no debe coincidir con "Python Pro"). El filtrado real en `diagnoseAndMatch()` usa `COURSE_MASTER_MAP` directamente por ID numérico.

## Qué tener en cuenta al modificar

- **Columnas del Sheet:** los índices de columnas se resuelven por nombre de cabecera en `mapRowsToCourses()` usando `findCol()`. Si el Sheet cambia el nombre de una columna, actualizar los strings de búsqueda ahí.
- **Columna M (índice 12) de grupos posibles:** es fija por posición, no por nombre. Cualquier inserción de columna en el Sheet antes de la M rompe el `clusterScore`.
- **Añadir un curso nuevo:** solo requiere registrarlo en `COURSE_MASTER_MAP` (mapea el valor del dropdown al ID numérico del Sheet).
