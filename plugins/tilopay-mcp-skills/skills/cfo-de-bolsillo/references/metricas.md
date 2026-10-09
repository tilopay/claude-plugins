# Definiciones de métricas (CFO de bolsillo)

Todas por moneda, sobre filas de `tilopay_list_transactions` con `test == 0`.

| Métrica | Numerador | Denominador / regla |
|---|---|---|
| Ventas aprobadas | filas con `code == "1"` y `capture == 1` | — |
| Bruto | Σ `amount` de ventas aprobadas | — |
| Reembolsos | Σ `amount` de filas `code == "1101"` | reportar aparte; restan del neto del período |
| Deducciones | Σ `commission + iva_commission + rent + retention_iva + cost + iva_tilopay` (ventas aprobadas) | — |
| Neto de ventas | Bruto − Deducciones | validar: debe coincidir con `netToLiquidate` del resumen |
| Costo efectivo de Tilopay | Σ `commission + iva_commission + cost + iva_tilopay` | / Bruto, en % |
| Retenciones fiscales | Σ `rent + retention_iva` | / Bruto, en %; no son costo de Tilopay |
| Ticket promedio | Bruto | / cantidad de ventas aprobadas |
| Tasa de aprobación | ventas aprobadas | / (aprobadas + rechazos reales); excluir abandonos (`code` vacío, `99` con texto 3DS) y reembolsos |
| Variación | (actual − anterior) | / anterior; "sin base" si anterior == 0 |
| Concentración top 3 | Σ bruto de los 3 `email` con más bruto | / Bruto; riesgo si > 50 % |
| Mejor día / hora | conteo o bruto por `date` convertido a hora local (UTC−6 en Costa Rica) | solo con ≥ 30 ventas y ≥ 5 días |
| Proyección de mes | bruto acumulado + promedio diario de últimos 14 días × días restantes | etiquetar "estimación lineal" |

Autorizaciones sin capturar (`code == "1"` y `capture == 0`) no son ventas: cuéntalas aparte como "pendientes de captura".

Campos con nombre distinto en `tilopay_sales_summary`: `rent` → `retention_rent`, `iva_tilopay` → `cost_iva`.
