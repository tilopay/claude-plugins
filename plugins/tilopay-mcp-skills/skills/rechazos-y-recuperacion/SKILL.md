---
name: rechazos-y-recuperacion
description: Analiza las transacciones rechazadas de un comercio Tilopay con el MCP de Tilopay, las agrupa por motivo real (código del procesador), detecta patrones (mismo cliente reintentando, misma orden, método de pago, horario) y entrega acciones concretas para recuperar cada venta, incluido el mensaje al cliente y un nuevo enlace de pago si hace falta. Úsala cuando el comercio pregunte "por qué me rechazan pagos", "cuánto dejé de cobrar", "qué le digo al cliente", o pida revisar rechazos, declinadas o ventas perdidas.
---

# Rechazos y recuperación con el MCP de Tilopay

Convierte una lista de rechazos en dinero recuperable: cuánto se perdió, por qué, y qué hacer con cada caso. Solo afirma lo que las filas sustentan.

## Herramientas que usa

| Herramienta | Para qué |
|---|---|
| `tilopay_list_transactions` con `status: "declined"` | Las filas rechazadas del período, con código, texto, cliente, método y canal. |
| `tilopay_list_transactions` con `status: "approved"` | Para saber si un rechazo se recuperó después (mismo cliente o misma orden aprobada más tarde). |
| `tilopay_sales_summary` | `declineReasons` agregados y tasa de aprobación del período. |
| `tilopay_create_charge` + `tilopay_send_payment_link_whatsapp` | Nuevo enlace y mensaje para recuperar una venta concreta (solo si el comercio lo pide). |
| `tilopay_help_guides` | Guías oficiales cuando la acción es "configurar algo en el panel". |

## Procedimiento

### 1. Período y datos

Por defecto los últimos 30 días en hora local del comercio (convierte a UTC: 00:00 Costa Rica = 06:00 UTC). Dos llamadas:

```
tilopay_list_transactions  startDate, endDate, status: "declined", sort: "newest", limit: 500
tilopay_list_transactions  startDate, endDate, status: "approved", sort: "newest", limit: 500
```

Si `total` iguala al `limit`, parte el rango por semanas y repite. Excluye `test == 1` salvo que el comercio quiera ver pruebas.

### 2. Separar lo que no es un rechazo de venta

De las filas con `status: "declined"`:

- `orderNumber` que empieza por `R-` o `Re-` → es un **reembolso o reversión fallida**, no una venta perdida. Repórtalo aparte ("1 reembolso no se pudo procesar, código 12").
- `code == "99"` ("Inicio verificación 3DS") o `code == ""` con `responseText` vacío → el cliente **abandonó** antes de que el banco respondiera. Es carrito abandonado, no rechazo del banco.
- `capture == 0` con `code == "1"` no aparece aquí, pero si lo ves en aprobadas es una autorización sin capturar: menciónalo si el comercio pregunta por dinero "que no llega".

Lo que queda son rechazos reales del emisor o del procesador.

### 3. Agrupar por motivo y poner la etiqueta en lenguaje simple

Agrupa por `code` (no por `responseText`, que varía entre adquirentes). Tabla de referencia en `references/codigos-rechazo.md`. Resumen:

