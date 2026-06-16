# ExceptionService

> Base URL: `https://exception.freischule.info`

Empfängt Exceptions von anderen Services, speichert sie in MongoDB (Auto-Delete nach 14 Tagen) und benachrichtigt Admins via EmailService.

**Auth-Varianten:**
- **API-Key** — `X-API-Key` Header; für interne Microservice-Aufrufe
- **JWT** — Bearer Token; für nutzerseitig gemeldete Exceptions
- **JWT Admin** — gültiges JWT mit Rolle `admin`

**Exception-Fingerprinting:** Exceptions werden dedupliziert durch Hashing von `service + normalizedMessage` (UUIDs, numerische IDs, Hex-Werte und Zeilennummern werden herausgefiltert). Jeder eindeutige Fingerprint erzeugt einen `ExceptionType`-Datensatz.

**E-Mail-Override pro Typ** (`emailOverride`-Feld auf `ExceptionType`):
- `null` — `sendEmail`-Flag des eingehenden Requests respektieren
- `true` — immer E-Mail senden
- `false` — nie E-Mail senden

---

## Public

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `GET` | `/` | — | Hello World |
| `GET` | `/health` | — | Health check — `{ status, service }` |

---

## Exception Reporting

| Method | Endpoint | Auth | Body | Description |
|--------|----------|------|------|-------------|
| `POST` | `/report` | API-Key oder JWT | `service`*, `message`*, `stack`, `statusCode`, `method`, `path`, `sendEmail`, `metadata` | Exception melden; `ExceptionType` upserten und optional Admin-E-Mail senden |

---

## Admin — Exceptions

> Alle Endpunkte erfordern JWT mit Rolle `admin`.

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/exceptions` | Exceptions auflisten — Query-Params: `service`, `emailSent`, `sendEmail`, `typeId`, `dateFrom`, `dateTo`, `page`, `limit` |
| `GET` | `/exceptions/:id` | Einzelne Exception (mit populiertem `typeId`) |
| `DELETE` | `/exceptions/:id` | Exception löschen |

---

## Admin — Exception Types

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/types` | Exception-Typen auflisten — Query-Params: `service`, `emailOverride`, `page`, `limit` |
| `GET` | `/types/:id` | Einzelnen Exception-Typ abrufen |
| `PATCH` | `/types/:id` | `emailOverride` setzen — Body: `{ emailOverride: true \| false \| null }` |
| `DELETE` | `/types/:id` | Typ löschen (wird beim nächsten Auftreten neu angelegt) |

---

## Admin — Key Management

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/refresh-key` | Gecachten JWT Public Key force-refreshen (nach Key-Rotation) |
