# GitService

> Base URL: `https://git.freischule.info`

Abstraktionsschicht für alle Git-Operationen. Abstrahiert den unterliegenden Git-Provider (GitHub oder Gitea), sodass alle Komponenten — Frontend, `gts` CLI und Claude Code — unabhängig vom Backend arbeiten.

**Auth-Varianten:**
- **JWT** (beliebige Rolle) — `Authorization: Bearer <token>`; für Frontend-Endpunkte
- **API-Key oder GITCLIENT JWT** — `X-API-Key` Header oder Bearer mit Rolle `GITCLIENT`; für CLI und Poller
- **API-Key** — `X-API-Key` Header; für Webhook-Endpunkte

**Input-Validierung auf allen Endpunkten:** `repo` matcht `/^[a-zA-Z0-9_.-]{1,100}$/`, `number` matcht `/^\d{1,9}$/`, `body` max 64 KiB. JSON-Body-Limit: 512 KiB.

---

## Public

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `GET` | `/` | — | Hello World |
| `GET` | `/health` | — | `{ status, service, provider, timestamp }` |

---

## Frontend (`/`) — JWT erforderlich

| Method | Endpoint | Body | Response | Description |
|--------|----------|------|----------|-------------|
| `GET` | `/repos` | — | `[{ name, fullName, url }]` | Alle Repositories des konfigurierten Git-Providers auflisten |
| `POST` | `/issue` | `repo`*, `title`*, `body`*, `labels` | `201 { number, url }` | Neues Issue anlegen — speichert Creator-E-Mail (aus JWT) für Benachrichtigungen |
| `POST` | `/issue/:number/comment` | `repo`*, `body`* | `201 { id }` | Kommentar als eingeloggter Nutzer posten |

---

## CLI (`/cli/`, `/issues`) — API-Key oder GITCLIENT JWT

| Method | Endpoint | Body / Query | Response | Description |
|--------|----------|--------------|----------|-------------|
| `GET` | `/issues` | — | `[{ number, title, body, state, creator, url, repo }]` | Alle offenen Issues über alle Repos — genutzt vom GitClient-Poller |
| `GET` | `/cli/issue/:number` | query: `repo`* | `{ number, title, body, state, creator, url }` | Einzelnes Issue lesen |
| `POST` | `/cli/issue/:number/comment` | `repo`*, `body`*, `type` | `201 { id, emailSent }` | Kommentar posten; `type: "question"` triggert E-Mail an Issue-Creator mit `Reply-To: gitservice@flussmark.de` |
| `PATCH` | `/cli/issue/:number/close` | `repo`* | `{ number, state: "closed" }` | Issue schließen |

**`type: "question"` E-Mail-Verhalten:** Kommentar wird immer gepostet. Wenn Creator-E-Mail bekannt (beim Issue-Anlegen gespeichert), wird E-Mail gesendet: Subject `[GitService #<n>] Frage zu: <title>`. Bei Fehler: `emailSent: false`, kein Fehler an Caller zurückgegeben.

---

## Webhook (`/webhook/`) — API-Key

| Method | Endpoint | Body | Response | Description |
|--------|----------|------|----------|-------------|
| `POST` | `/webhook/email-reply` | `from`, `subject`*, `body`* | `{ number, commentId }` | Vom EmailService aufgerufen, wenn eine Antwort bei `gitservice@flussmark.de` eingeht. Parst Issue-Nummer via `/\[GitService #(\d+)\]/` aus Subject und postet Antwort als Kommentar. `422` wenn Nummer nicht parsebar. Fallback: alle Repos abfragen wenn In-Memory-Store durch Neustart geleert. |

---

## `gts` CLI

Global installiert via `npm install -g` aus `git-client/`. Liest Auth aus `~/.gtsrc` (API-Key) oder fällt auf `~/.gitclient/tokens.json` (GITCLIENT JWT) zurück. Auto-Detection des Repos aus `git remote get-url origin`.

```bash
gts issue view <number> [--repo <name>]
gts issue comment <number> --body "..." [--type question] [--repo <name>]
gts issue close <number> [--repo <name>]
```

---

## GitClient (Developer PC Daemon)

Pollt `GET /issues` alle N Sekunden, startet `claude -p "<agent-team-prompt>"` im konfigurierten lokalen Repo-Verzeichnis für jedes neue offene Issue. Authentifiziert sich gegen AuthService mit dediziertem Account (Rolle: `GITCLIENT`). Tokens gespeichert unter `~/.gitclient/tokens.json` (Permissions `0600`).

```env
GIT_SERVICE_URL=https://git.freischule.info
GIT_SERVICE_API_KEY=...          # optional — fällt auf GITCLIENT JWT zurück
AUTH_SERVICE_URL=https://auth.freischule.info
REPO_PATHS=repo-name:/path/to/local/clone,...
POLL_INTERVAL=60
```
