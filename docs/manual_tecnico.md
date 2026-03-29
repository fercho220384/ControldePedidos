# Manual Técnico — Control de Pedidos de Materiales

## Stack tecnológico

| Componente | Tecnología | Versión | CDN |
|-----------|-----------|---------|-----|
| UI | HTML5 + CSS3 + JS vanilla | — | — |
| Lectura/escritura Excel | SheetJS (xlsx) | 0.18.5 | cdnjs |
| Búsqueda similitud | Fuse.js | 7.0.0 | cdnjs |
| Persistencia | IndexedDB (idb wrapper) | — | cdnjs |
| Íconos | Sin iconos externos (CSS shapes) | — | — |

## Estructura del archivo index.html

```
index.html
├── <head>  Scripts CDN (SheetJS, Fuse.js)
├── <style> Todo el CSS (variables, componentes, vistas)
└── <body>
    ├── #app-shell         Contenedor principal
    │   ├── #sidebar       Navegación lateral
    │   └── #main-content  Área de contenido dinámico
    └── <script>
        ├── DB Module       IndexedDB setup y operaciones CRUD
        ├── Import Module   Parsers CSV/XLSX (materiales y SPM)
        ├── Similarity      Motor de búsqueda Fuse.js
        ├── Pedidos         Lógica de negocio de pedidos
        ├── Workflow        Máquina de estados (flujo aprobación)
        ├── ERP Sync        Cruce con datos SPM del ERP
        ├── Reports         Generador de XLSX
        └── Router          Navegación SPA sin framework
```

## Base de datos local (IndexedDB)

### Stores

#### `materiales` — Base de stock
```javascript
// Key: COD_MATERIAL
{
  cod_material: "ACAAAD385182",
  descripcion: "SEAT WRENCH 2 NOM H2 CHOKE",
  dep: "SC",
  alm: "H",
  saldo: 0,
  unidad: "E",
  stock_seg: 0,
  precio_estimado: 320.0,
  precio_contable: 195.19,
  nro_parte: "626964-01-00-00",
  cod_ubicacion: "AFE  KNE-2",
  fecha_ultimo_mov: "2014-06-11",
  _search_tokens: ["seat", "wrench", "nom", "choke"]  // generado al importar
}
```

#### `pedidos` — Pedidos del sistema
Ver `samples/template_pedido.json` para la estructura completa.

#### `erp_spm` — Datos importados del ERP
```javascript
// Key: mwl (ej. "MWL2600001")
{
  mwl: "MWL2600001",
  campo: "CRC-MANTTO",
  solicitante: "García, Juan",
  fecha_solicitud: "2026-03-28",
  oca: "CMPLD",
  estado_inicial: null,
  estado_complementario: "Cerrado",
  analista: "Guzman, Nelcy M.",
  requisicion: "MWL2600001, DESCRIPCION...",
  oc: "CHN-260001",
  recepcion: "15/03/2026",
  despacho: null,
  items: [...]
}
```

#### `config` — Configuración global
```javascript
{ key: "materiales_version", value: "2026-03-01", loaded_at: "..." }
{ key: "erp_version", value: "2026-03-08", loaded_at: "..." }
{ key: "ultimo_mwl", value: "MWL2600045" }
```

## Parser del formato SPM (ERP)

El archivo `Resumen_de_pedidos.xlsx` tiene un formato irregular:
- Filas de **cabecera de pedido**: columna 0 contiene `# Gst: XXXXXXX`
- Filas de **ítems**: las filas siguientes hasta la próxima cabecera

```javascript
function parseSPM(worksheet) {
  const rows = XLSX.utils.sheet_to_json(worksheet, { header: 1 });
  const pedidos = [];
  let currentPedido = null;

  for (const row of rows) {
    const col0 = String(row[0] || '');
    
    if (col0.startsWith('# Gst:')) {
      // Nueva cabecera de pedido
      if (currentPedido) pedidos.push(currentPedido);
      currentPedido = {
        gst: col0.replace('# Gst: ', '').trim(),
        mwl: row[2],           // Columna "Código - Cond"
        campo: row[4],          // Columna "Cant. Solic." (contiene el campo)
        fecha_solicitud: parseFechaSol(row[6]),
        oca: row[8],
        estado_inicial: row[9],
        estado_complementario: row[11],
        analista: row[12],
        requisicion: row[13],
        oc: row[15],
        recepcion: row[16],
        despacho: row[17],
        items: []
      };
    } else if (currentPedido && row[1]) {
      // Fila de ítem
      currentPedido.items.push({
        nro: row[1],
        codigo: row[2],
        descripcion: row[3],
        cantidad: row[4]
      });
    }
  }
  if (currentPedido) pedidos.push(currentPedido);
  return pedidos;
}
```

