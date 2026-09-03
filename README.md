# DeliveryBot

**Sistema de gestión de pedidos para cafetería, construido sobre automatización de bajo código.**

Un bot de Telegram para el cliente, un panel web para la cocina, y una API propia que los conecta con una base de datos en Google Sheets.

---

## Índice

- [¿Por qué este proyecto?](#por-qué-este-proyecto)
- [¿Para qué sirve?](#para-qué-sirve)
- [Arquitectura](#arquitectura)
- [Base de datos](#base-de-datos)
- [La API](#la-api)
- [El bot de Telegram](#el-bot-de-telegram)
- [El panel de cocina](#el-panel-de-cocina)
- [Decisiones técnicas](#decisiones-técnicas)
- [Instalación](#instalación)
- [Estructura del repositorio](#estructura-del-repositorio)
- [Deuda técnica conocida](#deuda-técnica-conocida)

---

## ¿Por qué este proyecto?

Las cafeterías y restaurantes pequeños en Colombia gestionan sus domicilios por WhatsApp. El proceso típico es este: el cliente escribe, alguien le manda una foto del menú, el cliente responde qué quiere, y esa persona anota el pedido en un papel o lo grita hacia la cocina. Después hay que recordar la dirección, el método de pago y en qué punto va cada pedido.

Ese flujo tiene tres problemas que se agravan cuando aumenta el volumen.

**Primero, el cuello de botella humano.** Una persona atiende una conversación a la vez. En hora pico, los mensajes se acumulan y el tiempo de respuesta se dispara, así que se pierden ventas por demora.

**Segundo, los errores de transcripción.** Un pedido dictado y anotado a mano se equivoca en la cantidad o en el precio. Además, el precio se calcula mentalmente, lo cual genera cobros inconsistentes.

**Tercero, la ausencia de trazabilidad.** Nadie sabe con certeza cuánto lleva esperando un pedido, ni cuántos se vendieron ayer, ni cuál es el producto más pedido. La información se pierde apenas se entrega el domicilio, y el cliente tampoco sabe en qué va su pedido salvo que pregunte.

Ahora bien, la solución obvia —comprar un software de domicilios— no es viable para un negocio de este tamaño. Las plataformas comerciales cobran comisiones por pedido, exigen contratos, e imponen su propio flujo de trabajo.

Este proyecto explora una tercera vía: **construir el sistema con herramientas de automatización de bajo código y servicios gratuitos**, de modo que el costo de operación sea prácticamente cero y el negocio conserve el control de sus datos.

---

## ¿Para qué sirve?

El sistema resuelve cinco cosas concretas.

**Atiende a varios clientes al mismo tiempo.** El bot no se satura: puede llevar veinte conversaciones simultáneas sin que ninguna espere. Por lo tanto, el personal deja de estar amarrado al teléfono y se dedica a preparar.

**Elimina el error de cálculo.** El cliente nunca escribe un precio ni una cantidad en texto libre: solo toca botones. El total lo calcula el sistema leyendo el catálogo, así que no hay discrepancias entre lo que se cobra y lo que vale.

**Le da visibilidad a la cocina.** El panel muestra cada pedido con un cronómetro de cuánto lleva esperando, y se pone en rojo pasados quince minutos. Un toque avanza el estado; otro lo cancela.

**Mantiene informado al cliente sin que nadie tenga que escribirle.** Cada vez que la cocina cambia el estado de un pedido —en preparación, en camino, entregado o cancelado—, el sistema le manda automáticamente un mensaje de Telegram al cliente. Nadie en el negocio tiene que acordarse de avisar.

**Deja los datos guardados.** Cada pedido queda en una hoja de cálculo con su fecha, sus productos, su total y su estado. En consecuencia, el negocio puede después responder preguntas que antes no podía: qué se vende más, a qué hora hay pico, cuánto se facturó.

### Objetivos de aprendizaje

Más allá del caso de uso, el proyecto sirvió para practicar:

- Diseño de una **API REST** y su contrato (rutas, métodos, cuerpos, respuestas)
- **Modelado de datos relacionales** sobre un almacenamiento no relacional
- **Máquinas de estado** aplicadas a una conversación (el problema de mantener contexto entre eventos independientes) y a un pedido (su ciclo de vida completo)
- **Separación cliente/servidor**: dos interfaces distintas consumiendo la misma capa de lógica
- **Frontend sin framework**: HTML, CSS y JavaScript puro, con la estructura separada en tres archivos
- **Notificaciones activadas por evento**, en vez de que el cliente tenga que preguntar por su estado
- Consideraciones básicas de **seguridad**: validación del lado del servidor y manejo de credenciales

---

## Arquitectura

```
   CLIENTE                                    COCINA
      │                                          │
┌─────▼──────────┐                    ┌──────────▼─────────┐
│ Bot de Telegram│                    │  Panel Web         │
│  (botones)     │◄───notificación────│  HTML + CSS + JS   │
└─────┬──────────┘                    └──────────┬─────────┘
      │                                          │
      │              HTTP / JSON                 │
      └────────────────┬─────────────────────────┘
                       │
        ┌──────────────▼───────────────┐
        │      n8n  ·  capa de API     │
        │                              │
        │  GET  /menu                  │
        │  GET  /pedidos               │
        │  POST /pedido/crear          │
        │  POST /pedido/estado ──┐     │
        └──────────────┬─────────┼─────┘
                       │        └──► Telegram (avisa al cliente)
        ┌──────────────▼───────────────┐
        │   Google Sheets              │
        │   MENU · PEDIDOS             │
        │   USUARIOS · SESSIONS        │
        └──────────────────────────────┘
```

**El principio rector es que toda la lógica de negocio vive en la capa de n8n, nunca en los clientes.** El bot y el panel son interfaces tontas: presentan información y disparan acciones, pero no calculan totales ni deciden reglas. Por lo tanto, agregar un tercer cliente —una app móvil, un tótem en el local— no requeriría reescribir nada de la lógica.

Los cuatro endpoints pueden vivir como **cuatro workflows independientes** (uno por responsabilidad, recomendado para mantenimiento y para la lectura del proyecto) o **fusionados en un solo workflow** con cuatro webhooks internos (más rápido de desplegar). Ambas versiones están en el repositorio; ver [Estructura del repositorio](#estructura-del-repositorio).

---

## Base de datos

Google Sheets funciona como base de datos, autenticada mediante **Google Service Account**. Cada hoja cumple el papel de una tabla.

### `MENU` — catálogo

| Columna | Tipo | Descripción |
|---|---|---|
| `id_producto` | string | Clave primaria (`P001`…`P010`) |
| `nombre` | string | Nombre comercial |
| `categoria` | string | `Bebidas Calientes`, `Bebidas Frías`, `Alimentos`, `Postres` |
| `precio` | number | Precio unitario en COP |
| `disponible` | boolean | Controla la visibilidad ante el cliente |
| `descripcion` | string | Texto mostrado en el bot |
| `imagen_url` | string | Opcional |

El campo `disponible` permite retirar un producto de la carta sin borrar el registro, para no dejar huérfanos los pedidos históricos que lo referencian.

### `PEDIDOS` — transacciones

| Columna | Tipo | Descripción |
|---|---|---|
| `id_pedido` | string | `PED-XXXXXXXX-NNN`, generado al crear |
| `chat_id` | string | Relación con `USUARIOS`, usado para notificar |
| `fecha_hora` | ISO 8601 | Momento de creación |
| `items_json` | string | Productos serializados en una celda |
| `total` | number | Calculado en el servidor |
| `estado` | string | `pendiente` → `en_preparacion` → `en_camino` → `entregado`, o `cancelado` en cualquier punto |
| `notas` | string | Dirección, pago, nombre y teléfono |

Los productos se guardan serializados como JSON en una sola celda. Es una desnormalización consciente: una hoja de cálculo no admite relaciones uno-a-muchos, así que la alternativa sería una segunda hoja de detalle con su propia gestión de claves.

### `USUARIOS` — clientes

| Columna | Descripción |
|---|---|
| `chat_id` | Identificador de Telegram, clave primaria |
| `nombre` | Nombre del cliente |
| `telefono` | Teléfono |
| `fecha_registro` | Primer pedido |
| `direccion` | Última dirección usada |

### `SESSIONS` — estado conversacional

| Columna | Descripción |
|---|---|
| `chat_id` | Usuario en conversación |
| `estado_conversacion` | Paso actual del flujo, con datos empaquetados |
| `carrito_json` | Carrito en construcción |
| `ultima_actualizacion` | Marca de tiempo |

Esta hoja resuelve el problema central del proyecto. Los workflows de n8n son **sin estado**: cada mensaje de Telegram dispara una ejecución nueva que no recuerda nada de la anterior. En consecuencia, el carrito y el paso de la conversación deben persistirse externamente. `SESSIONS` es esa memoria.

El campo `estado_conversacion` empaqueta más de un dato cuando hace falta, con el formato `paso|metodo_pago|direccion` — una técnica útil cuando agregar columnas nuevas a la hoja no es práctico a mitad de desarrollo.

### Relaciones

```
USUARIOS.chat_id ──┬── PEDIDOS.chat_id
                   └── SESSIONS.chat_id

MENU.id_producto ───── PEDIDOS.items_json[].id_producto
```

---

## La API

Cuatro endpoints, todos con CORS habilitado.

### `GET /webhook/menu`

Devuelve el catálogo, **filtrando los productos agotados**.

### `GET /webhook/pedidos`

Devuelve todos los pedidos con el estado normalizado (sin tildes, en minúscula, con guiones bajos), para que datos históricos escritos de forma inconsistente no rompan al cliente que los consume.

### `POST /webhook/pedido/crear`

```json
{
  "id_cliente": "123456789",
  "nombre_cliente": "Andrés Anaya",
  "telefono": "3001234567",
  "direccion_entrega": "Calle 10 #5-20",
  "metodo_pago": "efectivo",
  "items": [{ "id_producto": "P001", "cantidad": 2 }]
}
```

**El cuerpo no lleva precios.** El servidor los busca en `MENU` por `id_producto` y calcula el total. Valida que cada producto exista y esté disponible.

```json
{ "success": true, "id_pedido": "PED-23886863-512", "total": 9000, "estado": "pendiente" }
```

### `POST /webhook/pedido/estado`

```json
{ "id_pedido": "PED-23886863-512", "nuevo_estado": "en_preparacion" }
```

Valida que el estado sea uno de los cinco permitidos (`pendiente`, `en_preparacion`, `en_camino`, `entregado`, `cancelado`) y que el pedido exista; responde **400** con el motivo si algo falla, en vez de romperse.

Después de actualizar la hoja, **este endpoint le manda al cliente una notificación por Telegram** con un mensaje distinto según el nuevo estado. Si Telegram no responde por cualquier motivo, el endpoint sigue respondiéndole bien al panel — la notificación es un valor agregado, nunca debe tumbar la actualización real del pedido.

---

## El bot de Telegram

### Recorrido del cliente

```
/start
  └→ [Bebidas Calientes] [Bebidas Frías] [Alimentos] [Postres]
       └→ [Café Americano · $4.500] [Cappuccino · $6.000] …
            └→ ¿Cuántas?  [1] [2] [3] [4] [5]
                 └→ ✅ Agregado · Total actualizado
                      ├→ [➕ Seguir pidiendo]
                      └→ [✅ Confirmar pedido]
                           └→ [💵 Efectivo] [🏦 Transferencia]
                                └→ 🎉 Pedido confirmado
```

El cliente navega enteramente con botones, sin escribir texto libre en ningún paso, lo que elimina errores de digitación y la necesidad de validar entrada libre. La dirección y el teléfono, en la versión de demostración, quedan con un valor de referencia fijo; en una versión de producción se recuperarían pidiéndoselos al cliente o reutilizando lo guardado en `USUARIOS`.

Una vez confirmado el pedido, el cliente no necesita preguntar por su estado: **cada avance de la cocina en el panel le llega como un mensaje nuevo al chat**, sin que nadie del negocio tenga que escribirle.

### Estructura interna

Todos los eventos —mensajes y toques de botón— entran por un único `Telegram Trigger`. Un nodo `Clasificar` los etiqueta, y un `Switch` los reparte por ramas.

```
Telegram Trigger → Clasificar → Switch ──┬→ /start
                                         ├→ categoría
                                         ├→ producto
                                         ├→ cantidad
                                         ├→ carrito
                                         ├→ confirmar
                                         └→ pago → crear pedido
```

Los botones llevan escondido un dato con el formato `TIPO|VALOR` —por ejemplo `CANT|P001:2`—, que es lo que el `Switch` lee para decidir la rama.

### Detalle de implementación: los botones dinámicos

El nodo nativo de Telegram en n8n solo permite armar teclados con un constructor visual, botón por botón. Como el menú se genera desde la hoja y la cantidad de botones varía, ese nodo no sirve.

La solución fue reemplazar los nodos de envío por **HTTP Request** llamando directamente a la API de Telegram (`sendMessage` y `editMessageText`), lo cual permite pasar el teclado como JSON generado en tiempo de ejecución.

Además, el bot **edita el mensaje anterior** en lugar de enviar uno nuevo en cada paso. Por lo tanto, el chat no se llena de mensajes repetidos y la experiencia se siente como una aplicación.

---

## El panel de cocina

Tres archivos sin framework ni dependencias de compilación: `index.html`, `styles.css` y `app.js`.

### Criterio de diseño

El contexto de uso determinó todas las decisiones visuales: una tablet fija en cocina, vista a un metro de distancia, en un ambiente con ruido y prisa, por un operario con las manos ocupadas que mira la pantalla dos segundos.

De ahí que:

- **El cronómetro sea el elemento dominante** de cada comanda, y se ponga en rojo a los quince minutos.
- **La tipografía sea condensada** para los nombres de producto y monoespaciada para identificadores y tiempos.
- **Un toque avance el estado**, sin diálogos de confirmación; **cancelar sí pide confirmación**, porque es una acción destructiva que además le llega al cliente.

### Funcionamiento

Tres rieles verticales, uno por estado activo (`pendiente`, `en_preparacion`, `en_camino`). Cada comanda tiene dos botones: uno principal que avanza al siguiente estado, y uno secundario para cancelar el pedido, que dispara una confirmación antes de ejecutarse.

El pie del panel muestra cuántos pedidos se han entregado y cuántos se han cancelado en la sesión.

El panel consulta `/pedidos` cada quince segundos. Google Sheets y los webhooks no permiten *push*, así que el "tiempo real" es en realidad *polling* — una limitación consciente de la arquitectura, suficiente para el volumen de una cafetería.

---

## Decisiones técnicas

**El precio se calcula siempre en el servidor.** El cliente nunca envía un precio; el servidor lo busca en `MENU`, lo cual cierra la puerta a que alguien con la URL del webhook cree un pedido con el precio que quiera.

**El bot consume la propia API para crear pedidos**, en vez de escribir en `PEDIDOS` por su cuenta. Así, la generación del identificador, el cálculo del total y las validaciones viven en un solo lugar. La llamada usa `localhost`, porque n8n se invoca a sí mismo, lo cual la hace inmune a cambios en la URL pública de ngrok.

**El estado se normaliza al leer, no al escribir.** Datos históricos con formato inconsistente no rompen a los clientes que los consumen.

**Cancelar es un estado más del pedido, no un borrado.** El registro permanece en `PEDIDOS` con `estado: cancelado`, preservando el historial para análisis posterior, en vez de eliminar la fila.

**Las notificaciones no pueden tumbar la operación principal.** El nodo que le avisa al cliente por Telegram está configurado para continuar aunque falle, de modo que un problema de red con Telegram nunca impida que el estado del pedido se actualice correctamente en la hoja.

---

## Instalación

### Requisitos

- Docker y Docker Compose
- Una cuenta de Google con una hoja de cálculo y un Service Account
- Un bot de Telegram creado con [@BotFather](https://t.me/botfather)
- ngrok o equivalente, para exponer n8n a internet

### Pasos

**1. Levantar n8n**

```bash
docker compose up -d
```

**2. Exponer el servicio**

```bash
ngrok http 5678
```

Configurar la dirección resultante en la variable `WEBHOOK_URL` del `compose.yml`. Telegram necesita alcanzar n8n desde internet; el panel y las llamadas internas usan `localhost` y no dependen del túnel.

**3. Crear las credenciales en n8n**

- *Google Service Account API* — con el correo y la clave privada del Service Account
- *Telegram API* — con el token de BotFather

La hoja de cálculo debe estar compartida con el correo del Service Account.

**4. Importar los workflows**

Los archivos de `workflows/` — ya sea la versión de cuatro endpoints separados o la fusionada en uno solo, nunca ambas a la vez, porque comparten rutas y n8n no permite dos workflows escuchando la misma dirección. En cada uno hay que seleccionar las credenciales, y en los nodos HTTP Request del bot y de notificaciones reemplazar `TU_TOKEN_AQUI` por el token real. Publicar todos.

**5. Abrir el panel**

Abrir `panel-web/index.html` en un navegador. Si n8n no corre en el puerto por defecto, ajustar `BASE_URL` en `app.js`.

---

## Estructura del repositorio

```
deliverybot/
├── README.md
├── workflows/
│   ├── API - Consultar Menu.json
│   ├── API - Listar Pedidos.json
│   ├── API - Crear Pedido.json
│   ├── API - Actualizar Estado Pedido.json
│   ├── API - DeliveryBot (todos los endpoints).json   ← alternativa fusionada
│   └── BOT - Telegram.json
└── panel-web/
    ├── index.html
    ├── styles.css
    └── app.js
```

> **Nota sobre credenciales.** Los workflows exportados no contienen la clave del Service Account, únicamente una referencia a la credencial almacenada en n8n. Sin embargo, el bot y el endpoint de notificaciones sí llevan el token de Telegram escrito en la URL de sus nodos HTTP Request, así que **debe reemplazarse por un marcador antes de publicar el repositorio**.

---

## Deuda técnica conocida

Lo que un siguiente ciclo debería atender:

- **Los webhooks no tienen autenticación.** Cualquiera con la URL puede crear pedidos o cambiar estados. Correspondería una cabecera con token compartido.
- **La dirección y el teléfono son de referencia en la demo.** En producción se pedirían al cliente o se reutilizarían desde `USUARIOS`.
- **El cliente no puede cancelar su propio pedido** desde el bot; la cancelación solo está disponible desde el panel de cocina.
- **Cada consulta lee la hoja completa.** Con cientos de pedidos, la latencia crecería de forma lineal.
- **El carrito no se puede editar** una vez agregado un producto: no hay forma de quitar ni corregir cantidades.
- **No hay historial de pedidos accesible para el cliente** dentro del bot.

---

## Stack

| Componente | Tecnología |
|---|---|
| Orquestación y API | n8n (Docker) |
| Base de datos | Google Sheets API (Service Account) |
| Interfaz del cliente | Telegram Bot API |
| Interfaz de cocina | HTML, CSS y JavaScript sin framework |
| Exposición pública | ngrok |
| Pruebas de endpoints | Insomnia |
