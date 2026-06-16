# AdminClient

> Base URL: `https://admin.freischule.info`

Grafische UI zur Verwaltung aller Microservices. Bietet ein Browser-basiertes Dashboard und einen öffentlichen Health-Endpunkt.

---

## Public

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `GET` | `/` | — | Admin-UI (Login über Profil-Button, oben rechts) |
| `GET` | `/health` | — | Health check für alle konfigurierten Services — gibt Status, Latenz und HTTP-Code pro Service zurück |

### `GET /health` — Response

```json
{
  "activeGroup": "Produktion",
  "services": [
    { "key": "authServiceUrl",  "label": "AuthService",  "url": "https://auth.freischule.info",  "status": "ok",    "code": 200, "latency": 42  },
    { "key": "freeSchoolUrl",   "label": "FreeSchool",   "url": "https://api.freischule.info",   "status": "ok",    "code": 200, "latency": 38  },
    { "key": "emailServiceUrl", "label": "EmailService", "url": null,                            "status": "unconfigured"                        }
  ]
}
```

`status`-Werte: `"ok"` · `"error"` · `"unconfigured"`
