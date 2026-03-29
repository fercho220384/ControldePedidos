# Sistema de Control de Pedidos de Materiales
## Documento de Arquitectura y Diseño
**Versión:** 1.0 | **Fecha:** Marzo 2026

---

## 1. Visión General

Aplicación HTML standalone (sin backend) para el registro, aprobación y seguimiento de pedidos de materiales en operaciones de campo. Opera con datos importados desde el ERP (archivos CSV/XLSX) y persiste información localmente en el navegador.

---

## 2. Análisis de Archivos de Entrada

### 2.1 Base_de_datos_materiales_limpia.xlsx
- **Sheet:** `mtrListaMat`
- **Filas:** 157,355 | **Columnas:** 21
- **Campos clave:** `COD. MATERIAL`, `DESCRIPCION`, `DEP.`, `ALM.`, `SALDO`, `UNIDAD`, `STOCK SEG`, `PRECIO ESTIMADO`, `PRECIO CONTABLE`, `NRO. PARTE`, `COD. UBICACION`, `FECHA ULTIMO MOVIMIENTO`
- **Uso en app:** Cargada una vez como "base de stock". El motor de similitud la consulta para sugerir materiales existentes al crear un pedido.

### 2.2 Resumen_de_pedidos.xlsx
- **Sheet principal:** `SPM` (8,894 filas)
- **Campos clave:** `# Gst`, `Código - Cond`, `Descripción`, `Cant. Solic.`, `Estado Reg. Inicial`, `Estado Reg. Complementario`, `Requisición`, `O. Compra`, `Recepción`, `Despacho`
- **Sheet:** `Filtro Solicitudes por Area` — estadísticas por campo (CRC-MANTTO, PCH-MANTTO, VGR, etc.)
- **Uso en app:** Importado periódicamente para cruzar con pedidos registrados y actualizar estados de seguimiento.

### 2.3 Reporte_Pedidos_2026_03_Marzo.xlsx
- **Sheets:** `Dashboards`, `SPM` (enriquecido, 55 columnas), `Ajustes`, `Reportes`, `Informe`, `Estadistica Mensual`
- **Estados de Pedido (12):** `1 En Registro AAUTO/AREVI/INGRE`, `2 Sin Chaconet`, `3 En proceso Chaconet`, `4 Desierto`, `5 Con Orden de Compra`, `6 Entrega Parcial`, `7 Anulado`, `8 Contrato Maestro`, `8 Transferencia`, `9 Recibido`
- **Estados Chaconet (15):** desde `01 Aprobado por Gerencia` hasta `16 Cierre Manual`
- **Tabla Ajustes:** Input de forzado de estados — columnas: `MWL`, `Descripcion`, `Estado Pedido`, `Comentarios`
- **Estadística Mensual:** Tracking de pedidos por área operativa y mes
- **Uso en app:** Modelo base para el generador de reporte de salida. La tabla Ajustes se implementa como módulo de input en la app.

---

## 3. Arquitectura del Sistema

```
┌─────────────────────────────────────────────────────────┐
│              APLICACIÓN HTML (Single File App)           │
│                                                         │
│  ┌──────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │ Módulo   │  │  Motor de    │  │  Seguimiento     │  │
│  │ Pedidos  │  │  Similitud   │  │  ERP             │  │
│  └──────────┘  └──────────────┘  └──────────────────┘  │
│       │               │                   │             │
│  ┌────▼───────────────▼───────────────────▼──────────┐  │
│  │              LocalStorage (IndexedDB)              │  │
│  │  db_materiales | db_pedidos | db_ajustes          │  │
│  └────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
         │                              │
    Google Drive                     GitHub
    (ControlPedidos/)            (código versionado)
```

### 3.1 Decisiones de Arquitectura

| Decisión | Elección | Razón |
|----------|----------|-------|
| Backend | Ninguno | App standalone, sin servidor que mantener |
| Persistencia | LocalStorage / IndexedDB | Opera offline, datos en el navegador |
| Framework | HTML + JS vanilla | Sin dependencias, máxima portabilidad |
| Importación | File API (drag & drop) | El usuario arrastra el CSV/XLSX |
| Búsqueda similitud | Fuzzy matching client-side | Sin API externa, funciona offline |
| Versión código | GitHub | Control de cambios, colaboración |
| Archivos datos | Google Drive | Compartición dentro del equipo |

---

## 4. Módulos de la Aplicación

### Módulo 1 — Importación de Datos
- Drag & drop o selector de archivos para cargar `Base_de_datos_materiales_limpia.xlsx`
- Parser XLSX client-side (usando SheetJS / xlsx.js)
- Almacena en IndexedDB bajo clave `materiales_stock`
- Indicador de versión y fecha de la última carga
- Mismo flujo para cargar `Resumen_de_pedidos.xlsx`

