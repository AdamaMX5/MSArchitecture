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
