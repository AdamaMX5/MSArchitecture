# ObjectService

> Base URL: `https://object.freischule.info`

Generischer JSON-Objektspeicher. Jeder Microservice oder jede App kann beliebige JSON-Dokumente ohne festes Schema speichern, abfragen und verknüpfen.

**Datenmodell** — jedes gespeicherte Dokument hat zwei Payload-Felder:

| Feld | Zweck | Index |
|------|-------|-------|
| `data` | Beliebiges JSON-Payload | On-demand via Admin |
| `refs` | Foreign-Key-Map (z.B. `{ carId, userId }`) | Wildcard-Index — alle Keys immer indiziert |

Weitere Metadaten: `collectionName`, `isPublic`, `app`, `tags`, `createdBy`, `updatedBy`, `createdAt`, `updatedAt`, `deletedAt`, `deletedBy`.

> `deletedAt`/`deletedBy` steuern das **Soft-Delete** (s.u.): `deletedAt: null` (oder Feld fehlt, bei Altdaten) = aktiv; ein Timestamp = soft-gelöscht.

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
| `DELETE` | `/objects/:collection/:id` | JWT / API-Key | **Soft-Delete** — setzt `deletedAt` statt zu entfernen — `204` bei Erfolg |
| `POST` | `/objects/:collection/:id/restore` | JWT / API-Key | Wiederherstellen (Undelete) — dieselbe `editRoles`-ACL wie `DELETE` → gibt das Objekt zurück |
| `DELETE` | `/objects/:collection/:id/purge` | JWT Admin / API-Key | **Hartes Löschen** (endgültig, irreversibel) — nur Admin/API-Key — `204` bei Erfolg |

### Query-Parameter für `GET /objects/:collection`

| Param | Beispiel | Description |
|-------|---------|-------------|
| `page` / `limit` | `?page=2&limit=20` | Pagination (max limit: 100) |
| `isPublic` | `?isPublic=true` | Nach Sichtbarkeit filtern |
| `app` | `?app=VirtualOffice` | Nach App-Name filtern |
| `tags` | `?tags=sport,sedan` | Kommagetrennte Tag-Filter (`$all`) |
| `ref[key]` | `?ref[carId]=abc123` | Foreign-Key-Filter — trifft den Wildcard-Index |
| `includeDeleted` | `?includeDeleted=true` | Soft-gelöschte Objekte mit ausliefern — **nur** für Admin/API-Key wirksam, sonst ignoriert |

Mehrere `ref`-Keys werden AND-verknüpft: `?ref[carId]=abc&ref[userId]=xyz`

> `includeDeleted=true` wirkt auch auf `GET /objects/:collection/:id` (Einzelabruf) — ebenfalls nur für Admin/API-Key.

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
| `POST` | `/admin/classes` | Klasse anlegen/aktualisieren (Upsert auf `namespace`) — Body: `{ namespace*, readRoles?, writeRoles?, editRoles?, membershipField?, app? }` → `201` |
| `GET` | `/admin/classes` | Alle registrierten Klassen auflisten |
| `GET` | `/admin/classes/:namespace` | Einzelne Klasse — `404` wenn nicht registriert |
| `DELETE` | `/admin/classes/:namespace` | Klasse entfernen (→ Namespace wieder ungeprüft) |

---

## Klassen / Namespace-ACL (rollenbasierte Zugriffskontrolle)

Optionale, **server-seitige** Zugriffskontrolle pro Collection (= Namespace). Eine
Collection wird als **Klasse** registriert (gespeichert in der reservierten
Collection `_classes`, nur per Admin verwaltbar) und legt fest, welche Rolle ein
JWT-Nutzer für `read` / `write` / `edit` haben muss.

**Klassendefinition:**

```jsonc
{
  "namespace": "channels",                     // == collectionName
  "readRoles":  ["employee", "admin"],         // GET (list/get)
  "writeRoles": ["admin", "channel-admin"],    // POST (create)
  "editRoles":  ["admin", "channel-admin"],    // PUT / PATCH / DELETE
  "membershipField": "memberIds",              // optional — Member-Level-ACL (s.u.)
  "app": "MessangerClient"                      // optional
}
```

**Enforcement** bei jedem `/objects/:collection`-Zugriff (REST **und** GraphQL):

| Bedingung | Verhalten |
|-----------|-----------|
| Keine Klasse für `:collection` registriert | Kein Rollen-Check (rückwärtskompatibel — Migration) |
| Klasse vorhanden, Rollenliste **leer** | Keine zusätzliche Einschränkung über die Basis-Auth der Operation hinaus |
| Klasse vorhanden, Rollenliste **nicht leer** | Aufrufer muss authentifiziert sein **und** eine passende Rolle besitzen, sonst `403` (unauthentifiziert → `401`) |
| Aufruf via `X-API-Key` (interner Service) | **Nie** gegated — vertrauenswürdig |

