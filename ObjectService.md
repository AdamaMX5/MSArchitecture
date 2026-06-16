# ObjectService

> Base URL: `https://object.freischule.info`

Generischer JSON-Objektspeicher. Jeder Microservice oder jede App kann beliebige JSON-Dokumente ohne festes Schema speichern, abfragen und verknüpfen.

**Datenmodell** — jedes gespeicherte Dokument hat zwei Payload-Felder:

| Feld | Zweck | Index |
|------|-------|-------|
| `data` | Beliebiges JSON-Payload | On-demand via Admin |
| `refs` | Foreign-Key-Map (z.B. `{ carId, userId }`) | Wildcard-Index — alle Keys immer indiziert |

Weitere Metadaten: `collectionName`, `isPublic`, `app`, `tags`, `createdBy`, `updatedBy`, `createdAt`, `updatedAt`.

**Auth-Varianten:**
- **API-Key** — `X-API-Key` Header; für interne Microservice-Aufrufe
- **JWT** — Bearer Token; für Nutzer-Aufrufe
- **JWT Admin** — gültiges JWT mit Rolle `admin`

---

## HTTP — Public

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `GET` | `/` | — | Hello World |
| `GET` | `/health` | — | Health check — `{ status, service, timestamp }` |

---

## HTTP — Object REST API (`/objects`)

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `GET` | `/objects/:collection` | optional | Objekte auflisten — unauthentifizierte Aufrufer sehen nur `isPublic: true`-Dokumente |
| `GET` | `/objects/:collection/:id` | optional | Einzelnes Objekt — `403` wenn privat und unauthentifiziert |
| `POST` | `/objects/:collection` | JWT / API-Key | Anlegen — Body: `{ data*, refs?, isPublic?, app?, tags? }` → `201` |
| `PUT` | `/objects/:collection/:id` | JWT / API-Key | Vollständige Ersetzung |
| `PATCH` | `/objects/:collection/:id` | JWT / API-Key | Partielles Update — Body: `{ data?, refs?, merge? }` (`merge: true` shallow-merged, `false` ersetzt) |
| `DELETE` | `/objects/:collection/:id` | JWT / API-Key | Löschen — `204` bei Erfolg |

### Query-Parameter für `GET /objects/:collection`

| Param | Beispiel | Description |
|-------|---------|-------------|
| `page` / `limit` | `?page=2&limit=20` | Pagination (max limit: 100) |
| `isPublic` | `?isPublic=true` | Nach Sichtbarkeit filtern |
| `app` | `?app=VirtualOffice` | Nach App-Name filtern |
| `tags` | `?tags=sport,sedan` | Kommagetrennte Tag-Filter (`$all`) |
| `ref[key]` | `?ref[carId]=abc123` | Foreign-Key-Filter — trifft den Wildcard-Index |

Mehrere `ref`-Keys werden AND-verknüpft: `?ref[carId]=abc&ref[userId]=xyz`

---

## HTTP — Admin (`/admin`)

> Alle Endpunkte erfordern JWT mit Rolle `admin`.

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/admin/collections` | Alle Collection-Namen auflisten |
| `DELETE` | `/admin/collections/:collection` | Alle Objekte einer Collection löschen |
| `GET` | `/admin/indexes` | Alle MongoDB-Indexes auflisten |
| `POST` | `/admin/indexes` | Compound-Index auf `data`-Feld anlegen — Body: `{ field*, unique? }` — Dot-Notation unterstützt (`"address.city"`) |
| `DELETE` | `/admin/indexes/:name` | Index nach Name löschen |
| `POST` | `/admin/refresh-key` | Gecachten JWT Public Key force-refreshen |

---

## GraphQL (`/graphql`)

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `POST` | `/graphql` | optional / JWT | Apollo Server 4 GraphQL-Endpunkt |

### Queries

| Operation | Auth | Arguments | Description |
|-----------|------|-----------|-------------|
| `getObject` | optional | `collection: String!, id: ID!` | Einzelnes Objekt abrufen |
| `listObjects` | optional | `collection, page, limit, isPublic, app, tags, refs: JSON` | Auflisten mit Filtern |
| `searchObjects` | optional | `collection, query: JSON!, page, limit` | Gleichheitssuche auf `data`-Feldern — effizient nur mit Index |
| `listCollections` | JWT | — | Alle Collection-Namen |
| `listIndexes` | JWT Admin | — | Alle MongoDB-Indexes |

### Mutations

| Operation | Auth | Input | Description |
|-----------|------|-------|-------------|
| `createObject` | JWT | `collection, data, refs?, isPublic?, app?, tags?` | Anlegen |
| `updateObject` | JWT | `id, data?, refs?, isPublic?, app?, tags?` | Vollständige Ersetzung |
| `patchObject` | JWT | `id, data?, refs?, merge?` | Partielles Update |
| `deleteObject` | JWT | `id` | Löschen |
| `createDataIndex` | JWT Admin | `field: String!, unique?: Boolean` | Compound-Index `{ collectionName, data.<field> }` anlegen |
| `dropIndex` | JWT Admin | `name: String!` | Index nach Name löschen |

---

## Index-Strategie

```
refs.carId = "abc"   → Wildcard-Index { "refs.$**": 1 }            ← immer schnell
data.brand = "BMW"   → Admin-Index { collectionName, "data.brand" } ← schnell nach Anlage
data.year = 2023     → kein Index                                   ← Collection-Scan
```
