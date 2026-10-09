# Códigos de respuesta en transacciones de Tilopay

Campo `code` de `tilopay_list_transactions`. Fuente principal: la guía oficial **"Motivos de rechazo"** (https://tilopay.com/guias/panel-transacciones/motivos-de-rechazo), que cubre cerca del 90 % de los rechazos; los códigos marcados ✔ se observaron además en producción al construir esta skill (octubre 2026). Cuando la guía y esta tabla difieran, manda la guía.

## Aprobaciones

| Código | Significado |
|---|---|
| `1` ✔ | Venta aprobada |
| `1101` ✔ | Reembolso o reversión aprobada (orden `R-…` / `Re-…`) |

## Incompletas (no son rechazo del banco)

| Código | Texto típico | Tratamiento |
|---|---|---|
| `""` (vacío) ✔ | Sin respuesta del procesador | Abandono o error de red |
| `99` con texto "Inicio verificación 3DS" ✔ | El cliente no terminó la autenticación 3DS | Abandono (la guía oficial lista 99 como "el banco no aprobó"; si el texto dice 3DS, trátalo como abandono) |

## Rechazos recuperables con el cliente (etiquetas de la guía oficial)

| Etiqueta oficial | Códigos | Qué pasó | Qué hacer |
|---|---|---|---|
| El banco no aprobó el cobro | `3`, `99`, `05` (también `B05`, `DR`, `AP3`, `NS`, `E005`) | El emisor rechazó sin motivo específico (temporal o de seguridad) | Reintentar en unos minutos, otra tarjeta o que el cliente llame a su banco |
| Fondos o límite de la tarjeta | `51` ✔, `61`, `65` | Sin saldo o supera el límite / número de intentos | Otra tarjeta, SINPE Móvil, o reintentar en 1–3 días |
| No se pudo procesar el pago | `12` ✔ | Error al procesar o dato inválido; frecuente en reintentos de reembolso y cobros recurrentes repetidos | Rehacer el cobro; si siempre falla, soporte de Tilopay |
| Monto inválido | `13` | El monto no es válido para el emisor | Revisar monto y moneda |
| Cobro no permitido en esa tarjeta | `58` ✔, `57`, `62` | La tarjeta no tiene habilitado ese tipo de cobro (en línea, internacional) o está restringida | Otra tarjeta o habilitarlo con el banco |
| Número de tarjeta incorrecto | `14` | Dígitos mal escritos | Reenviar el enlace y pedir que revise |
| Código de seguridad (CVV) incorrecto | `N7`, `82`, `63` ✔ | CVV mal escrito | Reenviar el enlace |
| No se ubica la tarjeta | `25` | El emisor no encuentra la tarjeta | Otra tarjeta |
| Tarjeta vencida | `54` | Fecha de vencimiento pasada | Pedir tarjeta vigente |
| Datos de tarjeta incorrectos o tiempo agotado | `TO` ✔ | Tilopay/adquirente: datos incorrectos o tiempo agotado en el formulario (no está en la guía oficial todavía) | Reenviar el enlace; pedir que revise número, vencimiento y CVV |

## Rechazos del lado técnico (no dependen del cliente)

| Código | Texto típico | Causa | Acción |
|---|---|---|---|
| `95` ✔ | Reconcile error | Error de conciliación del procesador (no está en la guía oficial) | Reintentar; si se repite, soporte |
| `97` ✔ | Invalid 3DS field: browserLanguage | El sitio del comercio envía mal los campos 3DS (no está en la guía oficial) | Desarrollador: revisar la integración SDK/API |
| `96` | System malfunction | Falla del sistema del emisor o adquirente | Reintentar más tarde |
| `91` | Issuer unavailable | Banco emisor sin respuesta | Reintentar más tarde |

## Rechazos donde NO conviene insistir

| Etiqueta oficial | Códigos | Causa |
|---|---|---|
| Tarjeta bloqueada por el banco | `41`, `43`, `59` (también `B41`, `B43`) | Tarjeta reportada, retenida o bloqueada por seguridad |

Los códigos con prefijo `B` (`B51`, `B58`, …) son el mismo motivo que su número, reportado por otro adquirente.

Si un código no aparece aquí ni en la guía, muestra el `responseText` literal, trátalo como "rechazo general del banco" (como indica la guía) y sugiere reintentar con otra tarjeta.