> Operation-Mapping: `GET` → `readRoles`, `POST` → `writeRoles`, `PUT`/`PATCH`/`DELETE` → `editRoles`.
> `isPublic` bleibt erhalten: ohne registrierte (nicht-leere) `readRoles` gilt die bisherige `isPublic`-Logik unverändert.
> Klassendefinitionen werden in-memory gecacht (TTL `CLASS_CACHE_TTL_MS`, Default 30s). Mutationen invalidieren den Cache sofort; zusätzlich leert ein MongoDB-Change-Stream auf `_classes` den Cache **aller Instanzen** sofort bei jeder Änderung (schließt das Fail-Open-Fenster bei ACL-Verschärfung über mehrere Replicas). Bei transienten Stream-Fehlern (Failover/Netz) reconnectet der Stream mit Backoff; auf Standalone-MongoDB (Change-Streams nicht unterstützt) Fallback auf TTL-begrenzte Staleness. Der per-Eintrag-TTL bleibt unabhängig vom Stream-Status der ultimative Backstop. Der Stream wird bei `SIGTERM`/`SIGINT` sauber geschlossen.

**Migrationsstrategie:** Phase 1 — Deploy ohne Klassen → nichts ändert sich. Phase 2 —
sensible Namespaces (`spaces`, `channels`, …) schrittweise registrieren → ACL wird
nur für diese erzwungen. Kein Big-Bang, kein Zwischenlayer.

---

## Member-Level-ACL (objekt-genaue Mitgliedschaft)

Optionaler, **pro-Objekt**-Layer **zusätzlich** zur Rollen-ACL. Setzt eine Klasse das
Feld `membershipField`, so wird es als **Dot-Pfad innerhalb von `data`** interpretiert,
der auf ein **Array von User-IDs** (die JWT-`sub`-Werte) zeigt — z.B.
`"memberIds"` → geprüft gegen `data.memberIds`, `"acl.members"` → `data.acl.members`.
Fehlt/leer → reine Rollen-ACL (rückwärtskompatibel).

**Enforcement** (greift bei `read` Einzelobjekt und `edit`; **nicht** bei `write`/create):

| Bedingung | Verhalten |
|-----------|-----------|
| `membershipField` nicht gesetzt | Keine Mitglieds-Prüfung (nur Rollen-ACL) |
| Aufruf via `X-API-Key` | **Nie** gegated — vertrauenswürdig |
| Aufrufer hat Rolle `admin` | **Bypass** — darf moderieren (Objekt ändern, Member hinzufügen/entfernen); **kein** API-Key nötig |
| `read` auf `isPublic: true`-Objekt | Erlaubt — öffentliche Objekte bleiben lesbar |
| sonst: `sub ∈ data.<membershipField>` | Erlaubt, andernfalls `403` (unauthentifiziert → `401`) |

> Reihenfolge: erst Rollen-Check (`readRoles`/`editRoles`), dann Member-Check — **beide** müssen passen.
> **Edit** eines `isPublic`-Objekts erfordert weiterhin Mitgliedschaft/Admin (nur **Read** ist öffentlich).
> **Create** wird nicht über Mitgliedschaft gegated und der Ersteller wird **nicht** automatisch
> in die Member-Liste aufgenommen — der Client steuert `data` (inkl. der Member-Liste) selbst.

**List/Search:** Für authentifizierte Nicht-Admins werden die Ergebnisse server-seitig
gefiltert auf Objekte mit `data.<membershipField> = sub` **oder** `isPublic: true`.
Admin/API-Key → ungefiltert. Unauthentifiziert → nur `isPublic`.

> **Index-Empfehlung:** Für member-gegatete Collections einen Data-Index auf das
> Member-Feld anlegen (`POST /admin/indexes { "field": "memberIds" }`), sonst läuft
> die List-Filterung als Collection-Scan.

---

## Soft-Delete (wiederherstellbares Löschen)

