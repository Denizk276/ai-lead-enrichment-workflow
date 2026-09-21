# AI Lead Enrichment Workflow

Ein automatisierter **n8n-Workflow**, der eine Liste von Firmen mit KI-generierten Informationen anreichert. Das Projekt wurde als Portfolio-Projekt im Rahmen einer Bewerbung als Werkstudent im Bereich **AI & Automations** entwickelt.

## Was macht der Workflow?

Der Workflow liest Firmen aus einem Google Sheet ein und verarbeitet offene Datensätze automatisiert.

1. Liest Firmenname und Website aus einem Google Sheet ein
2. Vergibt für neue Datensätze automatisch den Status `Pending`
3. Verarbeitet nur Firmen mit dem Status `Pending`
4. Setzt den Status während der Verarbeitung auf `Processing`
5. Übergibt Firmenname und Website an ein KI-Modell über n8n
6. Das KI-Modell generiert:

   * eine passende Branchenzuordnung
   * ein kurzes Firmenprofil
   * einen personalisierten Gesprächseinstieg für Outreach
7. Die KI-Antwort wird als JSON geparst und auf die erwarteten Felder validiert
8. Bei erfolgreicher Verarbeitung wird der Status auf `Enriched` gesetzt
9. Bei einem Fehler wird der Status auf `Error` gesetzt
10. Bereits mit `Enriched` markierte Firmen werden bei späteren Läufen übersprungen

Der Workflow kann sowohl **manuell** als auch automatisch über einen **stündlichen Schedule Trigger** gestartet werden.

---

## Status-Logik

Der Workflow verwendet eine einfache Status-State-Machine:

```text
Leer
  ↓
Pending
  ↓
Processing
  ↓
Enriched
```

Bei einem Fehler:

```text
Processing
  ↓
Error
```

### Bedeutung der Status

| Status       | Bedeutung                                                    |
| ------------ | ------------------------------------------------------------ |
| `leer`       | Neuer Datensatz, noch nicht für die Verarbeitung vorbereitet |
| `Pending`    | Datensatz wartet auf Verarbeitung                            |
| `Processing` | Datensatz wird aktuell verarbeitet                           |
| `Enriched`   | Verarbeitung erfolgreich abgeschlossen                       |
| `Error`      | Verarbeitung ist fehlgeschlagen                              |

Ein Datensatz mit `Error` wird **nicht automatisch erneut verarbeitet**. Für einen erneuten Versuch kann der Status manuell wieder auf `Pending` gesetzt werden.

Dadurch werden bereits erfolgreich verarbeitete Datensätze nicht bei jedem neuen Workflow-Lauf erneut an das KI-Modell geschickt.

---

## Workflow-Architektur

Der Workflow ist als statusbasierter Prozess aufgebaut und verarbeitet die Firmen nacheinander.

```text
GetFirmen
    │
    ▼
Status leer?
    │
    ├── Ja → Status Pending setzen
    │
    └── Nein
           │
           ▼
     Pending filtern
           │
           ▼
    Loop Over Items
           │
           ▼
    Status Processing
           │
           ▼
        AI Agent
           │
           ├──────── Fehler ────────► Status Error
           │
           ▼
      JSON Parsing
           │
           ├──────── Fehler ────────► Status Error
           │
           ▼
    Ergebnisse validieren
           │
           ▼
      Status Enriched
           │
           ▼
     Google Sheets Update
           │
           ▼
          Wait
           │
           ▼
    Nächste Firma
```

Durch die Statusverwaltung kann der Workflow auch nach einem Abbruch oder Fehler bei einem späteren Lauf gezielt mit offenen Datensätzen weiterarbeiten.

---

## KI-Verarbeitung

Für die Anreicherung wird ein KI-Modell über die **Groq API** verwendet.

Aktuell kommt folgendes Modell zum Einsatz:

```text
openai/gpt-oss-20b
```

Das Modell erhält unter anderem:

```text
Firmenname
Website
```

und soll ausschließlich ein strukturiertes JSON mit folgenden Feldern zurückgeben:

