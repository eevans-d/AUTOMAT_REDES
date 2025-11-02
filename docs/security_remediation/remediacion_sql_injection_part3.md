Tabla 2. Archivos y funciones relevantes con uso de consultas a SQLite

| Archivo | Función/método | Propósito | Entrada de usuario | Uso de DB | Observaciones |
|---|---|---|---|---|---|
| instagram_manager_backup.py | _track_action(...) [506–515] | Registro de acciones de Instagram | action_type, target_username, status, message, error_message | INSERT en instagram_actions | Potencial concatenación en valores; requiere validación estricta |
| postiz_manager_backup_original_20251102_131125 | delete_campaign(...) [592–598] | Actualización de estado y limpieza | campaign_id | UPDATE con LIKE '%%%s%%' | Riesgo crítico por拼接 del valor en LIKE |
| server_simple.py | save_interaction(...) [1010–1024] | Persistencia de interacciones | user_id, platform, message, intent, confidence | INSERT en interactions | Concatenación de valores; sin placeholders |
| server_simple.py | handle_metrics(...) [839–864] | Métricas agregadas | N/A (lecturas) | SELECT COUNT, GROUP BY | Consultas directas sin parámetros; riesgo en reuso de lógica |
| postiz_manager_backup_original_20251102_131125 | delete_campaign(...) [577–607] (alcance extendido) | Flujo completo de borrado | campaign_id | UPDATE/SELECT | Interacción directa con identificadores de canal y campaña |

### Módulos y endpoints auditados

- GET /api/instagram/user/{username}: la entrada de ruta se refleja en lógica que puede invocar tracking o consultas en la tabla instagram_actions.
- GET /api/postiz/next-slot/{channel_id}: la entrada se usa para decisiones de scheduling; delete_campaign concatena campaña por LIKE, evidenciando vulnerabilidad.
- POST /api/whatsapp/test: persiste interacciones con concatenación directa de campos.
- POST /api/nlp/classify: expone métricas agregadas con executes directos sin parámetros.
- POST /api/e2e/whatsapp-flow: reutiliza la ruta de métricas y persiste interacciones, combinando vectores.

## Panorama de vulnerabilidades de SQLi en AutoSocial Core

Las vulnerabilidades se concentran en dos patrones:

- Concatenación de entradas del usuario en la construcción de consultas SQL (cláusulas LIKE, WHERE, VALUES).
- Executes directos de cadenas formateadas sin placeholders ni validaciones robustas.

La severidad es crítica porque los endpoints permiten manipular parámetros de ruta y cuerpo (username, channel_id, message, user_id) que llegan sin sanitización suficiente y acceden a tablas sensibles (interacciones, campañas, acciones), exponiendo datos y comprometiendo la integridad de la base.