Tabla 1. Resumen de las 6 instancias de SQLi (endpoint, módulo/línea, tipo de query, severidad, causa raíz, estado)

| Endpoint afectado (auditoría) | Módulo / línea | Tipo de query | Severidad | Causa raíz | Estado |
|---|---|---|---|---|---|
| GET /api/instagram/user/{username} | instagram_manager_backup.py:506–515 | INSERT (tracking) | Crítica | Concatenación en métodos auxiliares (rutina no visible completa); potencial exposición vía endpoint | Por remediación |
| GET /api/postiz/next-slot/{channel_id} | postiz_manager_backup:592–598 | UPDATE (LIKE sin proteger) | Crítica | Plantilla SQL con拼接 LIKE '%'+{id}+'%' | Por remediación |
| POST /api/whatsapp/test | server_simple.py:1010–1024 | INSERT (interacciones) | Crítica | Concatenación y formateo directo de valores sin parámetros | Por remediación |
| POST /api/nlp/classify | server_simple.py:839–864 | SELECT agregaciones (metrics) | Crítica | Múltiples executes directos sin parámetros en consultas agregadas | Por remediación |
| POST /api/e2e/whatsapp-flow | server_simple.py:839–864 | SELECT agregaciones (metrics, reuso) | Crítica | Reutilización de lógica vulnerable en métricas | Por remediación |
| GET /api/postiz/next-slot/{channel_id} | postiz_manager_backup:577–607 | SELECT (no evidenciada SQLi, pero interacción con canal) | Crítica | Uso directo de channel_id en lógica de negocio y potencial concatenación en rutinas llamadas | Por remediación |

Nota: La severidad "Crítica" se asigna por exposición a entradas no confiables y posibilidad de manipular cláusulas SQL (WHERE, LIKE, VALUES), con riesgo de exfiltración, alteración y denegación de servicio.

### Contexto y verificación de la auditoría

- El sistema fue probado con payloads típicos de SQLi: comilla para romper la consulta, operadores lógicos (OR 1=1) para evadir filtros, UNION para manipular el resultado y punto y coma para encadenar comandos. Estos payloads provocaron errores SQL visibles en las respuestas HTTP en 6 endpoints.
- La evidencia recuperada incluye errores SQL en respuestas de endpoints de Instagram (usuario), WhatsApp (test), NLP (clasificación) y E2E (flujo WhatsApp). Aunque el detalle de logs completos no está disponible, el reporte OWASP es consistente con patrones de concatenación de entradas del usuario en consultas.

## Alcance, archivos analizados y metodología

El análisis se basa en las revisiones de los siguientes módulos: instagram_manager_backup.py y postiz_manager_backup_original_20251102_131125 (versión con timestamp), además de server_simple.py. Se priorizaron las funciones que ejecutan consultas contra SQLite y que reciben entradas del usuario o parámetros de rutas y cuerpo HTTP. La metodología consistió en:

- Rastrear dónde se concatenan entradas del usuario dentro de cadenas SQL (por ejemplo, en cláusulas LIKE, WHERE o VALUES).
- Clasificar por tipo de consulta (SELECT, INSERT, UPDATE) y evaluar severidad según la posibilidad de manipular la lógica de la consulta.
- Proponer consultas parametrizadas (placeholders de SQLite) y validaciones de entrada que neutralicen la inyección. Donde el motor lo permita, se incluyen recomendaciones para migrar a PostgreSQL y usar placeholders de psycopg2.