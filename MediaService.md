# MediaService

> Base URL: `https://media.freischule.info`

Speichert und liefert hochgeladene Bilder und Videos. Dateien werden nach App und Ordner organisiert, sodass Teams in gemeinsamen Verzeichnissen zusammenarbeiten können.

**Disk-Layout:**

```
UPLOAD_DIR/
└── {app_name}/
    └── {folder}/          ← optional, Verschachtelung möglich: projects/design
        └── {uuid}.{ext}
```

Beispiel: `https://media.freischule.info/files/VirtualOffice/furniture/7856eab9-f5e8-4e41-a0b4-da25bf21fdf8.png`

---

## Upload

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `POST` | `/upload` | ✅ User (JWT) | Mediendatei hochladen |

**Multipart-Body:**

| Feld | Pflicht | Description |
|------|---------|-------------|
| `file` | ✅ | Die Mediendatei |
| `app_name` | ✅ | Identifizierende App, z.B. `"VirtualOffice"` |
| `folder` | — | Vorgeschlagener Speicherpfad, Verschachtelung möglich: `"projects/design"` |
| `name` | — | Menschenlesbarer Anzeigename |
| `description` | — | Kurzbeschreibung |

Response `201` — vollständiges Media-Dokument inkl. fertig verwendbarer `url`.

---

## File Access *(öffentlich, keine Auth erforderlich)*

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/files/:app_name/*` | Datei nach Pfad ausliefern — Browser-kompatibel |

```
GET /files/VirtualOffice/projects/design/3f2e1a….jpg
```

---

## Media — Metadaten & User Self-Service

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `GET` | `/media/:id` | ✅ User | Metadaten einer einzelnen Datei abrufen |
| `DELETE` | `/media/:id` | ✅ User | Eigene Datei löschen — `403` wenn nicht der Uploader |

---

## Browse — Ordner-Navigation

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `GET` | `/browse/:app_name` | ✅ User | Root einer App auflisten |
| `GET` | `/browse/:app_name/*` | ✅ User | Unterordner auflisten |

**Response-Shape:**
```json
{
  "path": "VirtualOffice/projects",
  "folders": ["design", "dev"],
  "files": [ { "...": "Media-Dokument" } ]
}
```

---

## Admin

> Alle `/admin`-Endpunkte erfordern ein JWT mit Rolle `ADMIN`.

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/admin/media` | Alle Dateien auflisten — Query-Params: `app_name`, `folder`, `uploaded_by`, `page`, `limit` |
| `DELETE` | `/admin/media/:id` | Beliebige Datei löschen, unabhängig vom Uploader |
