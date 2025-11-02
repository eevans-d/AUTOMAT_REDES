# AutoSocial Core: Análisis y Remediación Integral de Vulnerabilidades XSS (7+ instancias)

## Resumen ejecutivo

El sistema AutoSocial Core presenta vulnerabilidades de Cross-Site Scripting (XSS) en rutas críticas de su API y en su interfaz de dashboard. El análisis de código confirma al menos siete instancias concretas de XSS (reflected y event-handler) en endpoints que procesan entradas del usuario y devuelven contenido no escapado en respuestas JSON. A ello se suma un vector DOM-XSS en el frontend embebido en el propio dashboard, que usa textContent con datos provenientes de la API sin escapado de atributos. El veredicto de auditoría se mantiene en NO-GO para producción mientras persistan estas debilidades, dado su alto impacto en la ejecución de script en navegadores de usuarios y su interacción con configuraciones inseguras como CORS permisivo y ausencia de Content Security Policy (CSP) y otros headers de seguridad.[^1][^2]

Las siete instancias verificadas en código son:

1. server_simple.py:416 – POST /api/whatsapp/test – XSS reflected por reflejar message sin escape.
2. server_simple.py:416 – POST /api/whatsapp/test – XSS reflected por reflejar user_id sin escape.
3. server_simple.py:448 – POST /api/nlp/classify – XSS reflected por reflejar message sin escape.
4. server_simple.py:483 – POST /api/instagram/like – XSS reflected por reflejar username sin escape.
5. server_simple.py:509 – POST /api/instagram/follow – XSS reflected por reflejar username sin escape.
6. server_simple.py:532 – POST /api/instagram/dm – XSS reflected por reflejar username y message sin escape.
7. server_simple.py:688 – POST /api/e2e/whatsapp-flow – XSS reflected por reflejar message sin escape.

Como riesgos transversales, el sistema expone un CORS permisivo ("*"), carece de headers de seguridad (CSP, X-Frame-Options, HSTS, X-Content-Type-Options, Referrer-Policy), y su dashboard integra respuestas JSON directamente en el DOM mediante innerHTML, facilitando manipulación de atributos y potencial inyección de eventos en determinados contextos. Estos factores elevan el alcance y la probabilidad de explotación de XSS en escenarios reales.[^1][^2]

Para habilitar el paso a producción, las remediaciones deben eliminar el escape de salida en todas las rutas señaladas, implantar una política CSP restrictiva, endurecer CORS, incorporar validación whitelisting en endpoints clave (Instagram), e introducir encoding por contexto (HTML, atributos, URL, JSON) en servicios y plantillas. La ejecución deberá acompañarse de un plan de pruebas y re-testing con payloads estándar, además de un checklist de headers y controles adicionales que aseguren defensa en profundidad.

Para orientar la priorización, la siguiente matriz resume las siete instancias confirmadas y la acción de remediación requerida:

Tabla 1. Matriz de resumen de vulnerabilidades XSS confirmadas

| # | Endpoint                                   | Método | Ubicación (línea) | Tipo de XSS      | Causa raíz                                    | Severidad | Acción de remediación principal                            | Prioridad |
|---|--------------------------------------------|--------|-------------------|------------------|-----------------------------------------------|-----------|-------------------------------------------------------------|----------|
| 1 | /api/whatsapp/test                         | POST   | 416               | Reflected        | received_message sin escape                    | Alta      | Escapado HTML + JSON (quote=True)                          | P0       |
| 2 | /api/whatsapp/test                         | POST   | 416               | Reflected        | user_id sin escape                             | Alta      | Escapado HTML + JSON (quote=True)                          | P0       |
| 3 | /api/nlp/classify                          | POST   | 448               | Reflected        | original_message sin escape                    | Alta      | Escapado HTML + JSON (quote=True)                          | P0       |
| 4 | /api/instagram/like                        | POST   | 483               | Reflected        | username sin escape                            | Alta      | Validación alfanumérico + escape en respuesta              | P0       |
| 5 | /api/instagram/follow                      | POST   | 509               | Reflected        | username sin escape                            | Alta      | Validación alfanumérico + escape en respuesta              | P0       |
| 6 | /api/instagram/dm                          | POST   | 532               | Reflected        | username y message sin escape                  | Alta      | Validación alfanumérico + escape de mensaje                | P0       |
| 7 | /api/e2e/whatsapp-flow                     | POST   | 688               | Reflected        | original_message sin escape                    | Alta      | Escapado HTML + JSON (quote=True)                          | P0       |

Estas acciones, junto con CSP y ajustes de CORS, deben completarse antes de cualquier despliegue productivo, conforme al plan de fases de remediación detallado más adelante.[^1][^2]