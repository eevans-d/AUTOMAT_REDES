Tabla 6. Matriz de sanitización por endpoint/campo

| Endpoint                    | Campo                | Validación                     | Escape/Encoding requerido                                     | Regla CSP asociada                           | Prioridad |
|----------------------------|----------------------|--------------------------------|----------------------------------------------------------------|----------------------------------------------|----------|
| /api/whatsapp/test         | message              | Tipo string; longitud ≤ 2000   | HTML entities (quote=True); JSON ensure_ascii=False           | script-src 'self'; object-src 'none'         | P0       |
| /api/whatsapp/test         | user_id              | Tipo string; patrón alfanum.   | HTML entities (quote=True)                                    | script-src 'self'; object-src 'none'         | P0       |
| /api/nlp/classify          | message              | Tipo string; longitud ≤ 2000   | HTML entities (quote=True); filtrar on* y protocolos          | script-src 'self'; object-src 'none'         | P0       |
| /api/instagram/like        | username             | Alfanum. + [._]; len ≤ 30      | Escape en respuesta; validar previamente en manager           | script-src 'self'; object-src 'none'         | P0       |
| /api/instagram/follow      | username             | Alfanum. + [._]; len ≤ 30      | Escape en respuesta; validar previamente en manager           | script-src 'self'; object-src 'none'         | P0       |
| /api/instagram/dm          | username             | Alfanum. + [._]; len ≤ 30      | Escape en respuesta; validar previamente en manager           | script-src 'self'; object-src 'none'         | P0       |
| /api/instagram/dm          | message              | Tipo string; longitud ≤ 2000   | HTML entities (quote=True); filtrar on* y protocolos          | script-src 'self'; object-src 'none'         | P0       |
| /api/e2e/whatsapp-flow     | message              | Tipo string; longitud ≤ 2000   | HTML entities (quote=True); truncamiento seguro               | script-src 'self'; object-src 'none'         | P0       |

### Ejemplos aplicados

- Escapado de message y user_id en /api/whatsapp/test: aplicar html.escape(text, quote=True) antes de incluirlos en la respuesta JSON.
- Validación de username en /api/instagram/like y /api/instagram/follow: regex de aceptación y rechazo de patrones peligrosos; escape en la respuesta.
- Sanitización de message en /api/instagram/dm: escape HTML y filtrado de atributos y protocolos no permitidos.
- Ajuste del dashboard: preferir textContent para datos no confiables; eliminar innerHTML y construir nodos DOM con contenido escapado. De ser imprescindible usar innerHTML, aplicar escape de atributos y CSP.

## Controles transversales de seguridad (CORS, headers, CSP)

Para reducir la superficie de ataque y bloquear vectores de XSS avanzados:

- CORS: reemplazar "*" por una lista de orígenes permitidos; deshabilitar credenciales si no son necesarias; implementar validación de origen efectiva.
- Headers: establecer Strict-Transport-Security, X-Content-Type-Options: nosniff, X-Frame-Options: DENY o SAMEORIGIN, Referrer-Policy: no-referrer (o equivalente estricto).
- CSP: introducir una política restrictiva que prohíba scripts inline y event handlers, limite fuentes de script (script-src 'self' u orígenes explícitos), y bloquee object-src y base-uri.
- Rate limiting y autenticación: aunque la auditoría no detectó bypass evidente en pruebas puntuales, la ausencia de autenticación en endpoints protegidos es una preocupación que debe resolverse en paralelo.[^2]