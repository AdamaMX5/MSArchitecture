# AuthService

> Base URL: `https://auth.freischule.info`

JWT-Authentifizierung, Login, Registrierung, Rollen und Permissions.

**JWT-Payload** enthält: `email`, Liste von `roles`, Dict von `permissions`

---

## Public

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `GET` | `/` | — | Hello World |
| `GET` | `/jwt/public-key` | — | RSA Public Key für JWT-Verifikation — `{ status, algorithm, public_key }` |

> **Public Key:** Jeder Microservice holt diesen einmalig beim Start und verifiziert JWTs autonom — kein Round-Trip pro Request. Algorithmus: RS256.

---

## User (`/user`)

| Method | Endpoint | Body | Response |
|--------|----------|------|----------|
| `POST` | `/user/check-email` | `email`* | `{ status }` — `"login"` wenn registriert, `"register"` wenn unbekannt |
| `POST` | `/user/login` | `email`*, `password`*, `device_fingerprint`, `device_name` | `id`, `email`, `roles`, `access_token`, `status`, `last_login` |
| `POST` | `/user/register` | `email`*, `repassword`*, `device_fingerprint`, `device_name` | wie login |
| `POST` | `/user/register-complete` | `email`*, `password`*, `repassword`*, `device_fingerprint`, `device_name` | wie login — `409` wenn bereits registriert |
| `GET` | `/user/verify-email` | query: `token`*, `user_id`* | `{ status }` |
| `POST` | `/user/password-reset-request` | query: `email`* | — |
| `POST` | `/user/reset-password` | query: `token`*, `user_id`*, `new_password`*, `repassword`* | `{ status }` |
| `POST` | `/user/refresh` | — (cookies: `refresh_token`, `csrf_token`) | `{ access_token }` + rotierte Cookies |
| `POST` | `/user/logout` | — (cookie: `refresh_token`) | `{ status }` |
| `POST` | `/user/logout-all` | — (JWT) | `{ status }` |

### Cookies (gesetzt bei login/register/refresh)

| Cookie | HttpOnly | Secure | SameSite | Path | Lifetime | Beschreibung |
|--------|----------|--------|----------|------|----------|--------------|
| `refresh_token` | ✅ | ✅ | Strict | `/user/refresh` | 14 Tage | Opakes Refresh-Token (JS kann nicht lesen) |
| `csrf_token` | ❌ | ✅ | Strict | `/user/refresh` | 14 Tage | CSRF-Token — JS liest und sendet als `X-CSRF-Token` Header |

**CSRF-Schutz:** Bei jedem `/user/refresh` werden beide Cookies benötigt. Der SHA-256-Hash des `csrf_token` wird zusammen mit dem `refresh_token`-Hash in der DB gespeichert — neue Tokens werden nur ausgegeben, wenn beide Hashes übereinstimmen. Bei `/user/logout-all` werden alle Refresh-Tokens (inkl. zugehöriger CSRF-Hashes) widerrufen.

### Login Flow (empfohlen: Email-first)

1. `/user/check-email` aufrufen → `"login"` → Passwortfeld zeigen; `"register"` → Passwort + Bestätigung + Register-Button
2. `/user/login` oder `/user/register-complete` aufrufen

**`status`-Werte bei Login:** `login` · `login_with_verify_email_send` · `register`

---

## Admin (`/admin`) — JWT mit Rolle `ADMIN` erforderlich

| Method | Endpoint | Body | Description |
|--------|----------|------|-------------|
| `GET` | `/admin/users` | — | Alle Nutzer mit vollständigen Daten |
| `GET` | `/admin/users/{user_id}` | — | Einzelnen Nutzer abrufen |
| `PATCH` | `/admin/users/{user_id}` | `email`, `roles`, `permissions`, `is_email_verify`, `hashed_password`, … | Partielles Update |
| `POST` | `/admin/users/import` | multipart JSON-Datei | Bulk-Upsert aus JSON — `{ created, updated, skipped }` |
| `POST` | `/admin/set_roles` | `user_id`*, `roles`* | Alle Rollen eines Nutzers ersetzen |
| `POST` | `/admin/set_permissions` | `user_id`*, `permissions`* | Alle Permissions eines Nutzers ersetzen |
| `POST` | `/admin/upsert_permission` | `user_id`*, `key`*, `value`* | Einzelnen Permission-Key anlegen oder aktualisieren |
| `POST` | `/admin/remove_permission` | `user_id`*, `key`* | Einzelnen Permission-Key entfernen |
| `POST` | `/admin/jwt/keys` | `private_key`*, `public_key`*, `algorithm`, `persist_to_files` | JWT-Schlüsselpaar setzen |
| `GET` | `/admin/jwt/key-storage` | — | JWT-Key-Storage-Info lesen |

