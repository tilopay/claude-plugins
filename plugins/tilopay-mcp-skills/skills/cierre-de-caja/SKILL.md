---
name: cierre-de-caja
description: Cierre de caja de un comercio Tilopay con el MCP de Tilopay — ventas del día, semana o mes por moneda, aprobadas vs rechazadas, reembolsos, desglose exacto de comisiones y retenciones, neto a liquidar y próxima liquidación, con validación de que los números cuadran. Úsala cuando el comercio pida "cierre de caja", "cuánto vendí hoy/ayer/esta semana", "cuánto me van a depositar", "cuánto me cobró Tilopay de comisión" o un resumen de ventas de un período.
---

# Cierre de caja con el MCP de Tilopay

Entrega al dueño del comercio un cierre de caja claro y que cuadre, usando únicamente datos de las herramientas del MCP de Tilopay. Nunca inventes montos ni rellenes huecos: si un dato no vino, dilo.

## Herramientas que usa

| Herramienta | Para qué |
|---|---|
| `tilopay_switch_commerce` (sin parámetros) | Confirmar qué comercio está activo. Si la cuenta tiene varios, pregunta cuál antes de seguir. |
| `tilopay_list_transactions` | Fuente de verdad del período: todas las filas con sus campos de costo. |
| `tilopay_sales_summary` | Métricas calculadas (ticket promedio, tasa de aprobación, por día/hora, motivos de rechazo). Solo complementa; el cuadre se hace con las filas. |
| `tilopay_settlement_balance` (sin parámetros) | Dato OFICIAL: próxima fecha de depósito, frecuencia, IBAN enmascarado y balance acumulado pendiente por moneda con su desglose. |
| `tilopay_settlements_list` | Depósitos ya realizados ("¿ya me depositaron lo del viernes?"). |

## Procedimiento

### 1. Fijar el período en hora local

El comercio habla en su hora local (Costa Rica = UTC-6; si el perfil indica otro país, usa esa zona). Las herramientas reciben fechas `"YYYY-MM-DD HH:mm:ss"` y devuelven el campo `date` en UTC.

- "hoy" → desde las 00:00:00 locales de hoy hasta ahora.
- "ayer", "esta semana", "este mes", "la semana pasada" → rangos completos en hora local.
- Convierte el rango local a UTC antes de llamar (00:00 CR = 06:00 UTC). Si no conviertes, las ventas de la noche caen en el día equivocado.

### 2. Traer las transacciones

```
tilopay_list_transactions
  startDate, endDate (en UTC, formato "YYYY-MM-DD HH:mm:ss")
  status: "all"
  sort: "newest"
  limit: 500
```

Reglas de lectura de cada fila (ver `references/campos-transaccion.md`):

- **Aprobada** ⇔ `code == "1"`. Nada más. No uses el texto `responseText` para decidir (dice "aprobada" también en reembolsos y "no aprobada" en algunos rechazos).
- **Reembolso o reversión aprobada** ⇔ `code == "1101"`. Su `orderNumber` empieza por `R-` o `Re-`. Se reportan aparte y restan del neto; no son ventas ni rechazos.
- **Rechazada** ⇔ cualquier otro `code` (`TO`, `51`, `58`, `12`, `95`, `""`, etc.) en una fila cuyo `orderNumber` NO empieza por `R-`/`Re-`.
- Excluye las filas con `test == 1` salvo que el comercio pida ver pruebas.
- Si `total` en la respuesta es igual al `limit`, el período tiene más filas de las que trajiste: pártelo en sub-rangos (por semana) y vuelve a llamar. Nunca reportes un total parcial como si fuera completo.

### 3. Calcular por moneda (nunca mezcles CRC y USD)

Para cada moneda, con las filas **aprobadas** (`code == "1"`):

```
bruto            = Σ amount
comision         = Σ commission
iva_comision     = Σ iva_commission
retencion_renta  = Σ rent
retencion_iva    = Σ retention_iva
costo_tilopay    = Σ cost
iva_costo        = Σ iva_tilopay
total_deducido   = comision + iva_comision + retencion_renta + retencion_iva + costo_tilopay + iva_costo
neto_ventas      = bruto − total_deducido
reembolsos       = Σ amount de las filas code == "1101"
neto_periodo     = neto_ventas − reembolsos
```

Nombra cada campo exactamente así en el reporte. "Comisión" a secas mezcla seis conceptos distintos y produce cifras que no se pueden conciliar con el panel.

### 4. Validar antes de mostrar

1. `bruto − total_deducido == neto_ventas` (tolerancia 0.01).
2. Cuenta de filas: aprobadas + rechazadas + reembolsos == filas del período (sin pruebas).
3. Si llamaste `tilopay_sales_summary`, su `payments.approved` y `grossAmount` por moneda deben coincidir con tu conteo. Si no coinciden, reporta tus cifras (vienen de las filas) y menciona la diferencia en una línea; no promedies ni "ajustes".
4. Cruza con `tilopay_settlement_balance`: en `accumulated_balance_details` de cada moneda debe cumplirse `subtotal − commission − iva_commission − retention − retention_iva − cost − iva_tilopay − transfer − transfer_iva == total`. Ese `total` es lo que Tilopay va a depositar (puede ser negativo si los costos fijos superan las ventas; dilo con claridad, sin alarmar).

### 5. Entregar el cierre

Formato corto, en el idioma del comercio, una sección por moneda. Ejemplo para CRC:

```
Cierre de caja — martes 7 de octubre (hora Costa Rica)

Ventas aprobadas: 12 por ₡486.500
Rechazadas: 2 (₡35.000 no cobrados) — motivos: fondos insuficientes (1), datos de tarjeta (1)
Reembolsos: 1 por ₡12.000

Deducciones de Tilopay
  Comisión            ₡20.676
  IVA de la comisión   ₡2.688
  Retención renta      ₡8.562
  Retención IVA       ₡25.825
  Costo por transacción ₡4.200
  IVA del costo          ₡546
  Total deducido      ₡62.497

Neto a liquidar por estas ventas: ₡424.003
Menos reembolsos: ₡412.003

Próximo depósito de Tilopay: jueves 9 de octubre, a la cuenta CR96…9336 (frecuencia: diario)
Balance acumulado pendiente (oficial): ₡xxx
```

Cierra con una o dos observaciones verificables con esos datos (p. ej. "los 2 rechazos son del mismo cliente, vale la pena escribirle"). Si la muestra es pequeña (`sample.sufficient == false` en el resumen), no hables de tendencias.

## Qué NO hacer

- No pegues JSON ni IDs internos. El comercio quiere montos y fechas.
- No muestres el IBAN completo; usa el enmascarado que devuelve la herramienta.
- No llames `tilopay_analyze_sales` para el cierre: es un informe narrativo, no una fuente de cifras.
- No digas "aproximadamente" en un cierre de caja: o cuadra o explicas qué fila no cuadra.
- Si el comercio pide el cierre automático por WhatsApp cada noche, eso es `tilopay_cash_close` con `action: "set"` (hora local y días); confírmalo antes de activarlo porque solo el dueño puede configurarlo.