### Módulo 2 — Registro de Pedido (Técnico)
**Vista: Técnico**
- Formulario para crear nuevo pedido (MWL auto-numerado)
- Campos: Descripción, Área/Campo, Solicitante, Fecha, Prioridad
- Tabla de ítems con: Código sugerido, Descripción, Cantidad, Unidad
- Al ingresar cada ítem → **Motor de Similitud** muestra top-3 materiales del stock con porcentaje de coincidencia
- El técnico puede seleccionar un ítem del stock existente o registrar como nuevo
- Estado inicial: `Borrador`

### Módulo 3 — Aprobación (Supervisor)
**Vista: Supervisor**
- Lista de pedidos en estado `Pendiente Aprobación`
- Vista detalle de cada pedido con ítems y sugerencias de stock usadas
- Acciones: Aprobar → pasa a `Aprobado` / Rechazar con comentario → `Devuelto`
- Notificación visual de pedidos pendientes

### Módulo 4 — Gestión de Solicitud de Compra (Almacén)
**Vista: Administrador de Almacén**
- Recibe pedidos en estado `Aprobado`
- Asigna número MWL definitivo
- Registra solicitud de compra formal
- Puede indicar si hay stock disponible (Transferencia interna)
- Envía a Adquisiciones → estado `En Proceso`
- Genera el PDF/Excel de solicitud

### Módulo 5 — Seguimiento ERP
**Vista: Almacén / Supervisor**
- Importa periódicamente el `Resumen_de_pedidos.xlsx` del ERP
- Cruza por número MWL con pedidos registrados en la app
- Actualiza automáticamente: Estado Chaconet, OC, Fecha recepción estimada
- Muestra timeline de cada pedido: Registro → Chaconet → OC → Recepción → Despacho

### Módulo 6 — Tabla de Ajuste de Estados (Forzado)
**Vista: Almacén**
- Tabla editable con columnas: `MWL`, `Descripción`, `Estado Pedido`, `Comentarios`
- Permite forzar manualmente el estado final de un pedido (ej. Anulado, Transferencia)
- Equivalente a la hoja `Ajustes` del reporte actual
- Se exporta como parte del reporte mensual

### Módulo 7 — Estadística Mensual y Reportes
**Vista: Todos / Gerencia**
- Dashboard con KPIs: Total pedidos, % completados, Promedio días OC, Pedidos por área
- Gráfico de evolución mensual por estado
- Filtros: Área operativa, Mes, Estado, Analista
- Generador de reporte XLSX estilo `Reporte_Pedidos_2026_03_Marzo.xlsx`:
  - Sheet `Reportes` — tabla limpia de pedidos
  - Sheet `Ajustes` — tabla forzado de estados
  - Sheet `Informe` — resumen por responsable
  - Sheet `Estadistica Mensual` — tracking histórico
  - Sheet `Dashboards` — KPIs y totales

---

## 5. Motor de Similitud de Materiales

Algoritmo de búsqueda fuzzy para sugerir materiales existentes:

```
Entrada: descripción ingresada por técnico
  1. Tokenizar (split por espacios, eliminar stopwords en español)
  2. Para cada material en stock:
     a. Calcular score = Jaccard(tokens_entrada, tokens_descripcion)
     b. Bonus si NRO. PARTE coincide exacto
     c. Bonus si SALDO > 0 (hay stock disponible)
  3. Retornar top-5 ordenados por score DESC
  4. Mostrar: Código, Descripción, Saldo, Almacén, Ubicación
```

Librerías candidatas: `fuse.js` (fuzzy search), `natural` (NLP básico)

---

## 6. Modelo de Datos (LocalStorage / IndexedDB)

### Tabla: `pedidos`
```json
{
  "id": "MWL2600001",
  "campo": "CRC-MANTTO",
  "solicitante": "García, Juan",
  "fecha_solicitud": "2026-03-28",
  "descripcion": "REPUESTOS BOMBA CENTRIFUGA",
  "prioridad": "ALTA",
  "estado": "Aprobado",
  "items": [
    {
      "nro": 1,
      "codigo_material": "ACAAAD385182",
      "descripcion": "SEAT WRENCH 2 NOM H2 CHOKE",
      "cantidad": 2,
      "unidad": "E",
      "stock_sugerido": true,
      "saldo_disponible": 5
    }
  ],
  "historial": [
    {"estado": "Borrador", "usuario": "García, Juan", "fecha": "2026-03-28"},
    {"estado": "Aprobado", "usuario": "Pérez, Ana", "fecha": "2026-03-29"}
  ],
  "mwl_erp": null,
  "oc": null,
  "fecha_oc": null,
  "estado_chaconet": null,
  "fecha_recepcion": null,
  "estado_ajuste_manual": null,
  "comentario_ajuste": null
}
```

### Tabla: `materiales_stock`
Carga directa del XLSX, indexada por `COD. MATERIAL`.

### Tabla: `ajustes_estado`
```json
{
  "mwl": "MWL2600001",
  "descripcion": "...",
  "estado_forzado": "7 Anulado",
  "comentario": "Reemplazado por MWL2600045"
}
```

---

## 7. Roles de Usuario

