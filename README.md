# AI Lead Enrichment Workflow

Ein automatisierter n8n-Workflow, der eine Liste von Firmen recherchiert und mit KI-generierten Informationen anreichert — gebaut als Portfolio-Projekt im Rahmen einer Bewerbung als Werkstudent im Bereich AI & Automations.

## Was macht der Workflow?

1. Liest eine Liste von Firmen (Name + Website) aus einem Google Sheet ein
2. Schickt jede Firma an ein KI-Modell (Llama 3.3 über die Groq-API), das drei Dinge generiert:
   - eine passende Branchenzuordnung
   - ein kurzes Firmenprofil (2 Sätze)
   - einen personalisierten Gesprächseinstieg für Outreach
3. Verarbeitet die KI-Antwort (JSON-Parsing) und schreibt die Ergebnisse zurück in die entsprechende Zeile im Sheet
4. Markiert die Zeile als "Enriched", sobald sie verarbeitet wurde

## Verwendete Tools

- **n8n** (self-hosted via Docker/CasaOS) — Workflow-Engine
- **Google Sheets API** — als Datenquelle und Ziel
- **Groq API** (Llama 3.3 70B) — für die KI-Anreicherung, angebunden über die OpenAI-kompatible Schnittstelle
- **JavaScript** (n8n Code Node) — zum Parsen und Validieren der KI-Antworten

## Technische Herausforderungen & Lösungen

Beim Aufbau sind einige praxisnahe Probleme aufgetreten, die ich gelöst habe:

- **OAuth mit self-hosted n8n über private IP:** Google blockiert OAuth-Redirects auf private IP-Adressen. Gelöst über einen SSH-Tunnel und temporäre Umkonfiguration der n8n-Basis-URL auf `localhost`.
- **Rate-Limits bei der KI-API:** Bei parallelem Verarbeiten aller Firmen wurde das Groq-Rate-Limit überschritten. Gelöst mit einem Loop-Node (Batch-Größe 1) plus Wait-Node zwischen den Anfragen.
- **Unzuverlässiges JSON-Format:** Das LLM lieferte die Antwort gelegentlich mit Markdown-Codeblöcken oder zusätzlichem Text ummantelt. Gelöst mit robustem Parsing (Regex-Extraktion des JSON-Teils + Fehlerbehandlung, statt den Workflow bei einem fehlerhaften Format komplett abbrechen zu lassen).

## Workflow-Struktur

`Google Sheets (Read)` → `Loop Over Items` → `KI-Node (Groq/Llama)` → `Code Node (Parsing)` → `Google Sheets (Update)` → zurück zum Loop

## Setup

1. n8n-Instanz (Cloud oder self-hosted) mit aktivierten Google Sheets- und HTTP-Request-/OpenAI-Node-Credentials
2. Groq API-Key unter [console.groq.com](https://console.groq.com)
3. `workflow.json` in n8n importieren (Workflows → Import from File)
4. Eigenes Google Sheet mit den Spalten `Firmenname, Website, Status, Branche, Kurzprofil, Gesprächseinstieg` verbinden

## Weiterentwicklungsideen

- Fehlerbehandlung bei komplett fehlgeschlagenem KI-Aufruf (Retry-Logik)
- CRM-Anbindung statt Google Sheets
- Wechsel zu Claude/OpenAI möglich, da die Architektur providerunabhängig aufgebaut ist
