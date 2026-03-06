# Reporte Completo: Auditoría y Fixes del Proyecto ADTi

**Fecha:** 2026-03-06
**Proyecto:** ADTi (Atracción de Talento Integrado)
**Repositorio:** Donixar/ruflo
**Branch de trabajo:** `claude/repo-usage-guide-WLWas`
**Autor:** Auditoría asistida por Claude (sesión interactiva)

---

## Contexto General

### ¿Qué es ADTi?

ADTi (**Atracción de Talento Integrado**) es un **portal web interno de reportería** para el área de Atracción de Talento de **CCU Chile**. Permite a los analistas de talento humano visualizar dashboards interactivos sobre:

- Stock de vacantes
- Prácticas profesionales
- Jóvenes profesionales
- Concursos internos
- Encuestas de satisfacción (interna y externa)
- Evaluación de practicantes
- Encuestas de salida
- Simulador IS6 (indicador de selección)
- Tablero ejecutivo consolidado

### Stack Técnico

| Componente | Tecnología |
|-----------|-----------|
| Frontend | React 19.2.4 + Vite 7.3.1 |
| Charts | Recharts 3.7.0 |
| Data parsing | PapaParse 5.5.3 (CSV), SheetJS/xlsx 0.18.5 (Excel) |
| Cifrado | AES-256-GCM via WebCrypto API |
| Key derivation | PBKDF2 (100,000 iterations, SHA-256) |
| Deploy | GitHub Pages (actions/deploy-pages@v4) |
| CI/CD | GitHub Actions (deploy.yml + encrypt-data.yml) |
| Integraciones | Google Apps Script (3 scripts), GitHub Contents API |

### ¿Qué hemos estado haciendo?

En esta sesión de trabajo realizamos una **auditoría completa del proyecto ADTi** que cubrió:

1. **Build y dependencias** — Verificar que compila, revisar chunks, dependencias, vulnerabilidades
2. **Datos y cifrado** — Auditar todos los archivos de datos, verificar que están cifrados correctamente
3. **Flujo de datos** — Trazar el pipeline completo desde archivo cifrado hasta gráfico Recharts
4. **Deploy y CI/CD** — Revisar los workflows de GitHub Actions
5. **Integraciones externas** — GitHub API, Google Apps Scripts, sistema de recordatorios
6. **Problemas identificados** — ESLint errors, código frágil, código muerto
7. **Fixes aplicados** — 4 fixes solicitados, 3 ya estaban resueltos, 1 aplicado

---

## 1. Estado del Build y Dependencias

### Build

Compila **sin errores** en ~7.5 segundos. Output en `dist/` con code-splitting por ruta (lazy loading).

**Chunks más pesados:**

| Chunk | Tamaño | Contenido |
|-------|--------|-----------|
| `xlsx-BojT3SgY.js` | 424 KB | SheetJS — la dependencia más pesada |
| `BarChart-Bm3ZYf4B.js` | 343 KB | Tree de Recharts BarChart |
| `index-EIXGjfRt.js` | 284 KB | Core React + Router + layout |
| `SimuladorIS6-Li1U60cO.js` | 136 KB | Datos IS6 embebidos en el código |

### Dependencias

6 de producción, 7 de desarrollo. **Ninguna desactualizada** — todas en versión Wanted:

| Dependencia | Versión |
|-------------|---------|
| react | 19.2.4 |
| react-dom | 19.2.4 |
| react-router-dom | 7.13.1 |
| recharts | 3.7.0 |
| papaparse | 5.5.3 |
| xlsx | 0.18.5 |
| vite | 7.3.1 |

**1 vulnerabilidad `high`** en `xlsx` (Prototype Pollution + ReDoS). No tiene fix disponible — es una **limitación conocida de SheetJS**.

### ESLint

**8 errores, 0 warnings** (con `--quiet`). Desglose:

| Archivo | Línea | Error | ¿Fue corregido? |
|---------|-------|-------|-----------------|
| PeriodFilter.jsx | 38, 48 | useMemo llamado condicionalmente (hooks rules violation) | No — fuera del scope de fixes solicitados |
| PeriodFilter.jsx | 152 | onChange definido pero no usado | No — fuera del scope |
| ReminderBanner.jsx | 56 | setVisible(true) dentro de effect body | No — fuera del scope |
| AdminRecordatorios.jsx | 132 | setError/setLoading dentro de effect body | No — fuera del scope |
| EncuestasSalida.jsx | 407 | `day` asignada pero nunca usada | **Sí — Fix 4** |
| JovenesProfesionales.jsx | 108 | `ingresosMes` y `salidasMes` nunca usadas | **Sí — Fix 4** |

