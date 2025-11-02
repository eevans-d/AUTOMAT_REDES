Tabla 7. Checklist de headers de seguridad y estado

| Header                      | Estado     | Valor propuesto                                     | Razón                                                                 |
|----------------------------|------------|-----------------------------------------------------|------------------------------------------------------------------------|
| Strict-Transport-Security  | Ausente    | max-age=31536000; includeSubDomains; preload        | Forzar HTTPS y prevenir downgrades                                    |
| X-Content-Type-Options     | Ausente    | nosniff                                             | Evitar MIME sniffing                                                   |
| X-Frame-Options            | Ausente    | DENY o SAMEORIGIN                                   | Prevenir clickjacking                                                  |
| Content-Security-Policy    | Ausente    | script-src 'self'; object-src 'none'; base-uri 'none'; style-src 'self' | Bloquear inline scripts y fuentes no confiables                        |
| Referrer-Policy            | Ausente    | no-referrer o strict-origin-when-cross-origin       | Minimizar fuga de información en referrers                             |
| Access-Control-Allow-Origin| Permisivo  | Lista explícita de orígenes                         | Evitar abuso cross-origin                                              |
| Access-Control-Allow-Cred. | A revisar  | Deshabilitado salvo necesidad explícita             | Reducir riesgo de abuso con credenciales                               |

## Plan de remediación priorizado y cronograma

Se propone una ejecución en tres fases, alineada con el timeline de la auditoría, para alcanzar un GO seguro.[^1]

- Fase 1 (0–1 semana): mitigación crítica
  - Implementar escape/encoding por contexto en los siete endpoints confirmados.
  - Validación whitelisting para usernames en endpoints Instagram.
  - Introducir CSP restrictiva, configurar headers de seguridad y endurecer CORS.
  - Objetivo: superar checks no negociables (XSS, CORS, headers).

- Fase 2 (1–3 semanas): preparación para producción
  - Autenticación (JWT) en endpoints sensibles, rate limiting.
  - Validación de entradas sistemática (tipos, longitudes, whitelists).
  - Hardening de plantillas: eliminar innerHTML con datos no confiables.
  - Objetivo: robustecer defensas y reducir explotación de errores.

- Fase 3 (3–4 semanas): madurez operativa
  - Pipeline de CI/CD con escaneo de seguridad, alertas, backups, rollback, pruebas de carga.
  - Objetivo: operar de forma segura y mantenible.

Tabla 8. Cronograma y responsables

| Fase | Tareas clave                                        | Responsable            | Duración estimada | Entregables                                  |
|------|------------------------------------------------------|------------------------|-------------------|----------------------------------------------|
| 1    | Escape en 7 endpoints, CSP, headers, CORS           | Backend + Seguridad    | 1 semana          | PRs合并; headers activos; CSP aplicada       |
| 2    | Auth JWT, rate limiting, validación sistemática     | Backend + QA           | 2 semanas         | Endpoints protegidos; suite de validación    |
| 3    | CI/CD seguro, alertas, backups, rollback            | DevOps + Seguridad     | 1 semana          | Pipeline activo; plan de rollback probado    |

## Plan de pruebas y re-testing

El re-testing debe ejecutar payloads XSS estándar contra cada endpoint corregido y validar que el frontend no interprete contenido malicioso, apoyado en headers y CSP. La cobertura debe incluir casos de reflected, event-handler y DOM-XSS, con revisión de los encabezados y políticas.

Tabla 9. Matriz de casos de prueba

| Endpoint                    | Payload                            | Método de inyección | Resultado esperado                                     | Resultado real | Observaciones                      |
|----------------------------|------------------------------------|---------------------|--------------------------------------------------------|----------------|------------------------------------|
| /api/whatsapp/test         | <script>alert('XSS')</script>      | JSON body           | Payload no ejecutado; texto escapado                   | —              | Validar escape + CSP               |
| /api/whatsapp/test         | "><img src=x onerror=alert('XSS')> | JSON body           | Atributos no ejecutados; texto escapado                | —              | Verificar encoding de comillas     |
| /api/nlp/classify          | <svg onload=alert('XSS')>          | JSON body           | Sin ejecución; escapado                                | —              | Probar ensure_ascii + escape       |
| /api/instagram/like        | javascript:alert('XSS')            | JSON body           | No ejecutado; validación username y escape             | —              | Confirmar regex y longitud         |
| /api/instagram/follow      | " onfocus=alert('XSS') x="         | JSON body           | No ejecutado; atributos escapados                      | —              | Verificar construcción DOM         |
| /api/instagram/dm          | <img src=x onerror=alert('XSS')>   | JSON body           | No ejecutado; texto y atributos escapados              | —              | Truncamiento seguro                |
| /api/e2e/whatsapp-flow     | <script>alert('XSS')</script>      | JSON body           | No ejecutado; escape de original_message               | —              | Validar integración con frontend   |
| Dashboard                  | onclick="..." (manipulación)       | innerHTML           | Bloqueado por CSP y sin interpretación de handlers     | —              | Revisar uso de textContent         |

Se recomienda integrar estas pruebas en CI/CD para garantizar regresión segura en cada despliegue.

## Riesgos, impacto y verificación post-remediación

El riesgo principal es la ejecución de JavaScript en navegadores de usuarios, con capacidades para robo de sesiones y cookies, modificación de contenido y redirección a sitios maliciosos. Este impacto se amplifica por la combinación de CORS permisivo y ausencia de CSP y otros headers, lo que facilita ataques cross-origin y el uso de inline scripts y event handlers.[^2]

Las métricas de cierre deben evidenciar:

- 0 payloads XSS ejecutables en los siete endpoints.
- CSP y headers activos conforme al checklist; CORS endurecido con orígenes explícitos.
- Suite de pruebas automatizadas en CI/CD que bloquee regresiones.

El monitoreo continuo debe registrar intentos de inyección, patrones maliciosos y violaciones de CSP, con alertas al equipo de seguridad.

---

## Referencias

[^1]: AUDITORIA_TECNICA_CRITICA_AUTOSOCIAL_FINAL.md.
[^2]: REPORTE_SEGURIDAD_OWASP_COMPLETO.md.