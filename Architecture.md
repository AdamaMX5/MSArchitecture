# Microservice Architecture

## Applications

| App | URL | Description |
|-----|-----|-------------|
| **FreeSchool** | `https://api.freischule.info` | Learning platform |
| **VirtualOffice** | `https://virtualoffice.freischule.info` | Virtual collaboration space |
| **MessangerClient** | `https://messanger.freischule.info` | Reduced Messanger for VirtualOffice |
| **AdminClient** | `https://admin.freischule.info` | Graphical UI for admin endpoints |

---

## Services

| Service | Base URL | Beschreibung |
|---------|----------|--------------|
| **AuthService** | `https://auth.freischule.info` | JWT-Auth, Login, Rollen, Permissions |
| **ProfileService** | `https://profile.freischule.info` | Nutzerprofile (GraphQL + REST), mehrere Profile pro User |
| **EmailService** | `https://email.freischule.info` | E-Mail-Versand (Queue), IMAP-Empfang, Admin-Verwaltung |
| **ExceptionService** | `https://exception.freischule.info` | Exception-Sammlung, Deduplication, Admin-Benachrichtigung |
| **ObjectService** | `https://object.freischule.info` | Generischer JSON-Objektspeicher (REST + GraphQL) |
| **MessageService** | `https://message.freischule.info` | Direkte Nachrichten (DMs) zwischen Nutzern |
| **PresenceService** | `https://presence.freischule.info` | Online-Status, SSE-Push für Echtzeit-Benachrichtigungen |
| **MediaService** | `https://media.freischule.info` | Upload und Auslieferung von Bildern/Videos |
| **GitService** | `https://git.freischule.info` | Abstraktion für GitHub/Gitea (Issues, Comments, CLI, Webhooks) |
| **RecordingService** | `https://recording.freischule.info` | LiveKit-Aufzeichnungen |
| **LiveKit** | `https://live.freischule.info` | Echtzeit-Kommunikation (Audio/Video/Screen) openSource |
| **AdminClient** | `https://admin.freischule.info` | Browser-Dashboard zur Verwaltung aller Services |

---

## Globale Konventionen

- **JWT-Payload** enthält: `sub` (userId), `email`, `roles[]`, `permissions{}`
- **JWT-Verifikation**: Jeder Service holt den Public Key einmalig beim Start von `https://auth.freischule.info/jwt/public-key` (RS256) — kein Round-Trip pro Request
- **API-Key-Auth**: Interne Microservice-Kommunikation über `X-API-Key` Header
- **CORS**: Wird zentral auf NGINX-Ebene gehandhabt, nicht in den einzelnen Services
- **Exception-Reporting**: Alle Services melden Fehler an den ExceptionService

---

## Service-Dokumentation

Jeder Service hat eine eigene Datei:

- [AuthService](./AuthService.md)
- [ProfileService](./ProfileService.md)
- [EmailService](./EmailService.md)
- [ExceptionService](./ExceptionService.md)
- [ObjectService](./ObjectService.md)
- [MessageService](./MessageService.md)
- [MediaService](./MediaService.md)
- [GitService](./GitService.md)
- [RecordingService](./RecordingService.md)
