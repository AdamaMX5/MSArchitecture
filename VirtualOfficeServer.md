# VirtualOffice Server

Der VirtualOffice-Server (`/server`, Express + Node.js) ist gleichzeitig **Reverse-Proxy** (Auth, LiveKit, Kalender), **LiveKit-Token-/Egress-Aussteller**, **Auslieferer des statischen Client-Builds** und **integrierter Presence-WebSocket-Server**. Die schnelle Echtzeit-Kommunikation (Avatar-Positionen, Chat, Anrufe) läuft **nicht** über HTTP, sondern über den WebSocket auf `/ws`.

- Quelle HTTP-Routes: `server/src/index.ts`
- Quelle WebSocket-Protokoll: `server/src/presenceWs.ts`
- Quelle Typnamen (Client): `client/src/model/types.ts`

---

## Konfiguration (Environment Variables)

Aus `server/src/config.ts`:

| Variable | Default | Zweck |
|---|---|---|
| `PORT` | `3000` | HTTP-Port (WS teilt sich denselben Server) |
| `AUTH_URL` | `https://auth.freischule.info` | AuthService (Login/Register/Refresh-Proxy) |
| `OBJECT_URL` | `https://object.freischule.info` | ObjectService (Einladungs-Token-Persistenz) |
| `CLIENT_ORIGIN` | `https://office.freischule.info` | CORS-Origin |
| `LIVEKIT_URL` | `https://live.freischule.info` | LiveKit-Server |
| `LIVEKIT_API_KEY` | `devkey` | LiveKit API-Key (Token-/Egress-Signierung) |
| `LIVEKIT_API_SECRET` | `devsecret` | LiveKit API-Secret |
| `REDIS_URL` | `redis://localhost:6379` | Redis für Multi-Instanz-Presence (optional, Fallback In-Memory) |
| `ADMIN_URL` | `https://admin.freischule.info` | AdminClient (`/health`-Proxy für Service-Status) |

---

## HTTP-Endpoints (REST)

Alle Antworten sind JSON. Fehler werden als `{ "error": "<text>" }` mit passendem Statuscode zurückgegeben.

### Auth (Proxy → AuthService)

| Methode | Pfad | Body | Antwort | Beschreibung |
|---|---|---|---|---|
| `POST` | `/api/auth/login` | `{ email, password, deviceFingerprint?, deviceName? }` | normalisiertes Auth-Objekt (`accessToken`, `id`, …) + `Set-Cookie` (Refresh-Cookie) | Login durchreichen; Fehlerdetails werden zu einem `error`-String zusammengefasst |
| `POST` | `/api/auth/register` | `{ userid, repassword }` | normalisiertes Auth-Objekt + `Set-Cookie` | Registrierung durchreichen |
| `POST` | `/api/auth/refresh` | — (Refresh-Token kommt als Cookie) | `{ accessToken }` + neuer `Set-Cookie` | Access-Token erneuern; bei Fehler `401`-artig mit `error: "Session abgelaufen"` |

### LiveKit

| Methode | Pfad | Body | Antwort | Beschreibung |
|---|---|---|---|---|
| `POST` | `/api/livekit/token` | `{ room, identity, name? }` | `{ token, url }` | Stellt ein LiveKit-Access-Token aus (TTL 1 h, Grants: `roomJoin`, `canPublish`, `canSubscribe`) |
| `POST` | `/api/livekit/egress/start` | `{ room }` | `{ egressId }` | Startet serverseitige Aufnahme (Room-Composite → MP4 unter `recordings/{room}_{time}`) |
| `POST` | `/api/livekit/egress/stop` | `{ egressId }` | `{ ok: true }` | Stoppt eine laufende Egress-Aufnahme |

Ist `LIVEKIT_API_KEY`/`LIVEKIT_API_SECRET` nicht gesetzt → `500` mit `error: "LiveKit nicht konfiguriert"`.

### Presence / Sonstiges

| Methode | Pfad | Query/Body | Antwort | Beschreibung |
|---|---|---|---|---|
| `GET` | `/api/presence/users` | — | `{ users: [{ userId, name, department, title }] }` | Aktuell verbundene **echte** Nutzer (Bots `bot_*` und Gäste `g_*` herausgefiltert) — für das Empfangsmenü |
| `GET` | `/api/calendar/events` | `?url=<iCal-URL>&days=<n=14>` | `{ events: [...] }` | CORS-Bypass-Proxy für externe iCal/ICS-Feeds; bei Fehler `502` |
| `GET` | `/api/config` | — | `{ ok: true }` | Health-/Konfig-Ping des Clients |
| `GET` | `/api/services/status` | — | Durchgereichtes JSON von `ADMIN_URL/health` | Proxy auf den AdminClient-Health-Endpoint; bei Fehler `502` |