```json
{
  "branche": "...",
  "kurzprofil": "...",
  "gespraechseinstieg": "..."
}
```

Die Antwort wird anschließend in einem n8n Code Node geparst und validiert.

Wenn kein gültiger JSON-Output zurückgegeben wird oder erforderliche Felder fehlen, wird die Verarbeitung als Fehler behandelt und der Datensatz erhält den Status `Error`.

---

## Verwendete Technologien

* **n8n** – Workflow-Automatisierung
* **n8n AI Agent / LangChain** – Integration des KI-Modells
* **Groq API** – LLM-Provider
* **GPT-OSS 20B** – aktuell verwendetes KI-Modell
* **Google Sheets API** – Datenquelle und Speicherung der Ergebnisse
* **JavaScript** – Parsing und Validierung der KI-Antwort
* **Schedule Trigger** – automatische Ausführung
* **Manual Trigger** – manuelle Ausführung
* **Docker / CasaOS** – Self-hosted n8n-Umgebung

---

## Technische Herausforderungen & Lösungen

### 1. OAuth mit self-hosted n8n

Bei der Einrichtung von Google Sheets mit einer lokal betriebenen n8n-Instanz musste berücksichtigt werden, dass OAuth-Redirects nicht ohne Weiteres auf eine private IP-Adresse zeigen können.

Gelöst wurde dies während der Einrichtung über einen SSH-Tunnel und eine temporäre Anpassung der n8n-Basis-URL auf `localhost`.

---

### 2. Rate Limits bei der KI-API

Bei der parallelen Verarbeitung mehrerer Firmen wurden Rate Limits der KI-API erreicht.

Deshalb verarbeitet der Workflow die Firmen über einen **Loop Over Items** Node einzeln und verwendet einen **Wait Node** zwischen den Anfragen.

Dadurch lässt sich die Anzahl der gleichzeitig gesendeten Requests kontrollieren.

---

### 3. Unzuverlässiger JSON-Output

LLMs können trotz entsprechender Anweisung zusätzlichen Text oder ungültiges JSON zurückgeben.

Deshalb wird die Antwort nach der KI-Verarbeitung nicht ungeprüft in das Google Sheet geschrieben.

Der Code Node:

* prüft, ob überhaupt ein Output vorhanden ist
* versucht, den Output als JSON zu parsen
* prüft, ob alle benötigten Felder vorhanden sind
* erzeugt bei einem Fehler einen Workflow-Fehler

Beispiel:

```javascript
const output = $json.output;

if (!output) {
  throw new Error("AI Agent hat kein Output zurückgegeben.");
}

let parsed;

try {
  parsed = JSON.parse(output);
} catch (error) {
  throw new Error("AI Agent hat kein gültiges JSON zurückgegeben.");
}

if (
  !parsed.branche ||
  !parsed.kurzprofil ||
  !parsed.gespraechseinstieg
) {
  throw new Error("AI Output enthält nicht alle erforderlichen Felder.");
}
```

---

### 4. Nachvollziehbarkeit des Verarbeitungsstatus

Anstatt den gesamten Prozess als einen einzigen linearen Durchlauf aufzubauen, verwendet der Workflow die Statuswerte `Pending`, `Processing`, `Enriched` und `Error`.

Dadurch ist im Google Sheet jederzeit erkennbar, ob ein Datensatz:

* noch wartet,
* gerade verarbeitet wird,
* erfolgreich verarbeitet wurde oder
* einen Fehler verursacht hat.

---

### 5. Fehlerbehandlung

Fehler des AI Agents und Fehler bei der Verarbeitung bzw. Validierung der KI-Antwort werden abgefangen.

Ein fehlgeschlagener Datensatz wird auf:

```text
Error
```

gesetzt.

Er wird dadurch nicht bei jedem automatischen Lauf erneut verarbeitet.

Für einen erneuten Versuch kann der Status manuell auf:

```text
Pending
```

zurückgesetzt werden.

---

## Beispiel

### Eingabe

| Firmenname    | Website     | Status  |
| ------------- | ----------- | ------- |
| Beispiel GmbH | beispiel.de | Pending |

