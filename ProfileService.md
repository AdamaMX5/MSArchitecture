# ProfileService

> Base URL: `https://profile.freischule.info`

Speichert mehrere Profile pro User-ID. Exponiert einen GraphQL-Endpunkt; alle Queries und Mutations erfordern ein gültiges JWT.

---

## HTTP — General

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `GET` | `/` | — | Hello World |
| `GET` | `/health` | — | Health check — `{ status: "ok" }` |
| `POST` | `/graphql` | JWT | GraphQL-Endpunkt (Apollo Server) |

---

## HTTP — Profile REST API (`/profile`)

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `GET` | `/profile/:userId` | — | Alle gespeicherten Prefixe eines Nutzers auflisten |
| `GET` | `/profile/:userId/:prefix` | — | Einzelnes Profil nach User-ID und Prefix — `404` wenn nicht gefunden |
| `POST` | `/profile` | — | Profil anlegen oder aktualisieren — Body: `{ userId, prefix, profile: { ... } }` |

---

## GraphQL — Queries

| Operation | Auth | Arguments | Description |
|-----------|------|-----------|-------------|
| `myGlobalProfile` | JWT | — | Eigenes globales Profil zurückgeben |
| `globalProfile` | JWT | `userId: ID!` | Globales Profil eines beliebigen Nutzers nach ID |
| `myVirtualOfficeProfile` | JWT | — | Eigenes VirtualOffice-Profil |
| `myFreeSchoolProfile` | JWT | — | Eigenes FreeSchool-Profil |

---

## GraphQL — Mutations

| Operation | Auth | Input | Description |
|-----------|------|-------|-------------|
| `updateGlobalProfile` | JWT | `GlobalProfileInput` (`displayName`, `firstName`, `lastName`, `avatar`, `email`, `phone`, `address`, `matrixUsername`) | Globales Profil anlegen oder aktualisieren |
| `updateVirtualOfficeProfile` | JWT | `VirtualOfficeProfileInput` (`role`, `department`, `title`, `status`, `availability`, `workingHours`, `vacationPeriods`) | VirtualOffice-Profil anlegen oder aktualisieren |
| `updateFreeSchoolProfile` | JWT | `FreeSchoolProfileInput` (`role`, `classes`, `courses`) | FreeSchool-Profil anlegen oder aktualisieren |
