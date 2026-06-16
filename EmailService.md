# EmailService

> Base URL: `https://email.freischule.info`

E-Mail-Versand (Queue-basiert) und IMAP-Empfang.

**Auth-Varianten:**
- **API-Key** — `X-API-Key` Header; nur für interne Microservice-zu-Microservice-Aufrufe
- **JWT Admin** — gültiges JWT mit Rolle `admin`

---

## Public

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `GET` | `/` | — | Hello World |
| `GET` | `/health` | — | Health check — `{ status, service, timestamp }` |

---

## Microservice API

| Method | Endpoint | Auth | Body | Response |
|--------|----------|------|------|----------|
| `POST` | `/emails` | API-Key | `to`*, `subject`*, `body`*, `cc`, `bcc`, `from`, `replyTo`, `isHtml`, `type`, `metadata`, `ignoreException` | `201 { id, priority, status }` |

---

## Admin — E-Mail-Verwaltung

> Alle `/admin`- und `GET/DELETE /emails`-Endpunkte erfordern ein JWT mit Rolle `admin`.

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/emails` | E-Mails auflisten — Query-Params: `status`, `type`, `page`, `limit` |
| `GET` | `/emails/:id` | Einzelne E-Mail nach ID |
| `DELETE` | `/emails/:id` | Pending E-Mail abbrechen (`409` wenn bereits gesendet/fehlgeschlagen) |
| `POST` | `/admin/emails` | E-Mail direkt als Admin einreihen — gleicher Body wie Microservice-Endpunkt |

---

## Admin — Settings

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/admin/settings` | Alle Konfigurationseinstellungen auflisten |
| `PUT` | `/admin/settings/:key` | Einstellung anlegen oder aktualisieren — Body: `{ value, description }` |
| `DELETE` | `/admin/settings/:key` | Einstellung nach Key löschen |
| `POST` | `/admin/refresh-key` | Gecachten JWT Public Key force-refreshen (nach Key-Rotation verwenden) |
