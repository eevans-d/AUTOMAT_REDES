Tabla 3. Endpoints vs instancias vs tipo de query vs causa raíz vs evidencia

| Endpoint | Instancia | Tipo de query | Causa raíz | Evidencia |
|---|---|---|---|---|
| GET /api/instagram/user/{username} | instagram_manager_backup.py:506–515 | INSERT | Concatenación en tracking | Auditoría reporta error SQL con payloads tipo ' UNION SELECT… |
| GET /api/postiz/next-slot/{channel_id} | postiz_manager_backup:592–598 | UPDATE | LIKE '%%%s%%' sin parámetros | Auditoría reporta error SQL; patrón LIKE vulnerable |
| POST /api/whatsapp/test | server_simple.py:1010–1024 | INSERT | Valores interpolados | Errores SQL en pruebas de WhatsApp |
| POST /api/nlp/classify | server_simple.py:839–864 | SELECT | Executes directos en métricas | Errores SQL en clasificación NLP |
| POST /api/e2e/whatsapp-flow | server_simple.py:839–864 | SELECT | Reuso de métricas vulnerables | Errores SQL en flujo E2E |
| GET /api/postiz/next-slot/{channel_id} | postiz_manager_backup:577–607 | SELECT | Uso directo de channel_id | Error SQL asociado a rutas Postiz |

### Observaciones adicionales sobre el entorno de base de datos

- El sistema usa SQLite, con patrón de conexión sqlite3 y executes directos en varios puntos.
- Observaciones operativas relevantes: journal_mode=WAL y foreign_keys=ON, útiles para integridad y concurrencia, no para prevenir SQLi.
- La transición a PostgreSQL se planificación como parte de la preparación para producción, con adaptación de placeholders (Psycopg2 uses %) y políticas de acceso.

## Inventario detallado de las 6 instancias de SQLi y remediaciones propuestas

A continuación se detallan las instancias, su causa raíz, patrón de explotación y la remediación propuesta con consultas parametrizadas y validaciones de entrada. Los fragmentos de código muestran el "antes" y "después" en SQLite, con notas para PostgreSQL.

### Instancia 1: Módulo instagram_manager_backup.py (líneas 506–515), método _track_action (INSERT)

- Ubicación y propósito: _track_action registra acciones de Instagram en la tabla instagram_actions. La función recibe action_type, target_username, status, message, error_message.
- Tipo de query: INSERT.
- Causa raíz: probable concatenación de valores dentro de la cadena SQL (patrón visto en managers legados). Sin placeholders, cualquier comilla o operador puede alterar la consulta.
- Explotación: payloads en username o message pueden romper la sentencia y ejecutar SQL arbitrario.
- Propuesta de solución:
  - Validar entradas con listas blancas (regex) y límites de longitud.
  - Reescribir el INSERT para usar placeholders de SQLite (?). Por ejemplo:

Fragmento "antes" (ilustrativo):
```python
# Ejemplo de patrón inseguro (concatenación de valores)
sql = f"INSERT INTO instagram_actions (action_type, target_username, status, message, error_message) " \
      f"VALUES ('{action_type}', '{target_username}', '{status}', '{message}', '{error_message}')"
cursor.execute(sql)
```

Fragmento "después" (parametrizado):
```python
sql = """
INSERT INTO instagram_actions
(action_type, target_username, target_id, status, message, error_message, timestamp)
VALUES (?, ?, ?, ?, ?, ?, ?)
"""
now = datetime.utcnow().isoformat()
params = (action_type, target_username, target_id, status, message, error_message, now)
cursor.execute(sql, params)
```