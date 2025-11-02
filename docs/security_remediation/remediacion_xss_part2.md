## Alcance, fuentes y metodología

El presente análisis cubre el backend HTTP (módulos con handlers de endpoints), el dashboard HTML/JS integrado en el propio servidor, y la exposición de datos a través de respuestas JSON y plantillas embebidas. Se han inspeccionado las rutas afectadas reportadas por la auditoría (WhatsApp, NLP, Instagram, e2e), el manejo de plantillas del dashboard y el uso de datos provenientes de la API en el cliente, con foco en identificar causas de XSS reflected, event-handler y DOM-XSS.

La metodología consistió en:

- Lectura de endpoints que reciben entradas del usuario y las reflejan en respuestas JSON sin escapado.
- Revisión de la gestión del dashboard y su integración con datos de la API, identificando el uso de innerHTML y textContent en contextos susceptibles a manipulación de atributos.
- Verificación de patrones de sanitización ausentes o insuficientes, por contraste con un módulo de referencia que implementa validaciones y escapado (SecurityValidator).
- Mapeo de headers y políticas ausentes (CSP, HSTS, CORS) que incrementan la superficie de ataque.

Para contextualizar la cobertura, se inventarian las áreas analizadas:

Tabla 2. Inventario de componentes y archivos analizados

| Componente            | Endpoints revisados                                | Plantillas/UI                  | Observaciones clave                                                  |
|-----------------------|-----------------------------------------------------|---------------------------------|----------------------------------------------------------------------|
| HTTP server (backend) | /api/whatsapp/test; /api/nlp/classify; /api/instagram/like; /api/instagram/follow; /api/instagram/dm; /api/e2e/whatsapp-flow | Dashboard HTML/JS integrado     | Reflected XSS en 7 endpoints; uso de innerHTML en dashboard; CORS "*" |
| Motores/Integraciones | Instagram, Postiz, NLP, Resiliencia                 | —                               | Datos procesados y reflejados; validación insuficiente en Instagram  |
| Seguridad transversal | Headers, CSP, CORS                                  | —                               | Ausencia de CSP y headers; CORS permisivo                            |

Como referencia de buenas prácticas, se considera el módulo SecurityValidator, que demuestra validación de usernames y media_id, escapado HTML con quote=True, y patrones para detectar atributos y protocolos peligrosos. Este enfoque se utilizará como modelo para reforzar los endpoints vulnerables.

## Hallazgos confirmados en código (7+ instancias XSS)

Los siete endpoints críticos reflejan entradas del usuario sin escape en respuestas JSON. En varios casos, además, el frontend consume estas respuestas y las inserta en el DOM mediante innerHTML, lo que agrava el riesgo de ejecución de script en contextos de atributos, handlers de eventos y rutas de datos.

- Instancia 1 y 2 (POST /api/whatsapp/test). Línea 416. Recibe message y user_id y los devuelve en received_message y user_id sin escapado. Evidencia: response.message_received y response.user_id contienen entradas crudas. Tipo: XSS reflected.
- Instancia 3 (POST /api/nlp/classify). Línea 448. Devuelve original_message sin escape. Tipo: XSS reflected.
- Instancia 4 (POST /api/instagram/like). Línea 483. Devuelve username sin escape. Tipo: XSS reflected. Potencial event-handler si el frontend lo inserta en atributos HTML.
- Instancia 5 (POST /api/instagram/follow). Línea 509. Devuelve username sin escape. Tipo: XSS reflected.
- Instancia 6 (POST /api/instagram/dm). Línea 532. Devuelve username y message sin escape. Tipo: XSS reflected. Amplia superficie por longitud de texto libre.
- Instancia 7 (POST /api/e2e/whatsapp-flow). Línea 688. Devuelve original_message sin escape. Tipo: XSS reflected.

El dashboard usa innerHTML para mostrar datos de API (incluidas las respuestas de estos endpoints), lo que habilita vectores DOM-XSS al insertar contenido no confiable en el DOM con manipulación de atributos. Si bien el uso de textContent en algunos lugares es más seguro, la combinación de innerHTML y datos no escapados eleva el riesgo de event handlers maliciosos (onerror, onclick, onload).