---

## 2. Datos y Cifrado

### Estado de cifrado en `public/data/`

| Archivo | Cifrado | Módulo |
|---------|---------|--------|
| stock-vacantes.json | ✅ Sí | 1.1 |
| practicas.json | ✅ Sí | 1.2 |
| procesos-practicas.csv | ✅ Sí | 1.3 |
| retencion-practicas-2025.csv | ✅ Sí | 1.4 |
| jovenes-profesionales.json | ✅ Sí | 2.1 |
| concursos-internos.json | ✅ Sí | 2.3 |
| satisfaccion-externa.csv | ✅ Sí | 3.1 |
| satisfaccion-interna.csv | ✅ Sí | 3.2 |
| eval-practicantes.csv | ✅ Sí | 3.3 |
| encuesta-salida.csv | ✅ Sí | 4.1 |
| renuncias-voluntarias-2025.csv | ✅ Sí | 4.1 complementario |
| jp-baseline.json | ❌ No | Snapshot para detección de cambios JP |
| jp-previous.json | ❌ No | Snapshot periodo anterior JP |
| repositorio.json | ❌ No | Índice del repositorio (no contiene PII) |

### Formato del envelope cifrado

```json
{
  "iv": "94468a9a1d1327dc904ea864",
  "data": "base64..."
}
```

- **IV**: 12 bytes random, hex-encoded (24 chars)
- **Data**: ciphertext + 16-byte GCM auth tag, concatenados, base64-encoded
- **Key derivation**: PBKDF2 (100,000 iterations, SHA-256, salt fijo de 16 bytes hardcodeado)
- **Algoritmo**: AES-256-GCM via WebCrypto API

El script `encrypt-data.cjs` (Node.js, CI) y `crypto.js` (browser) comparten el mismo salt y parámetros PBKDF2.

### ¿Por qué jp-baseline.json y jp-previous.json NO están cifrados?

Comentado explícitamente en `encrypt-data.cjs`:

```javascript
// NOTE: jp-baseline.json and jp-previous.json are excluded — they are internal
// snapshots used by SyncADTi.js for change detection and must remain plaintext
// so the Google Apps Script can read them from the GitHub API.
```

`SyncADTi.js` (Google Apps Script) necesita leer estos archivos vía GitHub API para comparar el estado anterior de JP y detectar ingresos/salidas/movimientos.

---

## 3. Flujo de Datos

### Pipeline completo: archivo cifrado → gráfico Recharts

```
[Archivo cifrado en public/data/]
    │
    ▼ dataUrl('/data/archivo.json')          ← construye URL correcta para dev/prod
    │
    ▼ loadJSON() o loadCSV()                 ← en dataProcessing.js
    │                                     usa fetchAndDecrypt() internamente
    ▼ fetchAndDecrypt(url)                   ← en crypto.js
    │                                     fetch() → parse JSON envelope
    │                                     → decryptPayload(iv, data)
    │                                     → fallback a plaintext si no es envelope
    ▼ [Datos plaintext en memoria]
    │
    ▼ useMemo() en cada módulo              ← procesa, filtra, agrupa datos
    │
    ▼ Recharts components                    ← BarChart, PieChart, LineChart, RadarChart
    (config vía chartConfig.js)          (colores vía colors.js CHART_COLORS)
```

### DataLoader vs useEffect manual

`DataLoader.jsx` existe en `src/components/common/` como componente reutilizable con render props, pero **ningún módulo lo usa**. Es **código muerto**.

Los 11 módulos cargan datos manualmente con `useEffect`:

| Módulo | Método de carga |
|--------|----------------|
| StockVacantes.jsx | useEffect + loadJSON |
| Practicas.jsx | useEffect + loadJSON |
| ProcesosPracticas.jsx | useEffect + loadCSV |
| ConsolidadoPracticas2025.jsx | useEffect + loadCSV |
| JovenesProfesionales.jsx | useEffect + loadJSON |
| SimuladorIS6.jsx | datos embebidos (no carga archivo externo) |
| ConcursosInternos.jsx | useEffect + loadJSON |
| SatisfaccionExterna.jsx | useEffect + loadCSV |
| SatisfaccionInterna.jsx | useEffect + loadCSV |
| EvalPracticantes.jsx | useEffect + loadCSV |
| EncuestasSalida.jsx | useEffect + loadCSV |

`TableroEjecutivo.jsx` usa useEffect con `Promise.all` para cargar datos de 8 fuentes simultáneamente.