Usa las etiquetas de la guía oficial "Motivos de rechazo" (https://tilopay.com/guias/panel-transacciones/motivos-de-rechazo), que es lo que el comercio ve en su panel:

| Código | Etiqueta para el comercio | Recuperable | Acción |
|---|---|---|---|
| `51`, `61`, `65` | Fondos o límite de la tarjeta | Sí, alta | Reintentar en 1–3 días u ofrecer otra tarjeta / SINPE Móvil |
| `TO` | Datos de tarjeta incorrectos o tiempo agotado | Sí, alta | Enviar el enlace de nuevo y pedir que revise número, vencimiento y CVV |
| `14` · `N7`, `82`, `63` · `54` · `25` | Número incorrecto · CVV incorrecto · Tarjeta vencida · No se ubica la tarjeta | Sí, alta | Reenviar enlace; pedir que revise los datos o use otra tarjeta |
| `58`, `57`, `62` | Cobro no permitido en esa tarjeta | Sí, media | Habilitar compras en línea/internacionales con su banco o usar otra tarjeta |
| `3`, `99`, `05` (y `B05`, `DR`, `AP3`, `NS`, `E005`) | El banco no aprobó el cobro | Media | Reintentar en minutos, otra tarjeta o llamar al banco |
| `12`, `13` | No se pudo procesar el pago / Monto inválido | Depende | Rehacer el cobro; si siempre falla, soporte de Tilopay |
| `95`, `96`, `91` | Error del procesador o del emisor | Depende | Reintentar; si se repite, abrir caso con soporte |
| `97` | Error de integración 3DS | No por el cliente | Es del desarrollador del sitio: revisar los campos 3DS que envía (`browserLanguage`, etc.) |
| `41`, `43`, `59` | Tarjeta bloqueada por el banco | No | No insistir; no reenviar enlace |

Si un código no está en la tabla ni en la guía, muestra el `responseText` entre comillas, trátalo como "rechazo general del banco" (así lo indica la guía) y sugiere otra tarjeta; no inventes una explicación. Cuando el comercio quiera leer más, enlaza la guía.

### 4. Detectar patrones (solo los que los datos muestren)

Para cada grupo y en total, revisa:

- **Mismo cliente reintentando**: igual `email` (o `cardholder` + `lastname`) con 2+ rechazos seguidos. Es el candidato número uno a recuperar con un mensaje personal.
- **Misma orden**: igual `orderNumber` rechazada y luego aprobada → ya se recuperó; no la cuentes como perdida.
- **Método o canal**: concentración en `paymentLabel.title_es` o `platform` (`sdk`, `TilopayLink`, `TilopayRepeat`…). Si todos los `97` vienen de `platform: "sdk"`, el problema es la integración, no los clientes.
- **Marca o adquirente**: `brand`, `acquirer`.
- **Hora local**: rechazos concentrados de madrugada suelen ser intentos no legítimos.

Calcula el **monto no cobrado** por moneda = Σ `amount` de rechazos reales no recuperados. Nunca mezcles CRC y USD.

### 5. Entregar el análisis

```
Rechazos del 7 de septiembre al 7 de octubre (hora Costa Rica)

Aprobadas 21 · Rechazadas 4 · Tasa de aprobación 84 %
Monto no cobrado: $0 — los 3 rechazos de venta se recuperaron después

Por motivo
• Datos de tarjeta incorrectos o tiempo agotado (TO): 3 intentos por $1,22, todos de la misma orden
    el 6 de octubre entre 5:29 y 5:31 am (2 de Carla B., 1 de otro correo).
    → Ya recuperado: la misma orden se aprobó a las 5:32 am. Nada que hacer.
• Error de conciliación (95): 1 — era un reembolso, no una venta. Si vuelve a pasar, caso con soporte.

Siguiente paso sugerido
Ninguno urgente. Si querés, activo el aviso de pago por WhatsApp para enterarte al instante de cada rechazo.
```

Si la muestra es pequeña (menos de 30 filas o menos de 5 días con ventas), dilo y no hables de "tendencias".

### 6. Recuperar una venta (solo a pedido del comercio)

1. `tilopay_create_charge` con el mismo monto, moneda y concepto de la venta rechazada (usa la skill `cobrar-por-whatsapp` si está disponible).
2. `tilopay_send_payment_link_whatsapp` con el teléfono o contacto; si la herramienta rechaza el dominio del enlace, arma el `wa.me` a mano.
3. Mensaje sugerido según motivo, por ejemplo para `51`: "Hola Carla, el pago de $1.22 no se completó por un tema con la tarjeta. Te dejo el enlace para intentarlo de nuevo cuando gustes, también podés pagar con SINPE Móvil: <enlace>".

Nunca pidas al cliente datos de tarjeta por WhatsApp ni le digas el motivo exacto que dio su banco si suena acusatorio ("fondos insuficientes"): usa "no se completó por un tema con la tarjeta".

## Qué NO hacer

- No decidas aprobado/rechazado por el texto: solo por `code` (`"1"` aprobada, `"1101"` reembolso aprobado, resto rechazo o incompleto).
- No cuentes reembolsos fallidos ni abandonos 3DS como "rechazos del banco".
- No propongas cambios de adquirente, antifraude o configuración que las filas no justifiquen.
- No llames `tilopay_analyze_sales` como fuente de cifras; si lo usas, es solo para redactar y debes verificar cada número contra las filas.
