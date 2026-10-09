# Campos de una transacción de Tilopay (`tilopay_list_transactions`)

Verificado contra respuestas reales del MCP (octubre 2026).

| Campo | Significado | Notas |
|---|---|---|
| `code` / `response` / `responseCode` | Código de respuesta del procesador | `"1"` aprobada · `"1101"` reembolso/reversión aprobada · otro = rechazada o incompleta |
| `responseText` | Texto del procesador | Solo informativo. No sirve para decidir aprobación. |
| `orderNumber` | Número de orden del comercio | Prefijo `R-` o `Re-` = reembolso/reversión de la orden original |
| `amount` | Monto bruto (string) | Convertir a número |
| `currency` | `CRC`, `USD`, … | Agrupar siempre por moneda |
| `commission` | Comisión de Tilopay (porcentual) | |
| `iva_commission` | IVA sobre la comisión | |
| `rent` | Retención de renta | En `tilopay_sales_summary` aparece como `retention_rent` (las filas v2 traen ambos nombres) |
| `retention_iva` | Retención de IVA | |
| `cost` | Costo fijo por transacción | En CRC viene ya convertido a colones |
| `iva_tilopay` | IVA sobre el costo fijo | En `tilopay_sales_summary` aparece como `cost_iva` |
| `date` / `created_at` | Fecha en UTC | Restar 6 h para Costa Rica |
| `test` | `1` = transacción de pruebas | Excluir por defecto |
| `capture` | `1` capturada · `0` solo autorizada | Una autorización sin capturar aún no es dinero cobrado |
| `liquidation` | `1` si ya fue incluida en una liquidación | |
| `dispute` | `1` si tiene contracargo | |
| `platform` | Canal: `TilopayLink`, `sdk`, `API`, `TilopayRepeat` (recurrente), `StaticPagePayment` (QR estático), `agente whatsapp tilopay` | Útil para explicar de dónde vino la venta |
| `email`, `cardholder`, `lastname` | Datos del cliente | `tilopay_find_payment` busca por `email`, no por `cardholder` |
| `brand`, `acquirer` | Marca de tarjeta y adquirente | |
| `paymentLabel.title_es` | Método: Tarjeta, SINPE Móvil, Recurrente… | |

## Códigos de rechazo vistos en producción

| Código | Texto típico | Qué significa en la práctica |
|---|---|---|
| `TO` | "verifique que los datos de su tarjeta estén correctos" | Datos incorrectos o tiempo agotado en el formulario |
| `51` | Insufficient funds | Fondos insuficientes |
| `58` | Transaction not allowed | El banco emisor no permite la operación (tarjeta no habilitada para compras en línea / internacionales) |
| `63` | Service Not Allowed | Servicio no permitido por el emisor |
| `12` | "La transacción ha sido cerrada" | Orden ya cerrada; típico en reintentos de reembolso |
| `95` | Reconcile error | Error de conciliación del procesador |
| `97` | Invalid 3DS field | Error de integración en 3DS (lado del desarrollador) |
| `99` | Inicio verificación 3DS | El cliente no terminó la autenticación 3DS |
| `""` (vacío) | — | Intento abandonado sin respuesta del procesador |
