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
| `GET` | `/profile/:userId` | optional JWT | Alle gespeicherten Prefixe eines Nutzers auflisten — als privat geflaggte Profile werden Fremden nicht gelistet |
| `GET` | `/profile/:userId/:prefix` | optional JWT | Einzelnes Profil nach User-ID und Prefix — `404` wenn nicht gefunden; **Privacy wird angewandt** (siehe unten) |
| `POST` | `/profile` | — | Profil anlegen oder aktualisieren — Body: `{ userId, prefix, profile: { ... } }` |

---

## Owner-only-Sichtbarkeit (`_privacy`)

Profile sind freie JSON-Dokumente. Der Eigentümer kann ein Profil oder einzelne Felder als **privat (owner-only)** markieren, indem er einen reservierten `_privacy`-Schlüssel ins Dokument legt. **Default ist öffentlich** — ohne `_privacy` wird das gesamte Dokument an alle ausgeliefert.

```jsonc
{
  "publicKey": "MFkw...",                       // bleibt öffentlich
  "keyBackup": "enc:...",                        // geschützt
  "channelKeyring": { "v1": "..." },             // geschützt
  "_privacy": {
    "profile": false,                            // true => ganzes Profil owner-only
    "fields": ["keyBackup", "channelKeyring"]    // diese Felder werden Fremden entfernt
  }
}
```

**Durchsetzung auf allen Lese-Pfaden** (REST `GET` + GraphQL Fremdabruf):

| Aufrufer | Verhalten |
|----------|-----------|
| Eigentümer (`JWT.sub == userId`, bzw. `my*`-Queries) | erhält das **vollständige** Dokument inkl. `_privacy` |
| Fremder / unauthentifiziert | `_privacy.fields` werden entfernt; der `_privacy`-Key selbst wird nie ausgeliefert |
| Fremder bei `_privacy.profile == true` | ganzes Profil verborgen — REST `404`, GraphQL `null`, nicht in der Prefix-Liste |

> `_privacy.fields` bezieht sich auf **Top-Level-Schlüssel** des Dokuments — sensible Werte gehören also auf die oberste Ebene, damit sie geschützt werden können (kein Tiefen-Stripping in verschachtelten Objekten).
>
> Der MessangerClient setzt z.B. `_privacy.fields: ["keyBackup", "channelKeyring"]`; `publicKey` bleibt damit für andere lesbar (E2E-Verschlüsselung), das Schlüsselmaterial nicht.
>
> **Hinweis:** Ein interner `X-API-Key`-Bypass ist (noch) nicht implementiert — die Durchsetzung ist aktuell rein JWT/Eigentümer-basiert.

---

## Reputation / Level-System (`/reputation`, `/internal/xp-events`)

