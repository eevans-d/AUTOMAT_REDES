## 2. Resumen ejecutivo de bloqueadores y decisión NO-GO

AutoSocial Core presenta ocho bloqueadores críticos que impiden su despliegue. La tabla siguiente sintetiza severidad e impacto y sustenta el plan de remediación y validación.

Tabla 1. Bloqueadores críticos (ID, severidad, impacto, área afectada)

| ID      | Descripción                                             | Severidad | Impacto Potencial                                        | Área Afectada                  |
|---------|----------------------------------------------------------|-----------|-----------------------------------------------------------|--------------------------------|
| SEC-001 | Inyección SQL                                           | Crítica   | Compromiso total de la base de datos, extracción/modificación/eliminación de datos | Backend, Base de Datos, API    |
| SEC-002 | Cross-Site Scripting (XSS)                              | Crítica   | Ejecución de scripts en navegadores, robo de sesiones/cookies, desfiguración | Frontend, API                  |
| SEC-003 | Credenciales hardcodeadas y expuestas                   | Crítica   | Acceso no autorizado a servicios de terceros              | Configuración, Integraciones   |
| SEC-004 | Sin cifrado de tráfico (HTTP en producción)             | Crítica   | Interceptación de credenciales y datos en tránsito        | Infraestructura, Red           |
| CONF-001| Ausencia de configuración de producción                 | Crítica   | Imposibilidad de conectar DB y gestor de secretos         | Configuración, Despliegue      |
| CONF-002| Falta de Feature Flags y Kill Switches                  | Crítica   | Incapacidad para desactivar funcionalidades sin redespliegue | Operaciones, Arquitectura      |
| SEC-005 | CORS permisivo ("*")                                    | Crítica   | Peticiones cross-site no autorizadas y CSRF               | Seguridad, API                 |
| SEC-006 | Falta de sistema de autenticación                       | Crítica   | Endpoints sensibles sin verificación de identidad         | Seguridad, Acceso              |

La ejecución de este plan tiene como prerrequisito: detener despliegues mientras se implementan los controles, habilitar acceso a gestor de secretos y definir staging con datos sintéticos. Sin estos prerrequisitos, las validaciones y la seguridad del despliegue quedarían comprometidas.

## 3. Plan maestro de implementación (Timeline y gobernanza)

El plan se ejecuta en tres fases con objetivos y entregables claros, y gobernanza transversal para garantizar seguridad y trazabilidad.

Tabla 2. Cronograma por fase: objetivo, duración, entregables, responsables, dependencias

| Fase  | Duración         | Objetivo                                         | Entregables clave                                                                                                   | Responsables                  | Dependencias                               |
|-------|------------------|--------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|-------------------------------|--------------------------------------------|
| 1     | 0–1 semana       | Mitigación crítica                               | SQLi parcheado; XSS neutralizado; CORS restrictivo; HTTPS habilitado; headers seguridad; secretos fuera del código | Backend, Frontend, DevOps     | Acceso Vault; orígenes CORS; certificados  |
| 2     | 1–3 semanas      | Preparación para producción                      | Autenticación/autorización JWT; DB PostgreSQL; rate limiting; validación de entradas; configuración producción     | Backend, DevOps, QA           | Decisión DB; políticas JWT; límites rate   |
| 3     | 3–4 semanas      | Madurez operativa                                | CI/CD con escaneo ZAP; alertas y monitoreo; backups y rollback; pruebas de carga                                   | DevOps, QA, Seguridad         | Pipeline CI/CD; stack de monitoreo         |

Gobernanza del cambio:
- Bitácora de cambios con aprobaciones por el responsable de Seguridad.
- Pruebas unitarias, integración y seguridad obligatorias antes de promoción a staging y producción.
- Criterios de salida: evidencia objetiva de mitigación por bloqueador y validación cruzada por QA.

### 3.1 Fase 1 (0–1 semana): Mitigación crítica
Se implementan prepared statements en endpoints afectados, salida segura en frontend con CSP, CORS con lista de orígenes, HTTPS con HSTS, headers de seguridad y extracción de secretos a variables de entorno + Vault. Objetivo: superar checks no negociables.

### 3.2 Fase 2 (1–3 semanas): Preparación para producción
Se introduce autenticación y autorización JWT con scopes, rate limiting, validación de entradas, migración a PostgreSQL y configuración de variables de entorno de producción. Objetivo: robustez y readiness técnico.

### 3.3 Fase 3 (3–4 semanas): Madurez operativa
Se establece CI/CD con escaneo continuo (ZAP), alertas, respaldos, plan de rollback y pruebas de carga. Objetivo: operación segura y mantenible.