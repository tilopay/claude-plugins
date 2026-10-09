---
name: conciliacion-mensual
description: Conciliación contable de un período (mes, quincena o rango) para un comercio Tilopay usando el MCP de Tilopay — cruza las ventas aprobadas, los reembolsos y las liquidaciones depositadas, con los seis campos de costo etiquetados, y entrega un CSV o Excel listo para el contador más un resumen de diferencias. Úsala cuando el comercio o su contador pidan "conciliar", "cuadrar el mes", "exportar las transacciones del mes con comisiones", "cuánto me depositaron vs cuánto vendí" o un reporte para contabilidad.
---

# Conciliación mensual con el MCP de Tilopay

Objetivo: que el contador reciba un archivo donde cada venta aparece con su monto bruto, cada deducción con su nombre exacto, el neto, y en qué depósito de Tilopay cayó (o si sigue pendiente). Nada se estima: si un dato no vino, la celda queda vacía y se explica.

## Herramientas que usa

| Herramienta | Para qué |
|---|---|
| `tilopay_switch_commerce` | Confirmar el comercio activo (si hay varios, conciliar uno a la vez). |
| `tilopay_list_transactions` | Todas las filas del período (`status: "all"`, `limit: 500`, partir por semanas si hace falta). |
| `tilopay_refunds_list` | Historial de reembolsos y reversiones con su estado (`transactionStatus`: refund / reverse) y fecha. |
| `tilopay_settlements_list` | Liquidaciones ya depositadas: fecha de depósito `date_to_report`, período, `subtotal`, deducciones y `total` transferido, IBAN enmascarado. |
| `tilopay_settlement_transactions` | Transacciones incluidas en una liquidación concreta (por `settlement_id`). |
| `tilopay_settlement_balance` | Balance acumulado pendiente de depósito, por moneda, con desglose oficial. |
| `tilopay_billing_list` | Facturas que Tilopay le emite al comercio (son gasto del comercio, no venta). |

## Procedimiento

### 1. Período y moneda

Rango en hora local del comercio convertido a UTC (00:00 CR = 06:00 UTC). Pregunta la moneda solo si el comercio vende en más de una y no la dijo; de lo contrario concilia cada moneda por separado, nunca mezcladas.

### 2. Traer y clasificar las transacciones

`tilopay_list_transactions` con `status: "all"`, `limit: 500`. Si `total == limit` o la respuesta trae `truncated: true`, parte el rango por semanas y concatena. Excluye `test == 1`. Ordena tú las filas por `date` al armar el archivo (no dependas del parámetro `sort`).

Clasifica cada fila **solo por `code`**:

- `"1"` → venta aprobada (si `capture == 0`, es autorización sin capturar: va en una hoja aparte "Autorizaciones pendientes", no en ventas).
- `"1101"` → reembolso o reversión aprobada (`orderNumber` con prefijo `R-`/`Re-`); va en la hoja "Reembolsos" y resta.
- Cualquier otro → rechazada o incompleta; va en "Rechazadas" solo como referencia, sin montos en el total.

### 3. Deducciones por venta (seis campos, nombre exacto)

| Columna del archivo | Campo de la fila |
|---|---|
| Comisión | `commission` |
| IVA de la comisión | `iva_commission` |
| Retención renta | `rent` |
| Retención IVA | `retention_iva` |
| Costo por transacción | `cost` |
| IVA del costo | `iva_tilopay` |
| Total deducido | suma de las seis |
| Neto | `amount` − total deducido |

Verifica por fila que `amount − deducciones == neto` y, por moneda, que la suma de netos coincide con lo que reporta `tilopay_sales_summary` (`netToLiquidate`). Si no coincide, reporta ambas cifras y la diferencia; no ajustes ninguna.

### 4. Cruzar con las liquidaciones

