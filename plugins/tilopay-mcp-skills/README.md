# Skills de Tilopay para agentes de IA

Seis skills que usan el MCP de Tilopay (https://mcp.tilopay.com/mcp). Requieren tener el MCP conectado en el cliente.

- cierre-de-caja — ventas del día/semana/mes, deducciones exactas, neto y próximo depósito
- cobrar-por-whatsapp — crea un cobro desde una frase y arma el mensaje de WhatsApp
- rechazos-y-recuperacion — rechazos por motivo, patrones y acciones para recuperar la venta
- conciliacion-mensual — ventas, reembolsos y liquidaciones cruzadas para el contador
- cfo-de-bolsillo — comparación de períodos, costo efectivo y proyección
- catalogo-y-cobros-fijos — productos, QR y feed del catálogo

Instalación
- Claude Code / Cowork (plugin): `claude plugin add ./tilopay-mcp-skills` o copiar la carpeta `skills/` dentro de `~/.claude/skills/`.
- claude.ai (web/escritorio): Configuración → Capacidades → Skills → subir cada carpeta de `skills/` como .zip (una skill por .zip).