### Statischer Client

| Methode | Pfad | Beschreibung |
|---|---|---|
| `GET` | `/*` (Catch-all) | Liefert den statischen Vite-Build aus `client/dist`; alle nicht-API-Pfade fallen auf `index.html` zurück (SPA-Routing) |

> Hinweis: Die Einladungs-Logik (`server/src/inviteTokens.ts`) wird aktuell **nur beim WS-Connect** über den `invite`-Query-Parameter genutzt. Es gibt derzeit **keine** registrierte `/api/invite/*`-HTTP-Route (der Kommentar im Code ist vorausschauend, aber nicht verdrahtet).

---

## WebSocket — Echtzeit-Protokoll (`/ws`)

Der WS-Server ist direkt in den HTTP-Server eingehängt (`attachPresenceWs`). Er trägt Avatar-Positionen, Chat, Anruf-Signalisierung und Raum-Locks. Bei verfügbarem Redis arbeitet er im **Multi-Instanz-Modus** (Pub/Sub über `vo:main:events`), sonst lokal In-Memory.

### Verbindungsaufbau

```
wss://<host>/ws?token=<JWT>&userId=<id>&invite=<token>&bot_id=<id>
```

| Query-Param | Pflicht | Zweck |
|---|---|---|
| `token` | nein | JWT des eingeloggten Nutzers. Ohne Token → Gast (`g_<n>`) |
| `userId` | nein | Explizite User-ID aus dem Login-Response (`data.id`); hat Vorrang vor `sub` aus dem JWT, da `sub` oft die E-Mail ist |
| `invite` | nein | Einladungs-Token; legt Startraum und Gastnamen fest, benachrichtigt den Einlader beim Beitritt |
| `bot_id` | nein | Nur von `127.0.0.1`/`::1` akzeptiert — der Empfangs-/Admin-Bot identifiziert sich darüber |

**ID-Auflösung (`resolveUserId`):** localhost+`bot_id` → Bot-ID; sonst `userId`-Param; sonst `id`/`userId`/`sub` aus dem JWT-Payload; sonst Gast `g_<counter>`.

**Beim Connect** sendet der Server sofort einen `snapshot` an den neuen Client und broadcastet ein `user_joined` an alle anderen. Die letzte Position wird aus Redis (`vo:main:lastpos:<id>`, TTL 7 Tage) bzw. dem lokalen Cache geladen — Default `{ x: 60, y: 45 }`.

### Nachrichtenformat

