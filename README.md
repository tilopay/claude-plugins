# Plugins de Tilopay para Claude

Skills oficiales para usar el [MCP de Tilopay](https://tilopay.com/developers/agentes/mcp) desde Claude Code y Cowork.

## Instalar en Claude Code

Dentro de Claude Code, escribí estos dos comandos:

```
/plugin marketplace add tilopay/claude-plugins
/plugin install tilopay-mcp-skills@tilopay
```

Listo. Desde ese momento, cuando pidas algo como «dame el cierre de caja de hoy» o «cóbrale 25 mil colones a Ana», Claude carga la skill correspondiente y usa tu conector de Tilopay.

Necesitás tener el MCP de Tilopay conectado: <https://tilopay.com/developers/agentes/mcp/conectar>

## Qué incluye `tilopay-mcp-skills`

| Skill | Para qué |
|---|---|
| `cierre-de-caja` | Cierre del día, la semana o el mes por moneda, con comisiones, retenciones y próximo depósito. |
| `cobrar-por-whatsapp` | Convierte una frase en un enlace de pago y el mensaje de WhatsApp listo para enviar. |
| `rechazos-y-recuperacion` | Analiza pagos rechazados, los agrupa por motivo y propone cómo recuperarlos. |
| `conciliacion-mensual` | Conciliación contable de un período con CSV/Excel para el contador. |
| `cfo-de-bolsillo` | Comparación de períodos, ticket promedio, mejores días y proyección de mes. |
| `catalogo-y-cobros-fijos` | Catálogo de productos con enlace de pago y QR, y conexión a redes. |

Documentación completa y descargas para claude.ai: <https://tilopay.com/developers/agentes/mcp/skills>

## Actualizar

```
/plugin marketplace update tilopay
/plugin update tilopay-mcp-skills@tilopay
```

## Soporte

soporte@tilopay.com
