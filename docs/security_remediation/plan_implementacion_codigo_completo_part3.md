## 4. Bloqueador SEC-001: Inyección SQL (Crítica)

### Evidencia y alcance

La auditoría confirma 16 instancias de SQL Injection en endpoints críticos:
- GET /api/instagram/user/{username}
- GET /api/postiz/next-slot/{channel_id}
- POST /api/whatsapp/test
- POST /api/nlp/classify
- POST /api/e2e/whatsapp-flow

La explotación permite extracción, modificación y eliminación de datos, y acceso a información sensible.

### Código a implementar

1) Capa de acceso a datos con SQLite (Node.js + sqlite3)
```js
// src/db/sqlite-connector.js
const sqlite3 = require('sqlite3').verbose();

function openDb() {
  return new sqlite3.Database(process.env.SQLITE_DB_PATH || './data/autosocial.db');
}

function all(db, sql, params = []) {
  return new Promise((resolve, reject) => {
    db.all(sql, params, (err, rows) => (err ? reject(err) : resolve(rows)));
  });
}

function get(db, sql, params = []) {
  return new Promise((resolve, reject) => {
    db.get(sql, params, (err, row) => (err ? reject(err) : resolve(row)));
  });
}

function run(db, sql, params = []) {
  return new Promise((resolve, reject) => {
    db.run(sql, params, function (err) {
      if (err) return reject(err);
      resolve({ lastID: this.lastID, changes: this.changes });
    });
  });
}

module.exports = { openDb, all, get, run };
```

2) DAO seguro para interactions por userId (sin concatenación)
```js
// src/dao/interactions-dao.js
const { all, get, run, openDb } = require('../db/sqlite-connector');

async function listInteractionsByUserId(userId) {
  const db = openDb();
  const sql = 'SELECT id, user_id, target_id, created_at FROM interactions WHERE user_id = ?';
  const rows = await all(db, sql, [userId]);
  db.close();
  return rows;
}

async function createInteraction({ userId, targetId }) {
  const db = openDb();
  const sql = 'INSERT INTO interactions (user_id, target_id, created_at) VALUES (?, ?, ?)';
  const result = await run(db, sql, [userId, targetId, new Date().toISOString()]);
  db.close();
  return { id: result.lastID };
}

module.exports = { listInteractionsByUserId, createInteraction };
```

3) Validación de parámetros (schema mínimo)
```js
// src/validators/interaction-validator.js
function isNonEmptyString(v) {
  return typeof v === 'string' && v.trim().length > 0;
}
function isPositiveInt(v) {
  return Number.isInteger(v) && v > 0;
}

module.exports = { isNonEmptyString, isPositiveInt };
```