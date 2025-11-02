Tabla 4. Mapa de validación por endpoint

| Endpoint                    | Campo(s)                  | Validación actual             | Patrones bloqueados          | Longitud máxima | Propuesta de corrección                                                                 |
|----------------------------|---------------------------|-------------------------------|------------------------------|-----------------|-----------------------------------------------------------------------------------------|
| /api/whatsapp/test         | message, user_id          | Ninguna                       | —                            | —               | Escapar HTML (quote=True); validar tipos; truncar a límites razonables                 |
| /api/nlp/classify          | message                   | Ninguna                       | —                            | —               | Escapar HTML (quote=True); filtrar si contiene etiquetas/handlers                      |
| /api/instagram/like        | username                  | Ninguna                       | —                            | —               | Validar alfanumérico + [._]; longitud <= 30; rechazar caracteres prohibited           |
| /api/instagram/follow      | username                  | Ninguna                       | —                            | —               | Validar alfanumérico + [._]; longitud <= 30; rechazar caracteres prohibited           |
| /api/instagram/dm          | username, message         | Ninguna                       | —                            | —               | Validar username alfanumérico; escapar message; bloquear on* y protocolos peligrosos  |
| /api/e2e/whatsapp-flow     | message                   | Ninguna                       | —                            | —               | Escapar HTML (quote=True); truncar a longitud segura                                  |

## Revisión de templates y responses API (JSON y HTML)

El servidor codifica respuestas JSON con json.dumps sin ensure_ascii=False ni escape de HTML. Además, el dashboard embebido inserta datos mediante innerHTML, lo que abre puertas para ejecución si el contenido procede de respuestas no sanitizadas. La ausencia de una política CSP que bloquee inline scripts y event handlers agrava la situación.

Tabla 5. Endpoints con respuestas JSON y mitigaciones

| Endpoint                    | Content-Type actual         | Uso de json.dumps | Campos reflejar usuario     | Mitigación requerida                                                                 |
|----------------------------|-----------------------------|-------------------|------------------------------|--------------------------------------------------------------------------------------|
| /api/whatsapp/test         | application/json            | Sí                | received_message, user_id    | Escapar HTML (quote=True) en esos campos; validación de tipos; truncar longitudes   |
| /api/nlp/classify          | application/json            | Sí                | original_message             | Escapar HTML (quote=True); filtrado de etiquetas/handlers                            |
| /api/instagram/like        | application/json            | Sí                | username                     | Validación alfanumérico + escape; validación previa en manager                      |
| /api/instagram/follow      | application/json            | Sí                | username                     | Validación alfanumérico + escape; validación previa en manager                      |
| /api/instagram/dm          | application/json            | Sí                | username, message            | Validación alfanumérico + escape HTML (quote=True); longitud acotada                |
| /api/e2e/whatsapp-flow     | application/json            | Sí                | original_message             | Escapar HTML (quote=True); truncamiento seguro                                      |

## Propuesta de sanitización y escaping por contexto

La defensa eficaz exige un enfoque por contexto, combinando validación whitelisting, escapado adecuado y políticas de seguridad.

- HTML entity encoding (quote=True): aplicar en todos los campos reflejados en respuestas JSON que puedan terminar en el DOM.
- Atributos: escape de comillas y bloqueo de caracteres peligrosos (espacios, comillas, ángulo).
- URL: validación estricta de esquemas (solo http/https), normalización y rechazo de javascript:, data:, file:, vbscript:.
- JSON: asegurar ensure_ascii=False y escapar caracteres especiales al injectar en HTML.
- Whitelisting de usernames: alfanumérico con punto y guion bajo; longitudes y patrones maliciosos.
- CSP restrictiva: bloquear inline scripts y event handlers, limitar script-src y object-src.
- Manager de Instagram seguro: adoptar patrones de validación y sanitización del módulo de referencia.