### KI-Ergebnis

```json
{
  "branche": "Software",
  "kurzprofil": "Beispiel GmbH entwickelt Softwarelösungen für Unternehmen. Der Schwerpunkt liegt auf der Digitalisierung interner Prozesse.",
  "gespraechseinstieg": "Ich habe gesehen, dass Sie Unternehmen bei der Digitalisierung ihrer Prozesse unterstützen..."
}
```

### Ergebnis im Google Sheet

| Firmenname    | Status   | Branche  | Kurzprofil | Gesprächseinstieg |
| ------------- | -------- | -------- | ---------- | ----------------- |
| Beispiel GmbH | Enriched | Software | ...        | ...               |

---

## Setup

### Voraussetzungen

* laufende n8n-Instanz
* Google-Sheets-Credentials
* Groq-API-Credentials
* eigenes Google Sheet

### Google-Sheet-Struktur

Das Sheet benötigt folgende Spalten:

```text
Firmenname
Website
Status
Branche
Kurzprofil
Gespraechseinstieg
```

Für einen neuen Datensatz müssen zunächst nur `Firmenname` und `Website` eingetragen werden.

Der Workflow übernimmt die weitere Statusverwaltung.

### Workflow importieren

1. n8n öffnen
2. Workflow importieren
3. Google-Sheets-Credentials konfigurieren
4. Groq-Credentials konfigurieren
5. Google Sheet und entsprechende Spalten auswählen
6. Workflow manuell testen
7. Anschließend den Schedule Trigger aktivieren

---

## Projektstruktur

```text
ai-lead-enrichment-workflow/
│
├── README.md
│
├── workflows/
│   ├── v1-basic-enrichment.json
│   └── v2-production-enrichment.json
│
├── examples/
│   └── example-output.json
│
└── screenshots/
    └── workflow-v2.png
```

Die beiden Versionen dokumentieren die Entwicklung des Projekts von einem einfachen AI-Enrichment-Workflow hin zu einer robusteren, statusbasierten Version mit Fehlerbehandlung.

---

## V1 → V2

### V1

Die erste Version konzentrierte sich auf den grundlegenden End-to-End-Prozess:

```text
Google Sheets
    ↓
AI Agent
    ↓
JSON Parsing
    ↓
Google Sheets
```

Dabei standen die grundlegenden Integrationen im Vordergrund:

* n8n
* Google Sheets
* KI-Modell
* Prompting
* JSON
* JavaScript

### V2

Die zweite Version erweitert den Workflow um eine strukturierte Verarbeitung:

* Status-State-Machine
* `Pending` / `Processing` / `Enriched` / `Error`
* sequenzielle Verarbeitung
* kontrollierte Request-Frequenz
* JSON-Validierung
* Error Handling
* Wiederaufnahme offener Datensätze
* manuelles Zurücksetzen fehlgeschlagener Datensätze

Dadurch ist die zweite Version näher an einem Workflow, der auch bei wiederholter Ausführung und auftretenden Fehlern kontrolliert arbeiten kann.

---

## Weiterentwicklung

Mögliche nächste Schritte:

* Speicherung einer detaillierten Fehlermeldung pro Datensatz
* differenzierteres Retry-System
* Webhook-Trigger als zusätzliche Ausführungsart
* Anbindung eines CRM-Systems anstelle von Google Sheets
* zusätzliche Datenquellen für die Anreicherung
* weitere Validierung der KI-generierten Inhalte
* Austausch des LLM-Providers, ohne die grundlegende Workflow-Architektur zu verändern

---

## Ziel des Projekts

Das Projekt dient als praktisches Portfolio-Projekt, um Kenntnisse in den Bereichen **Workflow-Automatisierung, n8n, LLM-Integration, APIs, strukturierte Datenverarbeitung und JavaScript** zu zeigen.

Der Fokus liegt dabei nicht nur auf der KI-Generierung selbst, sondern auch auf der Frage, wie ein KI-basierter Workflow **kontrolliert, wiederholbar und fehlertolerant** aufgebaut werden kann.
