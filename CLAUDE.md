# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Proyecto

**SecadoGHI Unificado** — aplicación web monolítica para el cálculo y seguimiento de curvas de secado de revestimiento refractario en hornos industriales (GHI Smart Furnaces).

Un único archivo HTML autocontenido: `SecadoGHI_Unificado.html`. No hay build system, bundler, dependencias npm, ni servidor. Se abre directamente en el navegador con doble clic.

## Ejecutar la aplicación

Abrir `SecadoGHI_Unificado.html` directamente en el navegador. No requiere servidor local.

Librerías cargadas desde CDN:
- Chart.js 4.4.1
- ExcelJS 4.3.0
- jsPDF 2.5.1 + jspdf-autotable 3.8.4
- Google Fonts (DM Sans, JetBrains Mono)

## Arquitectura

### Flujo de datos entre tabs

```
Tab 0 — Configuración
  └─ calculateProfile(config) → appState.segments, appState.totalHours, appState.alerts
       └─ renderRampTable() + generarRegistroDesdeRampa()

Tab 1 — Curva de Secado
  └─ Tabla de rampa editable + gráfica teórica
  └─ TC dinámicos: addTc() / removeTc() / removeTcAt(i)

Tab 2 — Registro Hora-Hora
  └─ cargarCSV() → mapea lecturas de termopares a appState.registroRows
  └─ exportExcel() / exportPDF() — siempre visibles, no requieren "Fin de Secado"
```

### Estado global (`appState`)

Todo el estado vive en el objeto `appState` al inicio del `<script>`:
- `segments` — array de tramos calculados (RAMPA / MESETA)
- `totalHours` — duración total en horas decimales
- `alerts` — alertas de seguridad generadas por `calculateProfile`
- `config` — configuración de material y geometría del último cálculo
- `registroRows` — filas hora-hora: `{ hora, fecha, horaStr, teoTemp, tcVals[], tNave, obs }`
- `tcNames`, `tcCount` — nombres y número de termopares (dinámico, sin límite fijo)
- `selectedMaterial` — material seleccionado de `HormigonesDatabase`

Persistencia en `localStorage` bajo la clave `ghiSecadoProject_v2`.

### Termopares dinámicos

Los TCs no tienen límite fijo. Se gestionan con:
- `addTc()` — añade un TC con nombre sugerido de `TC_DEFAULT_NAMES[]`
- `removeTc()` — elimina el último
- `removeTcAt(i)` — elimina el TC en posición `i`, preservando datos del resto
- `_syncTCCount()` — sincroniza el display, regenera tablas y extiende `registroRows`

Nombres por defecto: Tª Regulación, Tª Hormigón, Tª Ambiente, TC4, Tª Solera, Tª Bóveda, Tª Pared Izq, Tª Pared Der, Tª Entrada, Tª Salida.

### Base de datos de materiales

`HormigonesDatabase` — array de 268 objetos inline en el HTML (~líneas 459–4560). No modificar sin regenerar desde la fuente original. Cada material:
```js
{ name, brand, type, tipoHormigon, rate, water, hasFibers, hasAntiWetting, temperatura, al2o3, densidad, codigo }
```
`type`: CC | LCC | ULCC | SOL_GEL — controla los factores de cálculo.

### Motor de cálculo (`calculateProfile`)

- Divide cada rampa en sub-segmentos en los boundaries críticos: 110°C, 200°C, 350°C
- Aplica `getSafeRateAtTemp()` en el punto medio de cada sub-segmento
- Factores: LCC/ULCC ×0.8, anti-wetting ×0.6, fibras ×1.5 (solo >200°C)
- Zona crítica 110–350°C: máximo 15°C/h
- Hold times: `(espesor_mm/25.4) × factor(T)` + corrección difusiva si hay capa aislante

### Carga de CSV (`cargarCSV`)

Formato: separador `;`, saltar 2 líneas de cabecera, columna 0 = timestamp `YYYY-MM-DD HH:mm:ss.fff` (el espacio se sustituye por "T" antes de parsear), columna 1 = temperatura decimal (punto o coma). Búsqueda binaria; tolerancia ±30 min.

### Observaciones predefinidas

El `<datalist id="obs-predef">` al final del HTML define las sugerencias de la columna Observaciones: INICIO SECADO, FIN SECADO, INICIO/FIN RAMPA, PARADA PROGRAMADA, REANUDACIÓN, INCIDENCIA, FALLO TERMOPAR, CAMBIO DE TURNO, etc.

## Diseño GHI

Paleta oficial (no cambiar sin autorización):
- `--ghi-red: #C0022C` — botones primarios, badges activos, alertas
- `--ghi-graphite: #1F1F1F` — header, cabeceras de tabla
- `--ghi-gray-100/200/300` — fondos, bordes, celdas auto-calculadas

Celdas calculadas automáticamente llevan clase `auto-calc` (fondo gris, fuente mono, no editables).

## Atajos de teclado

- `Ctrl+Enter` — calcular curva (Tab 0)
- `Ctrl+S` — guardar proyecto
- `1` / `2` / `3` — navegar entre tabs (solo si el foco no está en un input)

## Archivos relacionados (fuera de este directorio)

| Ruta | Descripción |
|---|---|
| `..\Lasso\SecadosGHI\Calculadora_GHI_HMI.html` | App original de Lasso (React) — fuente del motor de cálculo |
| `..\Lasso\SecadosGHI\hormigones_database.js` | Fuente original de `HormigonesDatabase` (268 materiales) |
| `..\Ballen\vaporeon\curva_secado_v10 (7).html` | App original de Ballen — fuente de la lógica hora-hora y CSV |
