# 🏭 Control de Pedidos de Materiales

Sistema de registro, aprobación y seguimiento de pedidos de materiales para operaciones de campo.

## ¿Qué hace esta aplicación?

Aplicación HTML standalone (sin servidor, sin instalación) para gestionar el ciclo completo de pedidos de materiales:

```
Técnico crea pedido → Supervisor aprueba → Almacén genera solicitud → Adquisiciones gestiona compra
```

Con importación periódica del ERP para seguimiento automático de estados.

## Características principales

- **Registro de pedidos** con sugerencia automática de stock existente
- **Flujo de aprobación** en 4 etapas: Técnico → Supervisor → Almacén → Adquisiciones
- **Motor de similitud** que cruza descripciones nuevas con la base de 157k materiales
- **Seguimiento ERP** importando el Resumen de Pedidos (SPM) periódicamente
- **Tabla de forzado de estados** para ajuste manual de pedidos (MWL/Estado/Comentario)
- **Estadística mensual** por área operativa
- **Generador de reporte XLSX** en el formato del reporte mensual actual

## Estructura del repositorio

```
controldepedidos/
├── index.html              ← Aplicación completa (un solo archivo)
├── README.md               ← Este archivo
├── ARQUITECTURA.md         ← Diseño técnico completo
├── docs/
│   ├── manual_usuario.md   ← Guía para usuarios finales
│   └── manual_tecnico.md   ← Guía técnica y configuración
└── samples/
    ├── template_pedido.json        ← Estructura JSON de un pedido
    └── template_materiales_mini.csv ← Muestra CSV de materiales (10 filas)
```

## Cómo usar

1. Descargar `index.html`
2. Abrirlo en Chrome, Firefox o Edge (no requiere internet)
3. Primera vez: ir a **Configuración → Importar datos** y cargar:
   - `Base_de_datos_materiales_limpia.xlsx` (base de stock)
   - `Resumen_de_pedidos.xlsx` (pedidos ERP — actualizar periódicamente)
4. Seleccionar rol de usuario y comenzar

## Actualización de datos

| Archivo | Frecuencia recomendada | Dónde conseguirlo |
|---------|----------------------|-------------------|
| Base de materiales | Mensual o cuando haya cambios de stock | ERP → Export materiales |
| Resumen de pedidos | Semanal | ERP → Reporte SPM |

Ambos archivos se guardan también en Google Drive: `ControldePedidos/datos/`

## Tecnologías

- HTML5 + CSS3 + JavaScript vanilla (sin frameworks)
- [SheetJS](https://sheetjs.com/) para lectura/escritura de archivos Excel
- [Fuse.js](https://fusejs.io/) para búsqueda de similitud de materiales
- IndexedDB para persistencia local de datos

## Roles de usuario

| Rol | Función |
|-----|---------|
| **Técnico** | Crea y edita pedidos de su área |
| **Supervisor** | Aprueba o rechaza pedidos |
| **Almacén** | Gestiona solicitudes, importa ERP, ajusta estados |
| **Administrador** | Configuración, importación de datos, generación de reportes |

## Datos de la operación

- **Áreas:** CRC-MANTTO, PCH-MANTTO, VGR, y otras áreas operativas
- **Estados de pedido:** 12 estados desde registro hasta recepción
- **Estados Chaconet:** 15 estados del proceso de contratación
- **Base de materiales:** 157,355 ítems indexados

---

**Proyecto:** Fernando Noriega | **Iniciado:** Marzo 2026
