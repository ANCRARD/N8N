# DeliveryBot — Sistema de pedidos automatizado

Sistema de gestión de pedidos para domicilios, construido sobre **n8n** como capa de orquestación y **Google Sheets** como base de datos. El proyecto expone una API interna por webhooks, y sobre esa API se montan dos clientes: un bot de Telegram para el cliente final y un panel web para la cocina.

> **Estado del proyecto:** en desarrollo activo. La capa de API está parcialmente implementada; las interfaces de usuario están en fase de diseño. El detalle por componente está en la sección [Estado de los componentes](#estado-de-los-componentes).

---

## Arquitectura

```
┌──────────────────┐        ┌──────────────────┐
│  Bot de Telegram │        │  Panel Web       │
│  (cliente final) │        │  (cocina)        │
└────────┬─────────┘        └────────┬─────────┘
         │                           │
         │        HTTP / JSON        │
         └─────────────┬─────────────┘
                       ▼
         ┌─────────────────────────────┐
         │   Capa API — n8n Webhooks   │
         │  ┌───────────────────────┐  │
         │  │ Consultar Menú        │  │
         │  │ Crear Pedido          │  │
         │  │ Actualizar Estado     │  │
         │  └───────────────────────┘  │
         └─────────────┬───────────────┘
                       ▼
         ┌─────────────────────────────┐
         │  Google Sheets              │
         │  (DeliveryBot_DB)           │
         │  Service Account auth       │
         └─────────────────────────────┘
```

**Decisión de diseño:** toda la lógica de negocio vive en la capa de n8n, no en los clientes. Por lo tanto, tanto el bot como el panel web consumen exactamente los mismos endpoints, y agregar un tercer cliente (por ejemplo, una app móvil) no requeriría reescribir lógica.

---

## Base de datos — `DeliveryBot_DB`

Google Sheets, autenticado mediante **Google Service Account**.

El libro está compuesto por cuatro hojas, cada una con una responsabilidad distinta: catálogo, transacciones, clientes y estado conversacional.

### Hoja `MENU` — catálogo de productos

| Columna | Tipo | Descripción |
|---|---|---|
| `id_producto` | string | Identificador del producto (`P001`, `P002`, …) |
| `nombre` | string | Nombre comercial del producto |
| `categoria` | string | `Bebidas Calientes`, `Bebidas Frías`, `Alimentos`, `Postres` |
| `precio` | number | Precio unitario en COP |
| `disponible` | boolean | `TRUE` / `FALSE` — controla si el producto se muestra al cliente |
| `descripcion` | string | Descripción corta mostrada en el bot |
| `imagen_url` | string | URL de la imagen del producto (opcional) |

Actualmente cargados 10 productos (`P001`–`P010`) distribuidos en las cuatro categorías. Además, el campo `disponible` permite retirar un producto de la carta sin borrar el registro, lo cual preserva la integridad histórica de los pedidos que ya lo referencian.

### Hoja `PEDIDOS` — transacciones

| Columna | Tipo | Descripción |
|---|---|---|
| `id_pedido` | string | Identificador único generado al crear el pedido (`PED-XXXXXXXX-NNN`) |
| `chat_id` | string | ID del cliente en Telegram, usado para notificarle cambios de estado |
| `fecha_hora` | ISO 8601 | Timestamp de creación |
| `items_json` | string (JSON) | Detalle de los productos serializado en una sola celda |
| `total` | number | Suma de `cantidad × precio_unitario` de todos los items |
| `estado` | string | `pendiente` → `en_preparacion` → `en_camino` → `entregado` |
| `notas` | string | Dirección, método de pago, nombre y teléfono del cliente |

### Hoja `USUARIOS` — registro de clientes

| Columna | Tipo | Descripción |
|---|---|---|
| `chat_id` | string | ID de Telegram — clave con la que se relaciona con `PEDIDOS` |
| `nombre` | string | Nombre del cliente |
| `telefono` | string | Teléfono de contacto |
| `fecha_registro` | ISO 8601 | Primer contacto con el bot |
| `direccion` | string | Dirección de entrega guardada, para no volver a pedirla |

### Hoja `SESSIONS` — estado conversacional del bot

| Columna | Tipo | Descripción |
|---|---|---|
| `chat_id` | string | ID de Telegram del usuario en conversación |
| `estado_conversacion` | string | Paso actual del flujo (menú, carrito, confirmación, etc.) |
| `carrito_json` | string (JSON) | Carrito en construcción antes de convertirse en pedido |
| `ultima_actualizacion` | ISO 8601 | Timestamp del último mensaje procesado |

Esta hoja resuelve un problema propio de n8n: los workflows son **sin estado** entre ejecuciones, y cada mensaje de Telegram dispara una ejecución independiente. Por lo tanto, el carrito y el paso de la conversación deben persistirse externamente. `SESSIONS` cumple esa función de memoria de sesión.

### Relación entre hojas

```
USUARIOS.chat_id ──┬── PEDIDOS.chat_id
                   └── SESSIONS.chat_id

MENU.id_producto ───── PEDIDOS.items_json[].id_producto
```

---

## Estado de los componentes

| # | Componente | Tipo | Estado |
|---|---|---|---|
| 1 | API — Consultar Menú | Workflow n8n | ✅ Implementado y probado |
| 2 | API — Actualizar Estado | Workflow n8n | ✅ Implementado y probado |
| 3 | API — Crear Pedido | Workflow n8n | 🔧 En construcción — lógica definida, nodos en montaje |
| 4 | Panel Web de cocina | Front-end | ⏳ Pendiente — especificado, sin implementar |
| 5 | Bot de Telegram | Workflow n8n | ⏳ Pendiente — especificado, sin implementar |

---

## Endpoints

### `GET` / Consultar Menú — ✅

Devuelve el catálogo de productos disponibles desde la hoja de menú.

> ⚠️ **Completar:** path del webhook, parámetros y ejemplo de respuesta.

---

### `POST` / Actualizar Estado — ✅

Path: `pedido/estado`

Actualiza el campo `estado` de un pedido existente y responde con una confirmación en JSON.

> ⚠️ **Completar:** body de ejemplo y respuesta exacta.

---

### `POST` / Crear Pedido — 🔧

Path: `pedido/crear`

Recibe un pedido completo, calcula el total, genera el identificador y persiste la fila en la hoja `PEDIDOS`.

**Cadena de nodos:** `Webhook` → `Code (JavaScript)` → `Append Row in Sheet` → `Respond to Webhook`

**Lógica de transformación (nodo Code):**

```javascript
const body = $input.first().json.body;

// ID único: PED + últimos 8 dígitos del timestamp + random de 3 dígitos
const idPedido = 'PED-' + Date.now().toString().slice(-8) + '-' + Math.floor(Math.random() * 1000);

const fechaHora = new Date().toISOString();

// Total = suma de cantidad * precio_unitario
let total = 0;
for (const item of body.items) {
  total += item.cantidad * item.precio_unitario;
}

// Los items se serializan para caber en una sola celda
const itemsJson = JSON.stringify(body.items);

const notas = `Dirección: ${body.direccion_entrega} | Pago: ${body.metodo_pago} | Cliente: ${body.nombre_cliente} | Tel: ${body.telefono}`;

return {
  json: {
    id_pedido: idPedido,
    chat_id: body.id_cliente,
    fecha_hora: fechaHora,
    items_json: itemsJson,
    total: total,
    estado: 'pendiente',
    notas: notas
  }
};
```

**Request de ejemplo:**

```json
{
  "id_cliente": "123456789",
  "nombre_cliente": "Andrés Anaya",
  "telefono": "3001234567",
  "direccion_entrega": "Calle 10 #5-20",
  "metodo_pago": "efectivo",
  "items": [
    { "id_producto": "P001", "nombre": "Café Americano", "cantidad": 2, "precio_unitario": 18500 }
  ]
}
```

El nodo de Sheets usa **Map Automatically**, dado que los nombres de las claves del objeto retornado coinciden uno a uno con los encabezados de la hoja `PEDIDOS`.

---

## Pruebas

Los endpoints se prueban con **Insomnia**, apuntando a la URL de producción del webhook de cada workflow publicado en n8n. Cada versión se publica con una etiqueta descriptiva (por ejemplo, `v1 - crear pedido`).

---

## Próximos pasos

1. **Cerrar `Crear Pedido`** — terminar el montaje de los 4 nodos y validar contra la hoja `PEDIDOS` con el request de ejemplo.
2. **Bot de Telegram** — workflow con trigger de Telegram que consuma `Consultar Menú` para mostrar el catálogo, arme el carrito en memoria de sesión y dispare `Crear Pedido` al confirmar.
3. **Panel Web de cocina** — vista que liste los pedidos en estado `pendiente` y `en_preparacion`, con botones que llamen a `Actualizar Estado`.
4. **Notificaciones al cliente** — al cambiar el estado, usar el `chat_id` almacenado para avisar por Telegram.
5. **Validaciones** — verificar campos obligatorios y existencia de productos antes de escribir en la hoja.

---

## Stack

- **n8n** — orquestación y capa de API
- **Google Sheets API** — persistencia (Service Account)
- **Telegram Bot API** — interfaz del cliente final
- **Insomnia** — pruebas de endpoints