Jede Nachricht ist ein JSON-Objekt mit Diskriminator-Feld `type`. Die Typnamen unten sind die Werte des `type`-Feldes; die TypeScript-Interfaces (Spalte „Interface") liegen in `client/src/model/types.ts`.

---

### Eingehend (Client → Server)

| `type` | Felder | Interface | Wirkung |
|---|---|---|---|
| `set_name` | `name`, `department?`, `title?` | `WsMsgSetName` | Setzt/aktualisiert Profil; löst `user_joined`-Broadcast aus. Beim ersten `set_name` eines eingeladenen Gasts: einmalige `notify_user`/`guest_joined`-Benachrichtigung an den Einlader |
| `move` | `x`, `y` | `WsMsgMove` | Speichert Position (+ `lastpos`); broadcastet `user_moved`. Hochfrequent — wird serverseitig nicht geloggt |
| `refresh_token` | `token` | `WsMsgRefreshToken` | Aktualisiert die `user_id` der Verbindung aus dem neuen JWT (nach Token-Refresh) |
| `notify_user` | `targetUserId`, `callType?` (`'call'` \| `'appointment'` \| `'guest_joined'`) | `WsMsgNotifyUser` | Zielgerichtete Benachrichtigung. **Ohne** `callType` → beim Empfänger als `new_message` zugestellt (Unread-Badge). **Mit** `callType` → als `notify_user` durchgereicht (Klingeln/TTS) |
| `chat` | `text` (max. 500 Zeichen) | `WsMsgChat` | Globaler Chat; broadcastet `chat` mit `userId` |
| `proximity_enter` | `roomName`, `userCount`, `prio` | `WsMsgProximityEnter` | Näherungs-Anruf starten. **Gäste (`g_*`) und Bots werden abgewiesen.** Broadcastet `proximity_call` |
| `proximity_switch` | `oldRoomName`, `newRoomName` | `WsMsgProximitySwitch` | Wechsel der Näherungsgruppe; broadcastet `proximity_switch` |
| `proximity_exit` | `roomName` | `WsMsgProximityExit` | Verlässt Näherungsgruppe; broadcastet `proximity_ended` an alle (Clients im Raum trennen sich selbst) |
| `meeting_bg` | `backgroundUrl` (`string \| null`) | `WsMsgMeetingBg` | Setzt/entfernt Meeting-Hintergrund. Nur eingeloggte User (kein `g_*`/`bot_*`). Broadcastet `meeting_bg` |
| `room_lock` | `room`, `locked` (bool) | `WsMsgRoomLock` | Sperrt/entsperrt einen physischen Raum (Owner = Sperrender); broadcastet `room_lock_update` |
| `room_knock` | `room` | `WsMsgRoomKnock` | Anklopfen an gesperrtem Raum; sendet `room_knock_request` an den Owner |
| `room_admit` | `room`, `userId` | `WsMsgRoomAdmit` | Nur der Owner: lässt einen Anklopfer ein; sendet `room_admitted` an `userId` |

Unbekannte `type`-Werte werden serverseitig geloggt und ignoriert.

---

### Ausgehend (Server → Client)

| `type` | Felder | Interface | Anlass |
|---|---|---|---|
| `snapshot` | `users: RemoteUser[]` | `WsSnapshot` | Direkt nach Connect — vollständige Liste aller anderen Nutzer |
| `user_joined` | `user_id`, `name`, `department?`, `title?`, `x`, `y` | `WsJoined` | Neuer Nutzer verbunden oder `set_name` (auch als Profil-Update genutzt) |
| `user_moved` | `user_id`, `x`, `y` | `WsMoved` | Ein Nutzer hat sich bewegt (hochfrequent) |
| `user_left` | `user_id` | `WsLeft` | Nutzer hat die Verbindung getrennt |
| `new_message` | `senderId` | `WsNewMessage` | Abgeleitet aus `notify_user` ohne `callType` — neue DM, Badge aktualisieren |
| `notify_user` | `targetUserId`, `senderId`, `callType?`, `guestName?` | `WsNotifyUser` | Eingehender Anruf/Termin/Gast-Beitritt (zielgerichtet) |
| `chat` | `userId`, `text` | `WsChatMessage` | Globale Chat-Nachricht |
| `proximity_call` | `fromUserId`, `fromName`, `roomName`, `userCount`, `prio` | `WsProximityCall` | Näherungs-Anruf wird angeboten |
| `proximity_switch` | `oldRoomName`, `newRoomName` | `WsProximitySwitch` | Näherungsgruppe hat gewechselt |
| `proximity_ended` | `roomName` | `WsProximityEnded` | Näherungs-Anruf beendet |
| `meeting_bg` | `backgroundUrl` (`string \| null`) | `WsMeetingBg` | Meeting-Hintergrund geändert |
| `room_lock_update` | `room`, `locked`, `lockerId?` | `WsRoomLockUpdate` | Raum-Sperrstatus geändert (auch Auto-Unlock bei Disconnect des Owners) |
| `room_knock_request` | `room`, `userId`, `name` | `WsRoomKnockRequest` | Jemand klopft am gesperrten Raum (nur an Owner) |
| `room_admitted` | `room` | `WsRoomAdmitted` | Owner hat Zutritt gewährt (nur an den Anklopfer) |

`RemoteUser` = `{ user_id, name, department?, title?, x, y }`.

**Routing-Regel:** `notify_user`, `room_knock_request` und `room_admitted` sind **zielgerichtet** (nur an `targetUserId`). Alle anderen Events werden an alle Verbindungen außer dem Absender (`user_id`) gebroadcastet.

---

## Redis-Layout (Multi-Instanz)

| Key / Channel | Typ | TTL | Inhalt |
|---|---|---|---|
| `vo:main:events` | Pub/Sub-Channel | — | Alle Presence-Events, getaggt mit `_src=<INSTANCE_ID>` (eigene Events werden ignoriert) |
| `vo:main:users:<userId>` | Hash | 3600 s (Heartbeat alle 60 s) | `{ name, department, title, x, y }` — aktive Sitzung |
| `vo:main:lastpos:<userId>` | Hash | 604800 s (7 Tage) | `{ x, y }` — letzte Position, überlebt Reconnect |

Fällt Redis aus, schaltet der Server transparent auf lokalen In-Memory-Betrieb um; bestehende Verbindungen werden bei Redis-Wiederkehr nachsynchronisiert.

---

## Reception-/Admin-Bot

Beim Serverstart verbinden sich `startReceptionBot()` und `startAdminBot()` (`server/src/presence.ts`) als Pseudo-Nutzer über localhost mit `bot_id`. Client-seitige Bot-Erkennung: `user_id` beginnt mit `bot_` **oder** `name` endet auf `_Bot`.
