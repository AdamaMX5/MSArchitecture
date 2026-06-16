# MessageService

> Base URL: `https://message.freischule.info`

Verwaltet Direktnachrichten (DMs) zwischen eingeloggten Nutzern. Nachrichten werden in MongoDB gespeichert (TTL 180 Tage). Echtzeit-Benachrichtigungen via WebSocket vom VirtualOfficeServer. Ungelesene Nachrichten können periodisch via EmailService versendet werden.

**Datenmodell:**

| Feld | Typ | Description |
|------|-----|-------------|
| `senderId` | String | JWT `sub` des Absenders |
| `recipientId` | String | JWT `sub` des Empfängers |
| `body` | String | Nachrichtentext (max. 5.000 Zeichen) |
| `readAt` | Date | `null` = ungelesen |
| `createdAt` | Date | Auto-gesetzt; TTL-Index: 180 Tage |
| `deletedBySender` | Boolean | Soft-Delete-Flag für Absender |
| `deletedByRecipient` | Boolean | Soft-Delete-Flag für Empfänger |

**Auth-Varianten:**
- **JWT** — Bearer Token; für alle Nutzer-Endpunkte
- **JWT Admin** — gültiges JWT mit Rolle `admin`

---

## Public

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `GET` | `/` | — | Hello World |
| `GET` | `/health` | — | Health check — `{ status: "ok", service: "MessageService" }` |

---

## User (Bearer JWT)

| Method | Endpoint | Body / Query | Description |
|--------|----------|--------------|-------------|
| `POST` | `/messages` | `recipientId`*, `body`* | Nachricht senden |
| `GET` | `/messages/unread-count` | — | `{ count }` — Anzahl ungelesener Nachrichten |
| `GET` | `/messages/inbox` | `?page&limit&unreadOnly` | Empfangene Nachrichten, neueste zuerst (default limit 20, max 100) |
| `GET` | `/messages/sent` | `?page&limit` | Gesendete Nachrichten (default limit 20, max 100) |
| `GET` | `/messages/conversation/:userId` | `?page&limit` | Alle Nachrichten mit einem bestimmten Nutzer (beide Richtungen), chronologisch |
| `PATCH` | `/messages/:id/read` | — | Einzelne Nachricht als gelesen markieren (nur Empfänger) |
| `PATCH` | `/messages/read-all` | `{ senderId* }` | Alle Nachrichten eines Absenders als gelesen markieren |
| `DELETE` | `/messages/:id` | — | Eigene Nachricht soft-löschen (Absender oder Empfänger); hard-delete wenn beide Flags gesetzt |

---

## Admin (JWT mit Rolle `admin`)

| Method | Endpoint | Query | Description |
|--------|----------|-------|-------------|
| `GET` | `/admin/messages` | `?senderId&recipientId&page&limit` | Alle Nachrichten mit optionalen Filtern auflisten |
| `DELETE` | `/admin/messages/:id` | — | Beliebige Nachricht löschen |
| `POST` | `/refresh-key` | — | Gecachten JWT Public Key force-refreshen |