`DELETE /objects/:collection/:id` **entfernt das Dokument nicht mehr**, sondern setzt
`deletedAt` (Timestamp) und `deletedBy` (JWT-`sub`). So lassen sich versehentliche
oder böswillige Löschungen rückgängig machen (Issue #10). Wer löschen darf, richtet
sich unverändert nach der Klassen-ACL (`editRoles`) plus ggf. Member-Level-ACL.

| Aspekt | Verhalten |
|--------|-----------|
| `DELETE` | Setzt `deletedAt`/`deletedBy`; bereits gelöscht → `404`. `204` bei Erfolg. |
| Reads (Default) | REST **und** GraphQL, Listen **und** Einzel-/Search-Queries liefern soft-gelöschte Objekte **nicht** aus. |
| `includeDeleted` | Opt-in, um gelöschte Objekte mitzuliefern — REST `?includeDeleted=true`, GraphQL-Argument `includeDeleted`. Wird **nur** für Admin (JWT) bzw. API-Key beachtet, sonst ignoriert. |
| Restore | `POST /objects/:collection/:id/restore` bzw. `restoreObject(id)` — hebt das Flag auf; dieselbe `editRoles`-ACL wie `DELETE`. Nur ein tatsächlich gelöschtes Objekt → sonst `404`. |
| Purge (Hard-Delete) | `DELETE /objects/:collection/:id/purge` bzw. `purgeObject(id)` — endgültiges Entfernen, **nur Admin/API-Key**. Irreversibel. |

**Rückwärtskompatibel:**
- Altdaten **ohne** `deletedAt`-Feld gelten als **nicht gelöscht** (der Filter
  `{ deletedAt: null }` matcht auch fehlende Felder).
- Der FreeSchool-Client filterte bisher `data.deleted` selbst; sobald er auf das
  native `DELETE` umstellt, wird der Client-Filter zum harmlosen No-Op. (Das
  Soft-Delete nutzt das eigene Top-Level-Feld `deletedAt`, nicht `data.deleted`.)

> **Index-Empfehlung:** Der statische Index `{ collectionName: 1, deletedAt: 1 }`
> ist eingebaut und deckt den Default-„nur aktive"-Filter ab.

---

## GraphQL (`/graphql`)

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `POST` | `/graphql` | optional / JWT | Apollo Server 4 GraphQL-Endpunkt |

### Queries

| Operation | Auth | Arguments | Description |
|-----------|------|-----------|-------------|
| `getObject` | optional | `collection: String!, id: ID!, includeDeleted?` | Einzelnes Objekt abrufen (`includeDeleted` nur für Admin wirksam) |
| `listObjects` | optional | `collection, page, limit, isPublic, app, tags, refs: JSON, includeDeleted?` | Auflisten mit Filtern (`includeDeleted` nur für Admin wirksam) |
| `searchObjects` | optional | `collection, query: JSON!, page, limit, includeDeleted?` | Gleichheitssuche auf `data`-Feldern — effizient nur mit Index (`includeDeleted` nur für Admin wirksam) |
| `listCollections` | JWT | — | Alle Collection-Namen |
| `listIndexes` | JWT Admin | — | Alle MongoDB-Indexes |
| `listClasses` | JWT Admin | — | Alle registrierten Klassen-ACLs |
| `getClass` | JWT Admin | `namespace: String!` | Einzelne Klassendefinition |

### Mutations

| Operation | Auth | Input | Description |
|-----------|------|-------|-------------|
| `createObject` | JWT | `collection, data, refs?, isPublic?, app?, tags?` | Anlegen |
| `updateObject` | JWT | `id, data?, refs?, isPublic?, app?, tags?` | Vollständige Ersetzung |
| `patchObject` | JWT | `id, data?, refs?, merge?` | Partielles Update |
| `deleteObject` | JWT | `id` | **Soft-Delete** — setzt `deletedAt` (wiederherstellbar) → `Boolean` |
| `restoreObject` | JWT | `id` | Soft-gelöschtes Objekt wiederherstellen (dieselbe `editRoles`-ACL) → gibt Objekt zurück |
| `purgeObject` | JWT Admin | `id` | **Hartes Löschen** (endgültig) → `Boolean` |
| `createDataIndex` | JWT Admin | `field: String!, unique?: Boolean` | Compound-Index `{ collectionName, data.<field> }` anlegen |
| `dropIndex` | JWT Admin | `name: String!` | Index nach Name löschen |
| `setClass` | JWT Admin | `namespace: String!, readRoles?, writeRoles?, editRoles?, membershipField?, app?` | Klasse anlegen/aktualisieren (Upsert) |
| `deleteClass` | JWT Admin | `namespace: String!` | Klasse entfernen |

---

## Index-Strategie

```
refs.carId = "abc"   → Wildcard-Index { "refs.$**": 1 }            ← immer schnell
data.brand = "BMW"   → Admin-Index { collectionName, "data.brand" } ← schnell nach Anlage
data.year = 2023     → kein Index                                   ← Collection-Scan
```
