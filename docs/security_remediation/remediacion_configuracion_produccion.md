# AutoSocial Core: Remediación de Configuración de Producción

## Resumen ejecutivo

La auditoría de AutoSocial Core ha identificado deficiencias críticas en la configuración de producción que impiden el despliegue seguro del sistema. Estas deficiencias incluyen ausencia de variables de entorno de producción, falta de configuración HTTPS, secretos hardcodeados, y configuración incorrecta de CORS. 

Este documento establece las configuraciones necesarias para la producción, las correcciones implementadas, y el plan de validación para garantizar un despliegue seguro.

## Problemas identificados en configuración

### 1. Variables de entorno faltantes

**Problema**: Las siguientes variables críticas no están definidas:
- `DATABASE_URL`: Configuración de base de datos PostgreSQL
- `REDIS_URL`: Configuración del servidor Redis
- `VAULT_TOKEN`: Token para acceso al gestor de secretos
- `JWT_SECRET`: Clave secreta para JWT
- `SSL_KEY_PATH`: Ruta al certificado SSL privado
- `SSL_CERT_PATH`: Ruta al certificado SSL público

**Corrección implementada**:
```javascript
// src/config/secure-config.js
function required(name) {
  const v = process.env[name];
  if (!v) throw new Error(`Variable de entorno requerida: ${name}`);
  return v;
}

module.exports = {
  databaseUrl: required('DATABASE_URL'),
  redisUrl: required('REDIS_URL'),
  jwtSecret: required('JWT_SECRET'),
  vaultToken: required('VAULT_TOKEN'),
  corsAllowedOrigins: (process.env.CORS_ALLOWED_ORIGINS || '').split(',').map(s => s.trim()).filter(Boolean),
  sslKeyPath: required('SSL_KEY_PATH'),
  sslCertPath: required('SSL_CERT_PATH'),
};
```

### 2. Configuración HTTPS ausente

**Problema**: El servidor opera sobre HTTP, exponiendo todo el tráfico.

**Corrección implementada**:
```javascript
// src/server-https.js
const fs = require('fs');
const https = require('https');
const express = require('express');
const securityHeaders = require('./middleware/security-headers');

const app = express();
app.use(securityHeaders);

// Cargar certificados desde entorno seguro
const key = fs.readFileSync(process.env.SSL_KEY_PATH, 'utf-8');
const cert = fs.readFileSync(process.env.SSL_CERT_PATH, 'utf-8');

const httpsServer = https.createServer({ key, cert }, app);
const httpServer = http.createServer((req, res) => {
  res.writeHead(301, { Location: `https://${req.headers.host}${req.url}` });
  res.end();
});

httpsServer.listen(process.env.PORT_HTTPS || 8443, () => {
  console.log(`HTTPS listening on ${httpsServer.address().port}`);
});
httpServer.listen(process.env.PORT_HTTP || 8080, () => {
  console.log(`HTTP redirect on ${httpServer.address().port}`);
});
```

### 3. Configuración de seguridad de encabezados (Headers)

**Problema**: Falta de headers de seguridad como HSTS, X-Frame-Options, etc.

**Corrección implementada**:
```javascript
// src/middleware/security-headers.js
function securityHeaders(req, res, next) {
  res.setHeader("Strict-Transport-Security", "max-age=63072000; includeSubDomains; preload");
  res.setHeader("X-Content-Type-Options", "nosniff");
  res.setHeader("X-Frame-Options", "DENY");
  res.setHeader("X-XSS-Protection", "1; mode=block");
  res.setHeader("Referrer-Policy", "no-referrer");
  res.setHeader("Content-Security-Policy",
    "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; connect-src 'self'; frame-ancestors 'none'; base-uri 'self'; form-action 'self'");
  next();
}
```

## Configuraciones para producción

Para garantizar un despliegue seguro, se deben configurar las siguientes variables de entorno en producción:

### Variables de base de datos
- `DATABASE_URL=postgres://usuario:password@host:puerto/nombre_bd`

### Variables de Redis
- `REDIS_URL=redis://usuario:password@host:puerto`

### Variables de autenticación
- `JWT_SECRET=clave_secreta_aleatoria_larga`
- `CORS_ALLOWED_ORIGINS=https://dominio1.com,https://dominio2.com`

### Variables de SSL
- `SSL_KEY_PATH=/ruta/a/clave-privada.key`
- `SSL_CERT_PATH=/ruta/a/certificado-publico.crt`

### Variables del gestor de secretos
- `VAULT_TOKEN=token_vault`