| Rol | Acceso | Funciones |
|-----|--------|-----------|
| Técnico | Vista Pedidos | Crear y editar pedidos propios |
| Supervisor | Vista Aprobación | Ver todos, aprobar/rechazar |
| Almacén | Vista Gestión + Seguimiento | Gestionar compras, importar ERP, ajustar estados |
| Administrador | Todo | Importar datos, generar reportes, configuración |

**Implementación:** Selección de rol simple al inicio (sin autenticación — app interna de confianza). El rol queda en sessionStorage.

---

## 8. Infraestructura y Conectores

### 8.1 Google Drive — Carpeta `ControlPedidos/`
Estructura de carpetas propuesta:
```
ControlPedidos/
├── datos/
│   ├── Base_de_datos_materiales_limpia.xlsx  ← Actualización periódica
│   └── Resumen_de_pedidos.xlsx               ← Actualización periódica
├── reportes/
│   ├── Reporte_2026_03_Marzo.xlsx
│   └── Reporte_2026_04_Abril.xlsx
└── app/
    └── index.html                             ← App HTML (también en GitHub)
```

**Conector MCP requerido:** Google Drive MCP (disponible en Claude.ai — ver sección 10)

### 8.2 GitHub — Repositorio `control-pedidos`
Estructura:
```
control-pedidos/
├── index.html           ← Aplicación principal (todo en un solo archivo)
├── README.md
├── ARQUITECTURA.md      ← Este documento
├── docs/
│   ├── manual_tecnico.md
│   └── manual_usuario.md
└── samples/
    ├── template_materiales.csv
    └── template_pedido.json
```

**Conector MCP requerido:** GitHub MCP (buscar en Claude.ai Settings → Connectors)

---

## 9. Plan de Fases de Desarrollo

### Fase 1 — App Base + Flujo de Aprobación (Semana 1-2)
- [ ] Estructura HTML + CSS de la aplicación
- [ ] Sistema de roles (selección de rol)
- [ ] Módulo de Pedidos: creación y edición
- [ ] Flujo Técnico → Supervisor → Almacén → Adquisiciones
- [ ] Persistencia en LocalStorage

### Fase 2 — Motor de Similitud + Stock (Semana 2-3)
- [ ] Importador de Base de Materiales (XLSX → IndexedDB)
- [ ] Motor de fuzzy matching
- [ ] Integración en formulario de pedido
- [ ] Indicador de stock disponible

### Fase 3 — Seguimiento ERP (Semana 3-4)
- [ ] Importador de Resumen de Pedidos ERP
- [ ] Motor de cruce MWL ↔ pedidos locales
- [ ] Actualización automática de estados
- [ ] Timeline visual de cada pedido
- [ ] Tabla de forzado de estados (Ajustes)

### Fase 4 — Reportes + Integración Drive/GitHub (Semana 4-5)
- [ ] Dashboard con KPIs y estadísticas
- [ ] Generador de reporte XLSX (formato actual)
- [ ] Estadística mensual por área
- [ ] Integración con Google Drive (upload/download)
- [ ] Publicación en GitHub

---

## 10. Skills y Conectores Necesarios

### Conectores MCP (configurar en Claude.ai Settings)
1. **Google Drive** — disponible en el registro, UUID: `37fb5d42-ef62-45d4-a12e-66551527a003`
   - Función: leer/escribir archivos en `ControlPedidos/`
2. **GitHub** — buscar en Settings → Connectors → "GitHub"
   - Función: crear/actualizar código en el repositorio

### Skills de Claude (ya disponibles)
- `xlsx` — creación y lectura de archivos Excel
- `frontend-design` — diseño de la interfaz HTML
- `file-reading` — lectura de archivos uploaded

### Skill a crear (recomendada)
- **`erp-data-processor`** — skill especializada en:
  - Parseo del formato irregular del SPM (filas de cabecera de pedido mezcladas con ítems)
  - Transformación del formato ERP → modelo de datos de la app
  - Generación del reporte XLSX en el formato exacto del reporte actual

---

## 11. Consideraciones Técnicas

### Rendimiento con 157k materiales
La base de datos de materiales tiene 157,355 filas. Para el motor de similitud:
- Usar **IndexedDB** (no localStorage, que tiene límite de 5MB)
- Indexar por `COD. MATERIAL` y por tokens de `DESCRIPCION`
- Búsqueda limitada al almacén del usuario por defecto
- Paginación en resultados: mostrar top-5 sugerencias

### Parsing del formato SPM
El archivo del ERP tiene un formato no estándar: filas de cabecera de pedido (con `# Gst:`) intercaladas con filas de ítems. Se necesita un parser específico que:
1. Detecte filas de pedido por presencia de `# Gst:` en columna 0
2. Asocie las filas siguientes como ítems de ese pedido
3. Extraiga el MWL de la columna `Código - Cond`

### Compatibilidad
- Navegadores: Chrome 90+, Firefox 88+, Edge 90+
- No requiere internet (opera 100% offline salvo para subir a Drive/GitHub)
- Tamaño estimado de la app: ~500KB (incluyendo SheetJS embebido)

---

*Documento generado: 28 de Marzo 2026*
*Proyecto: Sistema de Control de Pedidos de Materiales*
