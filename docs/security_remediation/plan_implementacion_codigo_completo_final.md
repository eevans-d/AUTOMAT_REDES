### Testing requerido

- Pruebas unitarias de validadores.
- Pruebas de integración por endpoint con payloads de inyección; deben retornar 400/404 y no ejecutar SQL malicioso.
- Suite OWASP/ZAP para SQLi; evidencia de 0 hallazgos críticos tras la corrección.
- Re-ejecutar security_tests.py sobre los endpoints afectados; comparación antes/después.

### Timeline de implementación
- Día 1–2: Refactor de endpoints críticos, aplicación de prepared statements y validadores.
- Día 3: Pruebas unitarias e integración; ajustes; evidencia de mitigación.

## 5. Bloqueador SEC-002: XSS (Crítica)

### Evidencia y alcance

Se detectan 60+ instancias de XSS reflected y event handler en endpoints como POST /api/whatsapp/test, /api/nlp/classify, /api/instagram/like, /api/instagram/follow, /api/instagram/dm y /api/e2e/whatsapp-flow. Sin escape de salida, un atacante puede ejecutar JavaScript en el navegador del usuario y robar sesiones.

### Código a implementar (Backend: sanitización y CSP)

1) Middleware de sanitización de entradas (remueve tags y atributos peligrosos)
```js
// src/middleware/xss-sanitizer.js
function escapeHTML(str) {
  return String(str)
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#39;');
}

function sanitizeInput(obj) {
  if (typeof obj === 'string') return escapeHTML(obj);
  if (Array.isArray(obj)) return obj.map(sanitizeInput);
  if (obj && typeof obj === 'object') {
    const out = {};
    for (const [k, v] of Object.entries(obj)) out[k] = sanitizeInput(v);
    return out;
  }
  return obj;
}

function xssSanitizer(req, res, next) {
  // Aplicar sanitización solo a campos que se reflejarán en respuestas
  req.body = sanitizeInput(req.body || {});
  req.query = sanitizeInput(req.query || {});
  next();
}

module.exports = { xssSanitizer, escapeHTML };
```

## Conclusión

La ejecución disciplinada de este plan corrige de manera verificable los ocho bloqueadores críticos, reduce el riesgo a niveles aceptables y habilita la preparación operativa para producción. La defensa en profundidad propuesta—desde la validación de entradas y salida segura hasta el cifrado de transporte, gestión de secretos, autenticación/autorización y flags—eleva el cumplimiento técnico y la postura de seguridad de AutoSocial Core. El éxito depende de cerrar brechas, cumplir exit criteria y sostener evidencia objetiva en CI/CD. La meta es clara: pasar de NO-GO a GO en cuatro semanas, con seguridad y gobernanza integradas desde el diseño hasta la operación.