### CargaSemanal.jsx — Flujo detallado (729 líneas)

1. **Guard de acceso**: `canUpload(user)` verifica permisos (después de todos los hooks)
2. **Token GitHub**: El usuario ingresa su PAT (`ghp_...`), se guarda en `sessionStorage` vía `githubApi.js`
3. **Upload Excel**: Drag & drop o file picker. Parsea con `XLSX.read()` (SheetJS)
4. **Selección de hoja**: Filtra la hoja "Resumen", auto-selecciona la primera hoja de datos
5. **Validación**: `validateStockVacantes()` retorna semáforo rojo/amarillo/verde con lista de errores/warnings por fila
6. **Edición inline**: `EditableCell` permite editar celdas directamente. Al editar, `revalidateRows()` re-ejecuta la validación
7. **Publicación**: `pushStockVacantes()` hace GET (obtener SHA actual) + PUT (upsert) al GitHub Contents API
8. **Repositorio**: Post-publish, `updateRepositorio()` actualiza `repositorio.json` (mueve entrada anterior a "Histórico")
9. **Historial**: `getFileHistory()` muestra últimos 10 commits del archivo, con opción de revert vía `revertFile()`

---

## 4. Deploy y CI/CD

### `deploy.yml`

```
Trigger: push a master, o manual (workflow_dispatch)
Steps:
  1. Checkout + Node 20 + npm ci
  2. Encrypt data files (si ADTI_PASSWORD está seteado)
  3. npm run build (vite build)
  4. Upload artifact (dist/)
  5. Deploy to GitHub Pages (actions/deploy-pages@v4)
```

Usa el flujo moderno de GitHub Pages con artifacts (no rama `gh-pages`).

### `encrypt-data.yml`

```
Trigger: push a master en paths public/data/**
Steps:
  1. Checkout + Node 20
  2. Revisa cada archivo de la lista — detecta si es plaintext (no es envelope {iv,data})
  3. Si hay archivos plaintext: ejecuta encrypt-data.cjs con ADTI_PASSWORD
  4. Commit automático: "chore: auto-encrypt data files after sync [skip ci]"
  5. git push
```

Esto garantiza que si `SyncADTi.js` pushea datos en plaintext, se cifran automáticamente antes de que el deploy los sirva.

### Secrets en GitHub Actions

Un solo secret: **`ADTI_PASSWORD`** — usado para derivar la key PBKDF2 para cifrar archivos. Es el mismo password que usan los analistas para loguearse en el portal (misma key = descifran correctamente).

---

## 5. Integraciones Externas

### githubApi.js — Endpoints

| Función | Endpoint | Método |
|---------|----------|--------|
| `getFileSHA()` | `/repos/{owner}/{repo}/contents/{path}` | GET |
| `getFileContent()` | `/repos/{owner}/{repo}/contents/{path}` | GET |
| `pushDataFile()` | `/repos/{owner}/{repo}/contents/{path}` | PUT |
| `pushStockVacantes()` | wrapper de pushDataFile | PUT |
| `getFileHistory()` | `/repos/{owner}/{repo}/commits?path=...` | GET |
| `getFileAtCommit()` | `/repos/{owner}/{repo}/contents/{path}?ref={sha}` | GET |
| `revertFile()` | GET old content + PUT current | GET+PUT |
| `updateRepositorio()` | GET+PUT repositorio.json | GET+PUT |

**Token**: vía `VITE_GITHUB_TOKEN` env var o ingreso manual en `sessionStorage`.

### Google Apps Scripts (3 scripts en `docs/google-apps-script/`)

#### SyncADTi.js — Sincronización diaria

- **Trigger**: 2 veces al día (7 AM y 7 PM hora Chile)
- Lee 7 Google Sheets (IDs hardcodeados) y convierte a CSV/JSON
- Pushea a GitHub vía Contents API
- Manejo especial para JP: compara vs `jp-previous.json` para detectar cambios
- Maneja 3 hojas de Concursos Internos (Concursos, Métricas, Degloce por área)

#### FeedbackADTi.js — Feedback del equipo

- Web App `doPost`: recibe JSON `{nombre, email, tipo, comentario, pagina}` y appendea a Google Sheet

#### ReminderTracker.js — Recordatorios Concursos

- `doPost`: registra confirmación de analista
- `doGet`: retorna status semanal de confirmaciones
- Valida que el email esté en lista de usuarios autorizados

### Sistema de Recordatorios — Flujo completo

