# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Proyecto

**SecadoGHI Unificado** — aplicación web monolítica para el cálculo y seguimiento de curvas de secado de revestimiento refractario en hornos industriales (GHI Smart Furnaces).

Un único archivo HTML autocontenido: `SecadoGHI_Unificado.html`. No hay build system, bundler, dependencias npm, ni servidor. Se abre directamente en el navegador.

## Ejecutar la aplicación

Abrir `SecadoGHI_Unificado.html` directamente en el navegador (doble clic o arrastrar). No requiere servidor local.

Todas las librerías se cargan desde CDN al abrir:
- Chart.js 4.4.1
- ExcelJS 4.3.0
- jsPDF 2.5.1 + jspdf-autotable 3.8.4
- Google Fonts (DM Sans, JetBrains Mono)

La aplicación funciona offline salvo las fuentes y librerías CDN.

## Arquitectura

### Flujo de datos entre tabs

```
Tab 1 (Configuración)
  └─ calculateProfile(config) → appState.segments, appState.totalHours, appState.alerts
       └─ renderRampTable() → Tab 2 (Curva de Secado)
            └─ generarRegistroDesdeRampa() → appState.registroRows → Tab 3 (Registro)
                 └─ cargarCSV() → mapea lecturas de termopares a registroRows
                      └─ exportExcel() / exportPDF()
```

### Estado global (`appState`)

Todo el estado vive en el objeto `appState` (definido al inicio del `<script>`):
- `appState.segments` — array de tramos calculados (RAMPA / MESETA)
- `appState.totalHours` — duración total en horas decimales
- `appState.alerts` — alertas de seguridad generadas por `calculateProfile`
- `appState.config` — configuración de material y geometría del último cálculo
- `appState.registroRows` — filas de la tabla hora-hora (fecha, hora, Tª teórica, TC1..10, observaciones)
- `appState.tcNames`, `appState.tcCount` — nombres y número de termopares activos
- `appState.selectedMaterial` — material seleccionado de `HormigonesDatabase`

El estado se persiste en `localStorage` bajo la clave `ghiSecadoProject_v2`.

### Base de datos de materiales

`HormigonesDatabase` — array de 268 objetos inline en el HTML. Cada material:
```js
{ name, brand, type, tipoHormigon, rate, water, hasFibers, hasAntiWetting, temperatura, al2o3, densidad, codigo }
```
`type` es CC | LCC | ULCC | SOL_GEL. Este campo controla los factores de cálculo.

### Motor de cálculo (`calculateProfile`)

Genera el cronograma de secado seguro. Lógica clave:
- Divide cada rampa en sub-segmentos al cruzar los boundaries críticos: 110°C, 200°C, 350°C
- Aplica `getSafeRateAtTemp()` en el punto medio de cada sub-segmento
- Factores acumulativos: LCC/ULCC ×0.8, anti-wetting ×0.6, fibras ×1.5 (solo >200°C)
- Zona crítica 110–350°C: velocidad máxima 15°C/h independientemente del material
- Hold times calculados por `calculateHoldDuration()`: base = (espesor_mm/25.4) × factor(T), más corrección difusiva si hay capa aislante

### Carga de CSV (`cargarCSV`)

Formato esperado: separador `;`, saltar 2 líneas de cabecera, columna 0 = timestamp `YYYY-MM-DD HH:mm:ss.fff`, columna 1 = temperatura decimal con punto. Usa búsqueda binaria sobre timestamps ordenados; tolerancia de mapeo: ±30 minutos.

## Diseño GHI

Paleta oficial (no cambiar sin autorización):
- `--ghi-red: #C0022C` — botones primarios, badges activos, alertas
- `--ghi-graphite: #1F1F1F` — header, cabeceras de tabla
- `--ghi-gray-100/200/300` — fondos, bordes, celdas auto-calculadas

Celdas de valor calculado automáticamente llevan clase `auto-calc` (fondo gris, fuente mono, no editables por el usuario).

## Archivos relacionados (fuera de este directorio)

| Ruta | Descripción |
|---|---|
| `..\Lasso\SecadosGHI\Calculadora_GHI_HMI.html` | App original de Lasso (React) — fuente del motor de cálculo |
| `..\Lasso\SecadosGHI\hormigones_database.js` | Fuente original de `HormigonesDatabase` (268 materiales) |
| `..\Ballen\vaporeon\curva_secado_v10 (7).html` | App original de Ballen — fuente de la lógica hora-hora y CSV |