1. `tilopay_settlements_list` (`limit: 100`, pagina si hace falta) y filtra las liquidaciones cuyo `period` o `date_to_report` toquen el rango. Ojo: una liquidación puede incluir ventas de días anteriores al período (p. ej. depósito del 2 del mes con ventas del 30 y 31 del anterior).
2. Para cada liquidación relevante, `tilopay_settlement_transactions(settlement_id)` y marca en la hoja de ventas la columna "Liquidación" con el `id` y la fecha de depósito de cada transacción encontrada. Si esa herramienta devuelve vacío para una liquidación con `subtotal > 0`, usa el campo `liquidation` de la fila (`1` = ya liquidada) y escribe "liquidada (detalle no disponible)".
3. Las ventas aprobadas sin liquidación asignada son "pendientes de depósito": su suma por moneda debe parecerse al `subtotal` de `tilopay_settlement_balance`; la diferencia típica son ventas de hoy aún no acumuladas.

Valida cada liquidación con la fórmula oficial: `subtotal − commission − iva_commission − retention − retention_iva − cost − iva_tilopay − transfer − transfer_iva == total` (tolerancia 0,01). Un `total` negativo es válido (costos fijos superaron las ventas del corte).

### 5. Reembolsos y facturas de Tilopay

- `tilopay_refunds_list` (`pending: false`, pagina): cada fila con `transactionStatus` (refund/reverse), monto, moneda, fecha y la orden original (`orderNumber` sin el prefijo). Cruza contra la hoja de ventas por orden original.
- Las filas `1101` de `tilopay_list_transactions` traen también `cost` e `iva_tilopay` (el costo fijo se cobra aunque la venta se reembolse) y `commission`/`rent`/`retention_iva` en 0. Inclúyelos en la hoja "Reembolsos" como deducción del reembolso; no los sumes a las deducciones de ventas.
- Una reversión (`Re-…`) de una autorización sin capturar (`capture == 0`) no es dinero devuelto: la venta nunca se cobró. Márcala como "autorización anulada".
- `tilopay_refunds_list` con `pending: true`: reembolsos solicitados aún no procesados → hoja "Reembolsos pendientes".
- `tilopay_billing_list`: facturas de Tilopay del período → hoja "Facturas Tilopay" (gasto del comercio; no se mezcla con deducciones por transacción).

### 6. Entregar

Un archivo (CSV si el comercio pide algo simple; Excel con una hoja por sección si pide "para el contador"), con estas hojas: Resumen, Ventas, Reembolsos, Rechazadas, Liquidaciones, Pendientes de depósito, Facturas Tilopay. En el chat, un resumen corto por moneda:

```
Conciliación septiembre 2026 — CRC

Ventas aprobadas: 128 por ₡4.812.500
Deducciones Tilopay: ₡287.430 (comisión ₡204.531 · IVA comisión ₡26.589 · ret. renta ₡84.699 · ret. IVA ₡255.063 · costo ₡…)
Neto de ventas: ₡4.525.070
Reembolsos: 3 por ₡45.000 → neto del período ₡4.480.070

Depositado en el período: 4 liquidaciones por ₡4.391.200 (última: 30 sep)
Pendiente de depósito al cierre: ₡88.870 (balance oficial: ₡88.870 ✓)

Diferencias encontradas: ninguna / 1 venta del 30 sep cayó en la liquidación del 2 oct (₡…)
```

Cierra diciendo qué no se pudo cruzar y por qué (p. ej. "2 liquidaciones sin detalle de transacciones en el API").

## Qué NO hacer

- No uses `responseText` para clasificar; solo `code`.
- No llames `tilopay_analyze_sales` para cifras.
- No inventes el detalle de una liquidación si la herramienta devuelve vacío.
- No muestres el IBAN completo (usa el enmascarado).
- No incluyas transacciones de prueba (`test == 1`) salvo que el comercio lo pida, y entonces en hoja aparte.
- No conviertas monedas: cada moneda se concilia por separado y el archivo no suma CRC con USD.
