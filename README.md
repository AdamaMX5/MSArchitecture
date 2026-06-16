# MSArchitecture
Dese Repo ist nur die Beschreibung meiner einzelnen MicroServices, damit ClaudeCode die Übersicht behalten kann.
Clone dieses Git Repo in den globalen .claude Ornder. /benutzername/.claude/MSArchitecture

## Architecture.md

Die Architecture.md enthält die Übersicht der Architektur der Microservices

## CLAUDE

Im Claude Ordner sind die Setup Dateien für ClaudeCode
Die CLAUDE.md Datei ist die globale Claude.md und enthält das AgentTeam zum Programmieren.
Diese muss manuell eine Ebene höher kopiert werden.
Zusätzlich muss in die settings.json:

  "env": {
	  "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }

eine Beispiel settings.json ist auch im Verzeichnis enthalten
Leider können diese beiden Claude Dateien veraltet sein. 