## Motor de similitud

```javascript
// Configuración Fuse.js para búsqueda en base de materiales
const fuseOptions = {
  keys: [
    { name: 'descripcion', weight: 0.7 },
    { name: 'nro_parte',   weight: 0.2 },
    { name: 'cod_material', weight: 0.1 }
  ],
  threshold: 0.4,     // 0 = match exacto, 1 = cualquier cosa
  minMatchCharLength: 3,
  includeScore: true,
  useExtendedSearch: true
};

// Bonus por stock disponible al rankear resultados
function rankWithStockBonus(results) {
  return results
    .map(r => ({
      ...r,
      finalScore: r.score - (r.item.saldo > 0 ? 0.1 : 0)
    }))
    .sort((a, b) => a.finalScore - b.finalScore)
    .slice(0, 5);
}
```

## Máquina de estados de pedidos

```
[Borrador]
    │ enviar()
    ▼
[Pendiente Aprobacion]
    │ aprobar()          │ rechazar(comentario)
    ▼                    ▼
[Aprobado]           [Devuelto]
    │ procesar()          │ (técnico puede editar y reenviar)
    ▼
[En Proceso Compra]
    │ (auto-actualización desde ERP)
    ▼
[estados ERP: 1..9 según Resumen_de_pedidos]
```

## Generador de reporte XLSX

El reporte de salida replica exactamente el formato de `Reporte_Pedidos_2026_03_Marzo.xlsx`:

```javascript
function generarReporteMensual(mes, anio) {
  const wb = XLSX.utils.book_new();
  
  // Sheet 1: Reportes (tabla limpia)
  const reportesData = buildReportesSheet(mes, anio);
  XLSX.utils.book_append_sheet(wb, reportesData, 'Reportes');
  
  // Sheet 2: Ajustes (forzado de estados)
  const ajustesData = buildAjustesSheet(mes, anio);
  XLSX.utils.book_append_sheet(wb, ajustesData, 'Ajustes');
  
  // Sheet 3: Informe (por responsable)
  const informeData = buildInformeSheet(mes, anio);
  XLSX.utils.book_append_sheet(wb, informeData, 'Informe');
  
  // Sheet 4: Estadística Mensual
  const statsData = buildEstadisticaSheet(mes, anio);
  XLSX.utils.book_append_sheet(wb, statsData, 'Estadistica Mensual');
  
  // Sheet 5: Dashboards (KPIs)
  const dashData = buildDashboardSheet(mes, anio);
  XLSX.utils.book_append_sheet(wb, dashData, 'Dashboards');
  
  XLSX.writeFile(wb, `Reporte_Pedidos_${anio}_${mes}.xlsx`);
}
```

## Variables de entorno / Configuración

Todo en el objeto `APP_CONFIG` al inicio del script:

```javascript
const APP_CONFIG = {
  DB_NAME: 'ControlPedidosDB',
  DB_VERSION: 1,
  MWL_PREFIX: 'MWL26',    // Año en curso
  AREAS: ['CRC-MANTTO', 'PCH-MANTTO', 'VGR', 'SCZ'],
  PRIORIDADES: ['BAJA', 'MEDIA', 'ALTA', 'URGENTE'],
  SIMILITUD_THRESHOLD: 0.4,
  SIMILITUD_MAX_RESULTS: 5
};
```

## Despliegue

La app es un único archivo `index.html`. Para distribuir:
1. Descargar `index.html` del repositorio GitHub
2. Compartir vía email, Drive, USB o cualquier medio
3. El usuario lo abre directamente en su navegador

Para actualizar: reemplazar el `index.html` con la nueva versión. Los datos en IndexedDB se preservan (están en el navegador, no en el archivo).
