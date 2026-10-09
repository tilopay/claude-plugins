---
name: cfo-de-bolsillo
description: Análisis financiero de ventas de un comercio Tilopay con el MCP de Tilopay — compara períodos (este mes vs el anterior, esta semana vs la pasada), ticket promedio, mejores días y horas, concentración de clientes, costo efectivo de Tilopay como porcentaje de la venta y una proyección simple de ingresos y de flujo a 7/30 días con el próximo depósito. Úsala cuando el comercio pregunte "cómo voy", "cuánto crecí", "cuál es mi mejor día", "cuánto me cuesta Tilopay en porcentaje", "cuánto voy a tener a fin de mes" o pida un análisis de tendencias.
---

# CFO de bolsillo con el MCP de Tilopay

Responde preguntas de negocio con números que salen de las transacciones reales, comparaciones justas (mismo largo de período, misma moneda) y proyecciones etiquetadas como estimación. Solo afirma lo que la muestra sustenta.

## Herramientas que usa

| Herramienta | Para qué |
|---|---|
| `tilopay_list_transactions` | Fuente de verdad: todas las filas de los períodos a comparar (`status: "all"`, `limit: 500`, partir por semanas si se trunca). |
| `tilopay_sales_summary` | Métricas calculadas para cruzar (tasa de aprobación, ticket promedio, `trends`, `sample.sufficient`). |
| `tilopay_settlement_balance` | Próximo depósito y balance pendiente por moneda (para el flujo). |
| `tilopay_settlements_list` | Depósitos ya realizados (para el flujo histórico). |
| `tilopay_get_business_profile` | Giro, moneda principal y país (zona horaria) del comercio. |

## Procedimiento

### 1. Definir la pregunta y los períodos

- "Cómo voy este mes" → mes actual hasta hoy vs **los mismos días** del mes anterior (del 1 al mismo día), no el mes anterior completo.
- "Esta semana vs la pasada" → lunes a hoy vs lunes a mismo día de la semana anterior.
- "Últimos 30 días" → 30 días vs los 30 anteriores.
- Si el comercio no da período, usa los últimos 30 días vs los 30 anteriores y dilo.

Rangos en hora local del comercio convertidos a UTC (Costa Rica: 00:00 = 06:00 UTC). Una moneda a la vez; si vende en dos, dos bloques, nunca sumados ni convertidos.

### 2. Traer las filas de ambos períodos

Dos llamadas a `tilopay_list_transactions` (una por período). Excluye `test == 1`. Clasifica solo por `code`: `"1"` venta aprobada (con `capture == 1`), `"1101"` reembolso, resto rechazo o incompleto. Si `total == limit` o `truncated: true`, parte por semanas.

### 3. Calcular por moneda (ver `references/metricas.md`)

Para cada período:

- Ventas: cantidad y bruto (Σ `amount` aprobadas), reembolsos (Σ `amount` 1101), **neto de ventas** = bruto − Σ seis campos de costo (`commission`, `iva_commission`, `rent`, `retention_iva`, `cost`, `iva_tilopay`).
- Ticket promedio = bruto / cantidad de aprobadas; mediana y máximo si hay ≥ 10 ventas.
- Tasa de aprobación = aprobadas / (aprobadas + rechazadas reales), excluyendo abandonos (`code` vacío o `99` con texto 3DS) y reembolsos.
- **Costo efectivo de Tilopay** = Σ (commission + iva_commission + cost + iva_tilopay) / bruto, en %. Las retenciones (`rent`, `retention_iva`) son impuestos retenidos, repórtalas aparte: no son costo de Tilopay.
- Distribución por día de la semana y por hora **calculada por ti** desde `date` convertido a hora local (no confíes en `byWeekday`/`byHour` del resumen).
- Concentración: % del bruto que aportan los 3 correos con más ventas; marca riesgo si > 50 %.
- Por canal (`platform`) y por método (`paymentLabel.title_es`) si aporta algo.

Variación = (actual − anterior) / anterior; si el anterior es 0, di "sin base de comparación" en vez de un porcentaje.

### 4. Validar

- Cruza bruto, cantidad y tasa de aprobación con `tilopay_sales_summary` del mismo rango; si difieren, reporta tus cifras y la diferencia.
- Muestra insuficiente (`sample.sufficient == false`, o < 30 ventas o < 5 días con ventas): entrega los totales y omite tendencias, mejores días/horas y proyecciones, diciéndolo.

### 5. Proyección y flujo (solo con muestra suficiente)

- Proyección de cierre de mes = bruto acumulado + promedio diario de los últimos 14 días × días restantes. Etiqueta: "estimación lineal, no considera estacionalidad".
- Flujo a 7/30 días: `tilopay_settlement_balance` da la fecha del próximo depósito y el `total` pendiente por moneda (puede ser negativo si los costos fijos superan las ventas; dilo con naturalidad). Suma el neto proyectado de las ventas futuras según la frecuencia de liquidación (`commerce.liquidation`: diario, semanal, etc.).

### 6. Entregar

Un bloque por moneda, cuatro a seis líneas de cifras y dos o tres observaciones accionables; nada de JSON ni campos internos.

```
Septiembre (1–30) vs agosto (1–30) — CRC

Ventas: 128 por ₡4.812.500 (+12 % en monto, +4 % en cantidad)
Ticket promedio: ₡37.600 (+8 %)
Tasa de aprobación: 91 % (agosto 87 %)
Costo efectivo de Tilopay: 4,9 % del bruto · Retenciones fiscales: 2,0 %
Neto de ventas: ₡4.480.070

Mejor día: viernes (28 % de las ventas) · Mejor franja: 6–9 pm
3 clientes concentran el 61 % de la venta → riesgo de concentración

Proyección octubre (estimación lineal): ₡5,1 M
Próximo depósito: jueves 9 de octubre, ₡88.870 netos
```

## Qué NO hacer

- No uses `tilopay_analyze_sales` como fuente de cifras; si lo usas, solo para redactar y verifica cada número.
- No compares períodos de distinto largo ni mezcles monedas.
- No presentes la proyección como pronóstico: siempre "estimación".
- No incluyas clientes de prueba (`@example.com`, `test`) en la concentración si el comercio tiene cuenta de pruebas mezclada; dilo.
- No sugieras cambios de precio, de adquirente o de comisiones; esos los decide el comercio con Tilopay.
