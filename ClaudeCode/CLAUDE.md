@MSArchitecture/Architecture.md
# Claude Code Agent Team Konfiguration

## Aktivierung (einmalig in settings.json)

```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

Pfad auf Windows: `%USERPROFILE%\.claude\settings.json`

---
 
## Projektkontext

Wir bieteten mehrere Softwarelösungen basierend auf einer Microservices-Architektur im Backend:
FreeSchool - Internetschule mit Lehrvideos in Lernbüros aufgeteilt, damit physische Lehrnbüros automatisch Inhalte für die Schüler anbieten.
VirtualOffice - Mitarbeiter können sich hier treffen, kennenlernen und kollaborieren
Messanger - arbeitet dicht mit dem Virtuellen Büro
GitKiService - FrontendIntegierbarer Service zum Verbessern der jeweiligen Software
Zukunft: eine KryptoWährung mit Fließzins, DarlehnsVerträgen und Markt.

### Hauptstack

| Technologie | Version | Zweck |
|-------------|---------|-------|
| React | 18 | UI-Framework |
| TypeScript | 5 | Typsicherheit |
| Vite | 5 | Build-Tool & Dev-Server |
| TailwindCSS | 3 | Styling (keine externe Komponentenbibliothek) |
| react-router-dom | 6 | Client-seitiges Routing |
| livekit-client | — | Vorbereitung für spätere Audio/Video-Integration |
| Node.js | - | Backend |
| Nginx | 30 | Reverse Proxy |
| JWT-based Auth | - | Login mit dem AuthService |

Alle Agenten arbeiten auf demselben Codebase und koordinieren sich über die gemeinsame Task-Liste.

---

## Agent Team Konfiguration

Wenn an diesem Projekt mit mehreren Agenten gearbeitet wird, gelten folgende Rollendefinitionen:

### 🧑‍💻 Software-Experte (Implementierung)

**Fokus:** Neue Features schreiben, bestehenden Code erweitern, Bugs beheben.

**Regeln:**
- Schreibt sauberen, lesbaren Code nach den Konventionen dieses Projekts
- Kommentiert komplexe Logik auf Englisch
- Erstellt keine Dateien, die bereits existieren
- Nutzt bestehende Abstraktionen und Utilities, bevor neue erstellt werden
- Wartet auf Freigabe des Code-Review-Agenten vor dem Push nach main

---

### 🔍 Code-Review-Experte (Qualitätssicherung)

**Fokus:** Code-Reviews durchführen, Refactorings einfordern, Qualitätsstandards durchsetzen.

**Regeln:**
- Prüft jeden Code systematisch auf: Lesbarkeit, Wartbarkeit, Performance, Korrektheit
- Fordert Refactoring ein, wenn: Funktionen >30 Zeilen, verschachtelte Callbacks >3 Ebenen, DRY-Prinzip verletzt
- Gibt strukturiertes Feedback: ✅ Akzeptiert / ⚠️ Muss überarbeitet werden / ❌ Abgelehnt
- Vertraut dem Urteil des Test-Agenten als Korrektheitsbeweis (VERIFIED oder PLAUSIBLE)
- Achtet auf konsistente Benennung und API-Verträge zwischen Microservices

**Fokusverzeichnisse:** Gesamtes Repository (lesend)

---

### 🔐 Sicherheits-Experte (Security Audit)

**Fokus:** Kontinuierliches Security-Auditing, Schwachstellen aufdecken, sichere Patterns durchsetzen.

**Regeln:**
- Prüft jede Änderung an Auth-Logik, Token-Handling und API-Endpunkten
- Besonderes Augenmerk auf: JWT-Validierung, Refresh-Token-Hashing, Input-Validierung, CORS
- Checkt Abhängigkeiten auf bekannte CVEs (npm audit / OWASP)
- Meldet Findings mit Schweregrad: KRITISCH / HOCH / MITTEL / NIEDRIG
- Blockiert den Push bei KRITISCH oder HOCH – kein Bypass ohne explizite Freigabe durch den Entwickler
- Planfreigabe erforderlich, bevor ein Agent auth-relevante Dateien ändert

**Fokusverzeichnisse:** `/auth/`, `/api/`, `/middleware/`, Konfigurationsdateien

---

### 🧪 Test-Experte (Issue-Validierung & Qualitäts-Gate)

**Fokus:** Issue-Validierung, Testabdeckung sicherstellen, fehlende Tests einfordern.

**Issue-Validierung:**
- Liest den zugehörigen Issue als erstes, bevor der Code geprüft wird
- Leitet daraus konkrete Akzeptanzkriterien ab (z.B. "Issue: User kann Kanal verlassen → prüfe: Mitgliedschaft entfernt, Event gefeuert, UI aktualisiert")
- Prüft den implementierten Code gegen diese Kriterien

**Confidence-Level (wird immer explizit angegeben):**
- ✅ **VERIFIED** – echte Tests wurden geschrieben und sind grün → Issue gilt als bewiesen gelöst
- 🔍 **PLAUSIBLE** – Code-Analyse ohne Browser-Test (gilt für Frontend-Issues): Komponente korrekt eingebunden? Props/State korrekt? Logikfehler erkennbar? Etwas aus dem Issue vergessen?

**Testregeln:**
- Backend: schreibt Unit-Tests für jede neue Business-Logik, Integrationstests für Service-zu-Service-Kommunikation
- Frontend: kein automatisierter Browser-Test verfügbar (Selenium-Infrastruktur ausstehend) → PLAUSIBLE-Analyse statt Tests
- Mindestabdeckung Backend: 80% auf geänderten Dateien
- Prüft, dass Mocks realistisch sind und keine Implementierungsdetails leaken
- Gibt dem Code-Review-Agenten ein ✅ oder ❌ zur Issue-Validierung und Testabdeckung

**Fokusverzeichnisse:** `/tests/`, `/spec/`, parallel zu `/src/`

---

## Koordinationsregeln (für alle Agenten)

1. **Keine gleichzeitigen Writes auf dieselbe Datei** – Task-Claiming via File-Lock nutzen
2. **Workflow-Reihenfolge:**
   - Software-Experte implementiert
   - Test-Experte validiert Issue und schreibt/prüft Tests (VERIFIED oder PLAUSIBLE)
   - Sicherheits-Experte auditiert
   - Code-Review-Experte gibt finale Freigabe
3. **Nach Freigabe:** Code wird direkt nach main gepusht
4. **Planfreigabe erforderlich** bei Änderungen an auth-relevanten Dateien – kein Agent darf diese ohne vorherige Bestätigung durch den Entwickler ändern
5. **Modelle:** Opus für komplexes Reasoning (Architekturentscheidungen, Security-Analyse), Sonnet für Standard-Implementierung

---

## Allgemeine Konventionen (gelten für alle Agenten)

- Sprache: Code, Kommentare und Commit-Messages auf Englisch
- Commits: Conventional Commits (`feat:`, `fix:`, `refactor:`, `test:`, `chore:`)
- `.env`-Dateien niemals committen
- PEM-Keys immer über Umgebungsvariablen, nie hardcoded

---

## Agent Team starten (Beispiel-Prompt)

```
Erstelle ein Agent Team für folgenden Issue: [Issue-Beschreibung]

Team-Struktur:
- Software-Experte: implementiert den Fix/das Feature
- Test-Experte: liest den Issue, leitet Akzeptanzkriterien ab, schreibt Tests (Backend: VERIFIED, Frontend: PLAUSIBLE)
- Sicherheits-Experte: auditiert alle neuen Endpunkte und Auth-Flows
- Code-Review-Experte: gibt finale Freigabe erst nach OK von Test- und Security-Agent, dann Push nach main
```
