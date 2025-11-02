# Plan de Implementación de Código para Resolver 8 Bloqueadores Críticos — AutoSocial Core

## 1. Propósito, alcance y método

Este plan técnico y operativo tiene como propósito levantar la decisión de NO-GO vigente sobre AutoSocial Core mediante la corrección de ocho bloqueadores críticos detectados en la auditoría. El entregable principal es el documento técnico con código, rutas de archivos, pruebas y cronograma: docs/security_remediation/plan_implementacion_codigo_completo.md.

El alcance abarca correcciones en backend (Node.js/Express), base de datos (SQLite durante remediación, migración a PostgreSQL en Fase 2), frontend (salidas HTML), infraestructura (HTTPS y CORS) y operación (gestión de secretos y configuración). Se siguen tres principios metodológicos: defensa en profundidad, mínimo privilegio y fail-safe. La ejecución se organiza en tres fases alineadas al veredicto y timeline del reporte consolidado, priorizando la mitigación inmediata y la validación exhaustiva antes del cambio de estado a GO.

### 1.1 Narrativa del problema

La auditoría documenta vulnerabilidades críticas que, de forma independiente, justifican el NO-GO: inyección SQL (16 instancias en endpoints clave), XSS reflected/event handler (60+ instancias), CORS permisivo con comodín, ausencia de HTTPS, secretos expuestos en configuración y falta de autenticación y autorización robustas. Estas condiciones exponen a compromiso total de datos, ataques cross-origin y robo de credenciales, con un riesgo inaceptable para producción.

### 1.2 Criterios de éxito

El sistema superará los checks no negociables cuando:
- SQL Injection eliminado (prepared statements en todos los endpoints).
- XSS neutralizado (CSP restrictiva + encoding/escape de salidas).
- CORS restringido a orígenes explícitos; sin comodín; headers y credenciales según política mínima.
- HTTPS habilitado con HSTS, redirección y headers de seguridad; certificates gestionados vía gestor de secretos.
- Secretos removidos del código y gestionados vía variables de entorno + Vault; rotación y acceso mínimo.
- Autenticación y autorización por JWT implementadas, con rate limiting y validación de scopes por endpoint.
- Configuración de producción definidas (DATABASE_URL, REDIS_URL, VAULT_TOKEN, JWT_SECRET) y Flags/Kill Switch operativos.
- Pruebas de seguridad integradas en CI/CD; escaneo continuo (ZAP); rollback y DR operativos; monitoreo y alertas activos.

Brechas de información a resolver en paralelo: lista exacta de rutas por lenguaje y stack, inventario completo de endpoints con consultas SQL y lugares donde se renderiza HTML/JS, catálogo de orígenes permitidos por CORS, decisión de base de datos en producción, política de dominios y scopes JWT, especificación de certificados, criterios de rate limiting por rol y endpoint, plan de secretos ( Vault/KeyMgmt ), infraestructura de CI/CD y entorno de staging y catálogo de flags y kill switches. Estas brechas no impiden iniciar la Fase 1, pero condicionan detalles de configuración fina y despliegue seguro.