1. `ReminderBanner.jsx` se muestra solo **lunes y jueves**
2. Verifica `localStorage` (ya confirmó hoy?) y `sessionStorage` (diferido esta sesión?)
3. Excluye managers (`EXCLUDED_EMAILS` array)
4. **"Ya los actualicé"**: guarda en `localStorage` + POST fire-and-forget a ReminderTracker
5. **"Lo haré más tarde"**: guarda en `sessionStorage` (se limpia al cerrar browser)
6. `AdminRecordatorios.jsx`: panel admin que hace GET a ReminderTracker y muestra tabla de cumplimiento semanal
7. Config centralizada en `reminderConfig.js` (URL Apps Script + URL Google Sheet Concursos)

---

## 6. Problemas Identificados

### TODOs/FIXMEs en el código

**Ninguno.** No hay `TODO`, `FIXME`, `HACK`, `XXX`, ni `WORKAROUND` en el código fuente.

### Errores ESLint restantes (no corregidos — fuera del scope solicitado)

1. **PeriodFilter.jsx:38,48** — `useMemo` llamado condicionalmente. Violación de reglas de hooks.
2. **PeriodFilter.jsx:152** — `onChange` definido pero nunca usado.
3. **ReminderBanner.jsx:56** — `setVisible(true)` llamado directamente en el body del useEffect.
4. **AdminRecordatorios.jsx:132** — `setError`/`setLoading` llamados directamente en el body del useEffect.

### Partes frágiles del código

1. **SimuladorIS6.jsx** (136 KB gzipped, 689 líneas): 174+ registros IS6 hardcodeados en el JSX. Cualquier actualización requiere editar el archivo directamente.
2. **EncuestasSalida.jsx** (77 columnas): Descubre columnas dinámicamente por nombre. Cambios en headers del Google Sheet pueden romper la lógica silenciosamente.
3. **DataLoader.jsx es código muerto**: Existe pero ningún módulo lo importa. Todos reimplementan el mismo patrón `useEffect + loadCSV/loadJSON`.
4. **Token GitHub en el browser**: El PAT se guarda en `sessionStorage`. Visible en DevTools.
5. **Salt PBKDF2 fijo**: Hardcodeado en el código fuente (tanto en `crypto.js` como `encrypt-data.cjs`). Si alguien obtiene el password, la derivación de key es determinística.

---

## 7. Lo que hicimos — Fixes aplicados

Se solicitaron **4 fixes**. 3 de ellos ya estaban resueltos en el código actual. Solo **Fix 4** requirió cambios:

### Fix 1 (CRÍTICO) — SimuladorIS6.jsx — CustomTooltip dentro de TabEvo

**Ya resuelto.** `EvoTooltip` está definido en la línea 228, fuera y antes de `TabEvo` que empieza en línea 247. Sin cambios necesarios.

### Fix 2 (CRÍTICO) — SimuladorIS6.jsx — useMemo condicional en TabCambios

**Ya resuelto.** En `TabCambios` (línea 403), el `useMemo` está en línea 404 y el early return en línea 410. Los hooks se ejecutan antes de cualquier return condicional. Sin cambios necesarios.

### Fix 3 (ALTO) — TableroEjecutivo.jsx — Promise.all sin .catch()

**Ya resuelto.** El `.catch()` existe en línea 78:

```javascript
}).catch((err) => {
  console.error("Error cargando datos del tablero:", err);
  setError("No se pudieron cargar los datos. Intenta recargar la página.");
  setLoading(false);
});
```

Y la UI de error se renderiza en líneas 226-243 con botón de recarga. Sin cambios necesarios.

### Fix 4 (MENOR) — Variables no usadas — APLICADO ✅

**Cambio 1:** `EncuestasSalida.jsx:407` — Eliminé la variable `day` en `parseRenunciaDate`. Solo se usaban `month` y `year` en la línea 410 (`if (month >= 1 && month <= 12 && year >= 2000) return { year, month }`).

**Cambio 2:** `JovenesProfesionales.jsx:108` — Cambié el destructuring de:

```javascript
const { base, movimientos, ingresosMes, salidasMes, mesReporte } = data;
```

a:

```javascript
const { base, movimientos, mesReporte } = data;
```

Las variables `ingresosMes` y `salidasMes` del destructuring nunca se usaban — el módulo calcula `filteredIngresos` y `filteredSalidas` independientemente a partir de `base` (líneas 213 y 226), y las asigna como `stats.ingresosMes` y `stats.salidasMes` en el return del `useMemo` (líneas 270-271).

### Resultado

- **Branch**: `claude/analyze-project-summary-tveF6`
- **Commit**: `fix: remove unused variables flagged by ESLint`
- **Archivos modificados**: 2 (`EncuestasSalida.jsx`, `JovenesProfesionales.jsx`)
- **Build post-fix**: Pasa limpio sin errores ✅
- **Push**: Completado al remote ✅

