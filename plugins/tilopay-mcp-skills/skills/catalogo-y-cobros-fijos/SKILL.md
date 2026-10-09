---
name: catalogo-y-cobros-fijos
description: Gestiona el catálogo de productos y servicios de un comercio Tilopay con el MCP de Tilopay — lista y busca productos con su enlace de pago, crea o edita productos desde una frase o una lista, entrega el QR de un producto o el QR permanente de monto abierto para mostrador, y conecta el catálogo a Facebook/Instagram, Google Shopping, TikTok o Pinterest. Úsala cuando el comercio diga "agrega al catálogo", "cuánto cuesta X en mi catálogo", "dame el link de pago de tal producto", "QR para imprimir", "QR del mostrador" o "conectar mi catálogo a Instagram".
---

# Catálogo y cobros fijos con el MCP de Tilopay

Cada producto del catálogo de Tilopay tiene precio fijo, un enlace corto de pago permanente y un QR. Esta skill los crea, los busca y los reparte; el cobro lo hace la página segura de Tilopay. Para cobros de monto variable a una persona concreta, usa la skill `cobrar-por-whatsapp`.

## Herramientas que usa

| Herramienta | Para qué |
|---|---|
| `tilopay_catalog_list_items` | Listar o buscar (`search`) productos: `id`, `reference` (nombre), `amount`, `currency`, `short_link`, `has_qr`, `picture`. |
| `tilopay_catalog_get_item` | Detalle de un producto por `itemId`, con su QR. |
| `tilopay_catalog_create_item` | Crear producto (`name`, `amount`, `currency`, `description`); devuelve enlace corto y QR. |
| `tilopay_catalog_update_item` | Editar nombre, precio, moneda, descripción o foto sin cambiar el enlace. |
| `tilopay_catalog_delete_item` | Borrar (pide código de confirmación por correo; el enlace deja de funcionar). |
| `tilopay_payment_link_qr` | QR como imagen de un producto (`catalog_id`) o de un enlace (`link_url`). |
| `tilopay_static_payment_qr` | Enlace y QR permanente de monto abierto del comercio (el cliente digita el monto). Solo consulta. |
| `tilopay_catalog_feed` | URL del feed del catálogo y pasos para conectarlo a un canal (`platform`). |
| `tilopay_store_list` | Microtiendas del comercio (tilopay.shop) y su enlace. |
| `tilopay_send_payment_link_whatsapp` / `_email` | Compartir el enlace de un producto con un cliente. |

## Procedimiento

### Consultar

"¿Cuánto cuesta el vaso?" → `tilopay_catalog_list_items` con `search: "vaso"`. Responde con nombre, precio formateado (`₡2.500`, `$1.00`) y el `short_link`. Si hay varios parecidos, lístalos todos; si no hay ninguno, dilo y ofrece crearlo.

Nunca pidas `include_qr: true` en el listado (devuelve imágenes en base64 de todos los ítems); para un QR usa `tilopay_payment_link_qr` con `catalog_id`.

### Crear

De "agrega al catálogo Clase de yoga a 8 mil colones" extrae `name: "Clase de yoga"`, `amount: 8000`, `currency: "CRC"` (si no dice moneda, usa la del comercio y dilo). `description` corta solo si el comercio la dio. Antes de crear, busca por nombre para no duplicar; si ya existe uno igual, pregunta si quiere actualizar el precio o crear otro.

Para una lista de varios productos (texto u hoja): confirma la tabla interpretada (nombre, precio, moneda) **antes** de crear, crea uno por uno y entrega al final una tabla con nombre, precio y enlace de cada uno. Si uno falla, sigue con los demás y repórtalo.

Usa el `short_link` y el `id` que devuelve la creación: el listado del catálogo puede tardar unos minutos en reflejar el producto nuevo, así que no lo "verifiques" relistando ni lo crees dos veces si no aparece.

Si el comercio mandó una foto por WhatsApp en los últimos 10 minutos, la herramienta la usa como imagen del producto; desde Claude o ChatGPT no hay foto: pásale `use_last_image: false` para que no tome una imagen vieja.

### Editar y borrar

`tilopay_catalog_update_item` con el `itemId` del listado y solo los campos que cambian. El enlace se mantiene. Para borrar, explica que el enlace dejará de funcionar y que llegará un código al correo del comercio; llama `tilopay_catalog_delete_item` sin código, pide el código, y vuelve a llamar con `confirmation_code`. Nunca borres sin que el comercio lo haya pedido explícitamente.

### QR para imprimir

- De un producto: `tilopay_payment_link_qr` con `catalog_id` y `label` (nombre del producto). Entrega la URL de la imagen; vence a los 7 días, así que si es para imprimir pídele que la descargue ya.
- De mostrador (monto abierto): `tilopay_static_payment_qr`. Si `enabled` es `false`, explica que se activa desde el panel de Tilopay (no lo actives tú) y ofrece la guía con `tilopay_help_guides`.

### Conectar el catálogo a redes

Si el comercio no dijo el canal, pregunta cuál (Facebook/Instagram, Google Shopping, TikTok o Pinterest) antes de llamar `tilopay_catalog_feed`. Entrega la URL del feed y los pasos en el orden que devuelve la herramienta; con `send_email: true` también se lo manda al correo.

### Compartir un producto

`tilopay_send_payment_link_whatsapp` con el `short_link` del producto, `amount`, `currency`, `description` (nombre) y el teléfono o contacto. Si la herramienta rechaza el dominio del enlace (acepta `tilopay.com`, `tilopay.shop/pagar/…` y `tilo.co/s/…`; los enlaces viejos `tp.cr/l/…` pueden no pasar), arma el mensaje y el `wa.me` a mano con el `short_link`.

## Respuesta al comercio

Corta, con precio formateado y enlace completo. Ejemplo:

```
Listo: "Clase de yoga" quedó en tu catálogo a ₡8.000.
Enlace de pago: https://tilo.co/s/AbC12345
¿Te genero el QR para imprimir o te armo el mensaje de WhatsApp para un cliente?
```

## Qué NO hacer

- No crees productos sin monto; pregúntalo.
- No borres ni cambies precios sin pedido explícito; confirma antes de editar en lote.
- No uses `include_qr: true` en listados.
- No actives el QR permanente ni cambies la configuración del comercio: solo consulta.
- No inventes el feed ni los pasos de conexión: entrega lo que devuelve la herramienta.
