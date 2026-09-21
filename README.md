# AI Lead Enrichment Workflow

Ein automatisierter n8n-Workflow, der eine Liste von Firmen recherchiert und mit KI-generierten Informationen anreichert — gebaut als Portfolio-Projekt im Rahmen einer Bewerbung als Werkstudent im Bereich AI & Automations.

## Was macht der Workflow?

1. Liest eine Liste von Firmen (Name + Website) aus einem Google Sheet ein
2. Verfolgt den Verarbeitungsstatus jeder Firma über eine eigene Status-Spalte: `leer → Pending → Processing → Enriched`
3. Verarbeitet nur Firmen, die noch nicht (vollständig) angereichert wurden — bereits verarbeitete Zeilen werden bei jedem Lauf automatisch übersprungen
4. Schickt jede offene Firma an ein KI-Modell (Llama 3.3 über die Groq-API, angebunden über n8n's natives AI-Agent/LangChain-System), das generiert:
   - eine passende Branchenzuordnung
   - ein kurzes Firmenprofil (2 Sätze)
   - einen personalisierten Gesprächseinstieg für Outreach
5. Verarbeitet die KI-Antwort (JSON-Parsing) und schreibt die Ergebnisse zurück in die entsprechende Zeile im Sheet
6. Kann sowohl manuell gestartet als auch automatisch nach Zeitplan (stündlich) ausgeführt werden

## Verwendete Tools

- **n8n** (self-hosted via Docker/CasaOS) — Workflow-Engine
- **n8n AI Agent Node** (LangChain-Integration) — für die strukturierte KI-Anreicherung
- **Groq API** (Llama 3.3 / GPT-OSS 20B) — als LLM-Provider, angebunden über n8n's Chat-Model-Node
- **Google Sheets API** — als Datenquelle und Ziel, inklusive Status-Tracking
- **JavaScript** (n8n Code Node) — zum Parsen und Validieren der KI-Antworten
- **Schedule Trigger + Manual Trigger** — für flexible, wiederholbare Ausführung

## Workflow-Architektur

Der Workflow ist als **Status-State-Machine** aufgebaut, nicht als einfacher linearer Durchlauf:

```
GetFirmen (Sheet lesen)
    → If (Status leer?)
        → Status Empty → SetToPending → Sheet-Update (Status: Pending)
        → (bereits verarbeitete Zeilen werden hier herausgefiltert)
    → Status Pending (Filter)
    → Loop Over Items (verarbeitet 1 Firma nach der anderen)
        → SetToProcessing → Sheet-Update (Status: Processing)
        → AI Agent (Groq/Llama) → generiert Branche, Kurzprofil, Gesprächseinstieg
        → Code Node (JSON-Parsing der KI-Antwort)
        → Edit Fields → Sheet-Update (finale Werte, Status: Enriched)
        → Wait → zurück zum Loop
```

Dieses Muster macht den Workflow **idempotent**: bricht die Ausführung mittendrin ab (z. B. durch ein Rate-Limit oder einen Neustart), weiß der nächste Lauf anhand der Status-Spalte genau, wo er weitermachen muss, statt bereits verarbeitete Firmen erneut anzufragen.

## Technische Herausforderungen & Lösungen

- **OAuth mit self-hosted n8n über private IP:** Google blockiert OAuth-Redirects auf private IP-Adressen. Gelöst über einen SSH-Tunnel und temporäre Umkonfiguration der n8n-Basis-URL auf `localhost`.
- **Rate-Limits bei der KI-API:** Paralleles Verarbeiten aller Firmen überschritt das Groq-Rate-Limit. Gelöst mit einem Loop-Node (Batch-Größe 1) plus Wait-Node zwischen den Anfragen.
- **Unzuverlässiges JSON-Format:** Das LLM lieferte die Antwort gelegentlich mit zusätzlichem Text ummantelt. Gelöst mit gezieltem Parsing im Code-Node.
- **Nachvollziehbarkeit bei Abbrüchen:** Statt eines einzigen Durchlaufs ohne Zwischenstände wurde eine mehrstufige Status-Logik (Pending/Processing/Enriched) eingebaut, damit der Fortschritt jederzeit im Sheet sichtbar ist.

## Setup

1. n8n-Instanz (Cloud oder self-hosted) mit aktivierten Google-Sheets- und Groq-Chat-Model-Credentials
2. Groq API-Key unter [console.groq.com](https://console.groq.com)
3. `workflow.json` in n8n importieren (Workflows → Import from File)
4. Eigenes Google Sheet mit den Spalten `Firmenname, Website, Status, Branche, Kurzprofil, Gespraechseinstieg` verbinden

## Weiterentwicklungsideen

- Retry-/Error-Handling für den Fall, dass die KI kein valides JSON liefert (aktuell noch ohne Fallback)
- Webhook-Trigger als zusätzliche Ausführungsart neben Schedule/Manual
- CRM-Anbindung statt Google Sheets
- Wechsel zu Claude/OpenAI jederzeit möglich, da die Architektur providerunabhängig aufgebaut ist