---

## 8. Próximos pasos sugeridos

Según `PLAN.md` y la auditoría ESLint, quedan pendientes:

| # | Tarea | Prioridad | Impacto |
|---|-------|-----------|---------|
| 1 | **PeriodFilter.jsx** — 2 useMemo condicionales + 1 variable no usada (3 errores ESLint) | Alta | Violación de reglas de hooks — puede causar bugs en renders |
| 2 | **ReminderBanner.jsx** — setState en effect body (anti-pattern) | Media | Podría usar `useState(initializer)` o mover la lógica |
| 3 | **AdminRecordatorios.jsx** — setState en effect body (mismo anti-pattern) | Media | Mismo patrón que ReminderBanner |
| 4 | **SimuladorIS6.jsx** — Externalizar los 174+ registros IS6 a un archivo JSON en `public/data/` | Media | Reduce tamaño del bundle, facilita actualizaciones |
| 5 | **DataLoader.jsx** — Decidir si eliminar (código muerto) o migrar los 11 módulos para usarlo | Baja | Limpieza de código o estandarización |

---

## Arquitectura Visual del Proyecto

```
┌─────────────────────────────────────────────────────────────┐
│                     USUARIOS (Analistas CCU)                │
│                    Browser + Password Login                  │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                   GITHUB PAGES (Static Site)                │
│                                                             │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────────┐  │
│  │ React 19    │  │ React Router │  │ Recharts          │  │
│  │ + Vite 7    │  │ (lazy load)  │  │ (BarChart, Pie,   │  │
│  │             │  │              │  │  Line, Radar)     │  │
│  └──────┬──────┘  └──────────────┘  └───────────────────┘  │
│         │                                                   │
│         ▼                                                   │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ crypto.js — WebCrypto API                           │    │
│  │ AES-256-GCM + PBKDF2 (100k iter)                   │    │
│  │ Descifra datos en el cliente                        │    │
│  └──────┬──────────────────────────────────────────────┘    │
│         │                                                   │
│         ▼                                                   │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ public/data/ — Archivos cifrados                    │    │
│  │ {iv: "hex", data: "base64"} envelope                │    │
│  │ 11 archivos cifrados + 3 plaintext (snapshots)      │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ CargaSemanal.jsx — Upload Excel                     │    │
│  │ SheetJS parse → validate → GitHub Contents API PUT  │    │
│  └──────┬──────────────────────────────────────────────┘    │
└─────────┼───────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────────┐
│                    GITHUB (Repositorio)                      │
│                                                             │
│  ┌──────────────────┐  ┌────────────────────────────────┐   │
│  │ deploy.yml       │  │ encrypt-data.yml               │   │
│  │ Build + Deploy   │  │ Auto-cifra datos plaintext     │   │
│  │ a GitHub Pages   │  │ post-sync                      │   │
│  └──────────────────┘  └────────────────────────────────┘   │
│                                                             │
│  Secret: ADTI_PASSWORD (PBKDF2 key derivation)             │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│              GOOGLE APPS SCRIPT (3 scripts)                 │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ SyncADTi.js — Sync diario (7AM + 7PM)               │   │
│  │ Lee 7 Google Sheets → CSV/JSON → Push a GitHub      │   │
│  │ Detección de cambios JP (baseline vs previous)      │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ FeedbackADTi.js — Web App doPost                    │   │
│  │ Recibe feedback → Append a Google Sheet             │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ ReminderTracker.js — Recordatorios                  │   │
│  │ doPost: registra confirmación                       │   │
│  │ doGet: status semanal de cumplimiento               │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                             │
│  Fuente de datos: 7 Google Sheets (IDs hardcodeados)       │
└─────────────────────────────────────────────────────────────┘
```

---

## Glosario

| Término | Significado |
|---------|-------------|
| **ADTi** | Atracción de Talento Integrado — nombre del portal |
| **CCU** | Compañía de Cervecerías Unidas — empresa chilena |
| **IS6** | Indicador de Selección 6 — métrica interna de RRHH |
| **JP** | Jóvenes Profesionales — programa de talento joven |
| **PAT** | Personal Access Token — token de GitHub |
| **GCM** | Galois/Counter Mode — modo de cifrado autenticado |
| **PBKDF2** | Password-Based Key Derivation Function 2 |
| **PII** | Personally Identifiable Information |
| **SyncADTi** | Script de Google Apps que sincroniza datos diariamente |
