# AdminClient

> Base URL: `https://admin.freischule.info`

Reines React-Client-Frontend. Der Node-Server hat nur zwei Aufgaben: den gebauten Client ausliefern und einen öffentlichen Health-Aggregator bereitstellen — keine sonstige Backend-API. Alle Service-URLs, mit denen der Browser spricht (Login, AuthService, GitService, MediaService, …), werden zur Build-Zeit über `VITE_*`-Umgebungsvariablen ins Frontend gebacken; der Node-Server kennt sie nicht und proxied nichts davon. Jede Umgebung (Produktion, Staging, …) braucht dementsprechend einen eigenen Frontend-Build mit eigener `.env` — es gibt keine Laufzeit-Umschaltung zwischen Servergruppen.

---

## Public

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `GET` | `/` | — | Admin-UI (Login über Profil-Button, oben rechts) |
| `GET` | `/health` | — | Health check für alle in der Server-`.env` konfigurierten Services — gibt Status, Latenz und HTTP-Code pro Service zurück. Wird u.a. von VirtualOffice's `/api/services/status`-Proxy konsumiert. |

### `GET /health` — Response

```json
{
  "services": [
    { "key": "AUTH_SERVICE_URL",  "label": "AuthService",  "url": "https://auth.freischule.info",  "status": "ok",    "code": 200, "latency": 42  },
    { "key": "FREESCHOOL_URL",    "label": "FreeSchool",   "url": "https://api.freischule.info",   "status": "ok",    "code": 200, "latency": 38  },
    { "key": "EMAIL_SERVICE_URL", "label": "EmailService", "url": null,                             "status": "unconfigured"                        }
  ]
}
```

`status`-Werte: `"ok"` · `"error"` · `"unconfigured"`
