---
name: cobrar-por-whatsapp
description: Crea un cobro de Tilopay a partir de una frase natural ("cóbrale 25 mil colones a Ana por el curso") usando el MCP de Tilopay, y entrega el enlace de pago con un mensaje de WhatsApp listo para enviar (wa.me), el QR si lo piden, y la confirmación de pago cuando el comercio pregunte "¿ya me pagó?". Úsala cuando el comercio quiera cobrarle a alguien, mandar un link de pago, generar un QR de cobro o verificar si un cobro ya entró.
---

# Cobrar por WhatsApp con el MCP de Tilopay

Convierte una instrucción de cobro en un enlace de pago real de Tilopay y en el mensaje que el comercio reenvía a su cliente. El dinero lo cobra la página segura de Tilopay; esta skill nunca pide ni maneja datos de tarjeta.

## Herramientas que usa

| Herramienta | Para qué |
|---|---|
| `tilopay_create_charge` | Crea el cobro y devuelve `url` (enlace corto), `code` y `orderNumber`. |
| `tilopay_send_payment_link_whatsapp` | Arma el mensaje y el enlace wa.me para un teléfono o contacto guardado. |
| `tilopay_send_payment_link_email` | Lo mismo por correo, si el comercio prefiere. |
| `tilopay_payment_link_qr` | QR del enlace como imagen, para imprimir o compartir. |
| `tilopay_list_receipts` | Comprobantes de cobros pagados (trae el `orderNumber` del cobro y la página del comprobante). |
| `tilopay_find_payment` | Verificar por monto si un pago ya entró. |
| `tilopay_list_contacts` / `tilopay_save_contact` | Agenda de clientes del comercio. |
| `tilopay_get_business_profile` | Moneda por defecto y país del comercio (solo si hace falta). |

## Procedimiento

### 1. Extraer los datos de la frase

De "cóbrale 25 mil colones a Ana por el curso de fotografía" saca:

- `amount`: 25000 (interpreta "25 mil", "25k", "veinticinco mil"; los montos en colones no llevan decimales).
- `currency`: `CRC` si dice colones/₡; `USD` si dice dólares/$. Si no dice, usa la moneda del comercio (omite el parámetro y Tilopay aplica la del país) y **dilo en la respuesta** ("lo creé en colones; si era en dólares, lo rehago").
- `client`: "Ana".
- `description`: el concepto tal como lo dijo el comercio, con espacios, acentos y mayúsculas: `"Curso de fotografía"`. Nunca lo conviertas en slug (`curso_fotografia` está mal).
- `reference`: texto normal y corto que identifique el cobro, p. ej. `"Curso de fotografía Ana"`.

Si falta el monto, pregúntalo. Es lo único imprescindible; nombre y concepto pueden quedar genéricos ("Pago") si el comercio tiene prisa.

### 2. Crear el cobro

```
tilopay_create_charge
  amount, currency, client, description, reference
  client_phone / client_email (solo si el comercio los dio)
```

Respuesta real: `{"code":"xb2yxptz","url":"https://tilo.co/s/bS8PB97B","orderNumber":"HPP-…"}`. Guarda los tres; el `orderNumber` sirve luego para `tilopay_find_payment`.

### 3. Preparar el mensaje de WhatsApp

Si el comercio dio un teléfono o el nombre de un contacto guardado:

```
tilopay_send_payment_link_whatsapp
  link_url: <url del paso 2>
  phone: "+506XXXXXXXX"  (o contact_name)
  amount, currency, description
  save_contact: true solo si el comercio lo pidió
```

Devuelve un enlace `wa.me` con el texto listo: el comercio lo toca y WhatsApp se abre con el mensaje escrito.

**Si la herramienta rechaza el enlace** (acepta `tilopay.com`, `tilopay.shop/pagar/…` y `tilo.co/s/…`; un enlace antiguo `tp.cr/l/…` puede no pasar), construye el mensaje tú mismo; el resultado es idéntico para el comercio:

```
Texto:
Hola Ana, te comparto el enlace para el pago de Curso de fotografía por ₡25.000:
https://tilo.co/s/bS8PB97B
Podés pagar con tarjeta o SINPE Móvil de forma segura con Tilopay. ¡Gracias!

Enlace de un toque:
https://wa.me/506XXXXXXXX?text=<texto codificado con encodeURIComponent>
```

Si no hay teléfono, entrega solo el texto para copiar y pegar. Nunca inventes un número.

Formato de montos: `₡25.000` (punto de miles, sin decimales) y `$25.00` (dos decimales).

### 4. QR, solo si lo piden

`tilopay_payment_link_qr` con `link_url` y `label` (nombre del concepto). Útil para mostrador o impresión; por WhatsApp basta el enlace.

### 5. "¿Ya me pagó Ana?"

Primero los comprobantes, que sí traen el `orderNumber` del cobro (`HPP-…`):

```
tilopay_list_receipts  limit: 20
→ busca un recibo cuyo order_number == orderNumber del paso 2
```

Si aparece, el cobro está pagado: reporta monto, fecha (convertida a hora local) y el `page_url` del comprobante, que el comercio puede reenviar.

Si no aparece, confirma con `tilopay_find_payment` con `reference` igual al `orderNumber` `HPP-…` del cobro (resuelve el cobro propio aunque Tilopay registre la transacción con otro número) o, en su defecto, por `amount` + `currency` (y `hours` según lo que lleve abierto el cobro, máximo 720). `name` compara contra el nombre del tarjetahabiente y luego contra el correo.

Si nada aparece, dilo tal cual: "todavía no entra el pago de ₡25.000 de Ana" y ofrece reenviarle el mensaje.

## Respuesta al comercio

Corta y accionable:

```
Listo, creé el cobro de ₡25.000 a Ana por "Curso de fotografía".

Enlace de pago: https://tilo.co/s/bS8PB97B
Mensaje para WhatsApp (tocá para abrir): https://wa.me/506…?text=…

Cuando pague te aviso si me preguntás "¿ya pagó Ana?".
```

## Qué NO hacer

- No llames `tilopay_charge_saved_card` ni ninguna herramienta que mueva dinero: esta skill solo crea enlaces para que el cliente pague por su cuenta.
- No crees el cobro dos veces si la herramienta tardó: revisa primero con `tilopay_find_payment` o `tilopay_list_receipts`.
- No pidas al cliente ni al comercio números de tarjeta, CVV ni contraseñas, nunca.
- No cambies la moneda que el comercio dijo; si tienes dudas entre colones y dólares, pregunta antes de crear.
- No prometas el QR en el mismo mensaje: en Claude/ChatGPT el QR llega como imagen aparte si se pide.