---

## Internal (`/internal`) — `X-API-Key` erforderlich, niemals public

Für Service-zu-Service-Aufrufe (z. B. TicketService beim Gast-Checkout). Kein JWT, kein Bypass über ein Admin-Token — ausschließlich der `X-API-Key`-Header zählt.

**Auth:** Header `X-API-Key`, ein eigener Key pro aufrufendem Service (nie ein geteilter Master-Key). Konfiguriert über Env-Variablen mit Präfix `INTERNAL_API_KEY_<SERVICENAME>`, z. B. `INTERNAL_API_KEY_TICKET_SERVICE`. Vergleich erfolgt konstant-zeitig (`hmac.compare_digest`); bei Match wird der Servicename (aus dem Env-Var-Namen abgeleitet) als Caller-Identity zurückgegeben. Rotation/Widerruf: Env-Variable ändern + redeployen.

| Method | Endpoint | Body | Response |
|--------|----------|------|----------|
| `GET` | `/internal/ping` | — | `{ status, caller_service }` — Reachability-Check |
| `POST` | `/internal/users/provision` | `email`* | `{ userId, isNewUser }` |
| `PATCH` | `/internal/users/{user_id}/email` | `newEmail`* | `{ userId, email, isEmailVerified }` |

### Guest-Checkout Registrierungsprozess (Ticket-/Marktplatzkauf ohne vorherigen Account)

1. **`POST /internal/users/provision`** — Aufrufer (z. B. TicketService) schickt nur die E-Mail des Käufers.
   - E-Mail bereits registriert → bestehende `userId` wird zurückgegeben, kein Duplikat (Unique-Index auf `email` schützt race-sicher gegen zwei gleichzeitige Gast-Käufe mit derselben Adresse).
   - E-Mail unbekannt → neuer User mit Rolle `CONSUMER`, `is_email_verify: false`, zufälligem (nie ausgegebenem) Passwort. Ein `password_reset_token` wird erzeugt und die bestehende Passwort-Setzen-Mail (`/user/reset-password`-Link) verschickt.
   - Response enthält niemals Passwort-Hash oder Token.
2. Der Ticketkauf selbst hängt **nicht** von `is_email_verify` ab — der aufrufende Service verschickt Ticket/QR unabhängig davon sofort per eigener Bestellbestätigung bzw. zeigt es direkt auf der Erfolgsseite.
3. **Tippfehler-Korrektur:** Falls die Bestätigungsmail nie ankommt, kann der Käufer über einen kurzlebigen, an den Kaufvorgang gebundenen Claim-Token (den nur der aufrufende Service kennt und prüft) seine Adresse korrigieren lassen: **`PATCH /internal/users/{user_id}/email`**.
   - Harte serverseitige Sperren, unabhängig vom Aufrufer: nur erlaubt, wenn `is_email_verify === false` **und** die Rollen des Accounts exakt `{"CONSUMER"}` sind (sonst `403`). Damit ist ausgeschlossen, dass dieser Weg zur Account-Übernahme für bereits verifizierte oder für privilegierte Accounts (z. B. ein frisch angelegter, noch unverifizierter `ADMIN`) missbraucht wird.
   - `newEmail` bereits durch einen anderen Account belegt → `409` (inkl. Race-Schutz).
   - Nach Änderung wird automatisch der passende Flow erneut ausgelöst: Passwort-Setzen-Mail, falls noch kein Passwort gesetzt ist, sonst die reguläre Verify-Email-Mail (wie bei `register-complete`).
4. Sobald der Käufer über den Passwort-Setzen-Link ein echtes Passwort gesetzt hat (`/user/reset-password`), kann er sich ganz normal über `/user/check-email` → `/user/login` einloggen und seine Tickets in der App sehen. `CONSUMER` ist danach eine reguläre, weiter nutzbare Rolle (kein Einmal-Zustand).