Event-basiertes XP-/Level-System (siehe `WavyMania/übersichtPlan.md`, "Reputation: kein
eigener Service") — kein eigenständiger Microservice, sondern ein Event-Konsument innerhalb
des ProfileService. WaveService und ActivationService feuern bereits (fire-and-forget, 3
Versuche mit Backoff, blockiert nie den Nutzer-Request) `POST /internal/xp-events` bei
Join/Share/Contribution bzw. verifiziertem Check-in.

**XP pro Event-Typ:** `checkin`: 20 · `wave.join`: 10 · `wave.share`: 15 ·
`wave.contribution`: 25. Unbekannte Typen zählen **0 XP** (kein Fehler) — ein Aufrufer, der
künftig einen neuen Event-Typ einführt, bricht damit nie an einer noch nicht aktualisierten
ProfileService-Instanz.

**Level-Kurve:** kumulative Schwelle für Level `L` = `50 × (L-1) × L` (quadratisch — jede
Stufe kostet 100 XP mehr als die vorherige: L2=100, L3=300, L4=600, L5=1000, …). Level wird
**nicht** persistiert, sondern bei jedem Lesezugriff aus `xp` abgeleitet — der Schreibpfad
bleibt dadurch ein einzelnes atomares `$inc`.

| Method | Endpoint | Auth | Body / Response | Description |
|--------|----------|------|------------------|--------------|
| `POST` | `/internal/xp-events` | `X-API-Key` | `{ userId*, type*, waveId? }` → `200 { userId, xp, level }` | XP-Event verbuchen (atomares `$inc`); `waveId` wird entgegengenommen, aber nicht persistiert (reiner Akkumulator, kein Event-Log) |
| `GET` | `/reputation/:userId` | — | `{ userId, xp, level, xpIntoLevel, xpForNextLevel }` | Öffentlich, keine Auth — Level/XP ist bewusst ein sichtbares Reputations-Signal (wie die Status-Ränge im Whitepaper), analog zur Standard-Sichtbarkeit von `GlobalProfile.avatar`. Nutzer ohne Events → Default `{ xp: 0, level: 1 }` |

> **Internal-Auth:** folgt der AuthService-Konvention — ein `X-API-Key` pro Caller über
> `INTERNAL_API_KEY_<SERVICENAME>`, konstant-zeitiger Vergleich (`crypto.timingSafeEqual`),
> kein geteilter Master-Key. Aktuell registriert: `INTERNAL_API_KEY_WAVE_SERVICE`,
> `INTERNAL_API_KEY_ACTIVATION_SERVICE`. Jeder Key muss (außerhalb `NODE_ENV=test`)
> mindestens 32 Zeichen lang sein — der Service startet sonst nicht (Mirror von
> WaveServices `REFERRAL_SECRET`-Gate). `userId` ist auf `/^[A-Za-z0-9_.-]{1,128}$/`
> begrenzt (analog zu GitServices `repo`-Validierung), bevor es in den `$inc`-Upsert geht.
>
> **Bekannte Lücke (Security-Review, NIEDRIG):** `POST /internal/xp-events` verbucht ein
> Event immer, sobald der `X-API-Key` gültig ist — es gibt noch **keine** Idempotenz über
> eine Event-ID. Da WaveService/ActivationService bei einem Timeout bis zu 3× erneut senden
> (siehe `WaveService/src/services/profileEvents.js`), kann ein bereits serverseitig
> verbuchtes Event bei einer verlorenen Antwort doppelt gezählt werden. Für ein reines
> Anzeige-Feature unkritisch; sobald XP in eine CPA-/Burn-Abrechnung einfließt (Phase 3/4),
> braucht es eine caller-seitige Event-ID + Unique-Index hier, analog zu PaymentServices
> Stripe-Event-Idempotenz.

---

## GraphQL — Queries

| Operation | Auth | Arguments | Description |
|-----------|------|-----------|-------------|
| `myGlobalProfile` | JWT | — | Eigenes globales Profil zurückgeben |
| `globalProfile` | JWT | `userId: ID!` | Globales Profil eines beliebigen Nutzers nach ID |
| `myVirtualOfficeProfile` | JWT | — | Eigenes VirtualOffice-Profil |
| `myFreeSchoolProfile` | JWT | — | Eigenes FreeSchool-Profil |
| `profile` | optional JWT | `userId: ID!`, `prefix: String!` | Generisches Profil (Fremdabruf) als `JSON` — Privacy wird angewandt |
| `myProfile` | JWT | `prefix: String!` | Eigenes generisches Profil als `JSON` — vollständig |
| `messangerProfile` | optional JWT | `userId: ID!` | MessangerProfil (Fremdabruf, prefix `messanger`) — Privacy wird angewandt |
| `myMessangerProfile` | JWT | — | Eigenes MessangerProfil — vollständig |

---

## GraphQL — Mutations

| Operation | Auth | Input | Description |
|-----------|------|-------|-------------|
| `updateGlobalProfile` | JWT | `GlobalProfileInput` (`displayName`, `firstName`, `lastName`, `avatar`, `email`, `phone`, `address`, `matrixUsername`) | Globales Profil anlegen oder aktualisieren |
| `updateVirtualOfficeProfile` | JWT | `VirtualOfficeProfileInput` (`role`, `department`, `title`, `status`, `availability`, `workingHours`, `vacationPeriods`) | VirtualOffice-Profil anlegen oder aktualisieren |
| `updateFreeSchoolProfile` | JWT | `FreeSchoolProfileInput` (`role`, `classes`, `courses`) | FreeSchool-Profil anlegen oder aktualisieren |
| `updateProfile` | JWT | `prefix: String!`, `data: JSON!` | Eigenes generisches Profil (inkl. `_privacy`-Flags) anlegen oder aktualisieren |
