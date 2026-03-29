# Manual de Usuario — Control de Pedidos de Materiales

## Acceso y roles

Al abrir la aplicación, seleccionás tu rol:

| Rol | Quién lo usa |
|-----|-------------|
| Técnico | Personal de campo que solicita materiales |
| Supervisor | Jefe de área que aprueba pedidos |
| Almacén | Administrador de almacén que gestiona compras |
| Administrador | Configuración y reportes generales |

---

## ROL: TÉCNICO — Cómo crear un pedido

### Paso 1 — Nuevo pedido
1. Ir a **Mis Pedidos → Nuevo Pedido**
2. Completar: Campo/Área, Descripción general del pedido, Prioridad

### Paso 2 — Agregar ítems
1. Hacer clic en **+ Agregar ítem**
2. Escribir la descripción del material que necesitás
3. El sistema buscará automáticamente en el stock y mostrará **hasta 5 sugerencias** de materiales similares disponibles
4. Si encontrás el material en stock → seleccionarlo (ahorra tiempo y costo)
5. Si no hay coincidencia → continuar escribiendo el ítem como nuevo requerimiento
6. Completar: Cantidad y Unidad

### Paso 3 — Enviar para aprobación
1. Revisar todos los ítems
2. Hacer clic en **Enviar al Supervisor**
3. El pedido cambia a estado `Pendiente Aprobación`

### Seguimiento de tu pedido
En **Mis Pedidos** podés ver el estado actual de cada pedido y el historial de movimientos.

---

## ROL: SUPERVISOR — Cómo aprobar pedidos

### Bandeja de aprobación
1. Ir a **Bandeja de Aprobación**
2. Verás todos los pedidos en estado `Pendiente Aprobación` de tu área
3. Hacer clic en un pedido para ver el detalle

### Aprobar o rechazar
- **Aprobar** → el pedido pasa a Almacén para gestión de compra
- **Devolver** → el pedido regresa al Técnico con tu comentario

---

## ROL: ALMACÉN — Gestión de compras

### Pedidos aprobados
1. Ir a **Gestión de Compras → Pedidos Aprobados**
2. Seleccionar un pedido para procesar
3. Verificar si algún ítem puede resolverse con **Transferencia de Stock**
4. Para ítems a comprar → registrar como nueva solicitud de compra (genera el MWL)
5. El pedido pasa a estado `En Proceso Compra`

### Importar actualización del ERP
Periódicamente (recomendado: semanal):
1. Ir a **Configuración → Importar Resumen ERP**
2. Cargar el archivo `Resumen_de_pedidos.xlsx` exportado del ERP
3. El sistema cruzará automáticamente los MWL y actualizará estados de Chaconet, OC, y recepciones

### Tabla de forzado de estados
Para pedidos que necesitan un estado manual (anulaciones, transferencias, contratos maestro):
1. Ir a **Seguimiento → Ajuste de Estados**
2. Buscar el MWL
3. Seleccionar el estado forzado y agregar comentario

---

## ROL: ADMINISTRADOR — Configuración y reportes

### Primera configuración
1. Ir a **Configuración → Base de Materiales**
2. Cargar `Base_de_datos_materiales_limpia.xlsx`
3. Esperar a que se procesen los 157k ítems (puede tomar 1-2 minutos)

### Generar reporte mensual
1. Ir a **Reportes → Reporte Mensual**
2. Seleccionar el período (mes/año)
3. Hacer clic en **Generar XLSX**
4. El reporte se descarga en el mismo formato del reporte actual con las hojas: Reportes, Ajustes, Informe, Estadística Mensual, Dashboards

---

## Preguntas frecuentes

**¿Necesito internet para usar la app?**
No. La app funciona completamente offline. Solo necesitás internet para subir reportes a Google Drive.

**¿Se pierden los datos si cierro el navegador?**
No. Los datos se guardan automáticamente en el navegador (IndexedDB). Sí se perderían si limpiás los datos del navegador.

**¿Puedo usar la app en varios equipos?**
Por ahora los datos son locales a cada equipo. La sincronización entre equipos se hace exportando/importando el backup de datos.

**¿Qué pasa si el material no aparece en las sugerencias?**
Podés registrarlo como ítem nuevo sin código. El sistema lo marcará como "sin código ERP" para que Almacén asigne el código correcto al procesar la compra.
