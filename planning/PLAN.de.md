# FinAlly — KI-Trading-Workstation

> ⚠️ **VERALTET — Stand `6b568a9`. Maßgeblich ist [`PLAN.md`](PLAN.md).**
>
> Diese Übersetzung wurde vor der Überarbeitung angefertigt und weicht inhaltlich deutlich ab. Sie enthält unter anderem noch: den Widerspruch zwischen `db/`-Bind-Mount (§4) und benanntem Volume (§11), den falschen Skillnamen `cerebras-inference` (§9), die unerfüllbare Zusage zur Fehlerrückmeldung im Chat (§9) sowie eine unspezifizierte SSE-Nutzlast (§6).
>
> **Nicht als Bauvorlage verwenden.** Agenten lesen `PLAN.md`; dieses Dokument dient nur als deutschsprachige Orientierung zum damaligen Stand.

## Projektspezifikation

## 1. Vision

FinAlly (Finance Ally) ist eine visuell beeindruckende, KI-gestützte Trading-Workstation, die Live-Marktdaten streamt, den Nutzern den Handel mit einem simulierten Portfolio ermöglicht und einen LLM-Chat-Assistenten integriert, der Positionen analysieren und im Auftrag des Nutzers Trades ausführen kann. Es sieht aus und fühlt sich an wie ein modernes Bloomberg-Terminal mit einem KI-Copiloten.

Dies ist das Capstone-Projekt für einen Kurs über agentisches KI-Coding. Es wird vollständig von Coding-Agenten gebaut und demonstriert, wie orchestrierte KI-Agenten eine produktionsreife Full-Stack-Anwendung erzeugen können. Die Agenten interagieren über Dateien in `planning/`.

## 2. Nutzererlebnis

### Erster Start

Der Nutzer führt einen einzigen Docker-Befehl aus (oder ein mitgeliefertes Startskript). Ein Browser öffnet sich auf `http://localhost:8000`. Kein Login, keine Registrierung. Sofort sichtbar sind:

- Eine Watchlist mit 10 Standard-Tickern und live aktualisierten Kursen in einem Raster
- 10.000 $ virtuelles Guthaben
- Eine dunkle, datenreiche Trading-Terminal-Ästhetik
- Ein KI-Chat-Panel, bereit zur Unterstützung

### Was der Nutzer tun kann

- **Kurse live verfolgen** — Kurse blinken grün (Aufwärtsbewegung) oder rot (Abwärtsbewegung) mit dezenten CSS-Animationen, die ausblenden
- **Sparkline-Minicharts betrachten** — Kursverlauf neben jedem Ticker in der Watchlist, im Frontend aus dem SSE-Stream seit dem Laden der Seite akkumuliert (die Sparklines füllen sich fortlaufend)
- **Einen Ticker anklicken**, um einen größeren Detailchart im Hauptchart-Bereich zu sehen
- **Aktien kaufen und verkaufen** — nur Market-Orders, sofortige Ausführung zum aktuellen Kurs, keine Gebühren, kein Bestätigungsdialog
- **Das Portfolio überwachen** — eine Heatmap (Treemap), die Positionen nach Gewichtung dimensioniert und nach G&V einfärbt, sowie ein G&V-Chart, der den Gesamtwert des Portfolios über die Zeit verfolgt
- **Eine Positionstabelle einsehen** — Ticker, Stückzahl, Durchschnittskosten, aktueller Kurs, unrealisierte G&V, prozentuale Veränderung
- **Mit dem KI-Assistenten chatten** — Fragen zum Portfolio stellen, Analysen erhalten und die KI Trades ausführen sowie die Watchlist per natürlicher Sprache verwalten lassen
- **Die Watchlist verwalten** — Ticker manuell oder über den KI-Chat hinzufügen/entfernen

### Visuelles Design

- **Dunkles Theme**: Hintergründe etwa `#0d1117` oder `#1a1a2e`, gedämpfte graue Ränder, kein reines Schwarz
- **Kurs-Blink-Animationen**: kurzes grünes/rotes Aufleuchten des Hintergrunds bei Kursänderung, ausblendend über ca. 500 ms via CSS-Transitions
- **Verbindungsstatus-Anzeige**: ein kleiner farbiger Punkt (grün = verbunden, gelb = Wiederverbindung, rot = getrennt), sichtbar im Header
- **Professionelles, datendichtes Layout**: inspiriert von Bloomberg-/Trading-Terminals — jeder Pixel verdient seinen Platz
- **Responsiv, aber Desktop-first**: optimiert für breite Bildschirme, funktionsfähig auf Tablets

### Farbschema
- Akzent-Gelb: `#ecad0a`
- Primär-Blau: `#209dd7`
- Sekundär-Violett: `#753991` (Submit-Buttons)

## 3. Architekturüberblick

### Ein Container, ein Port

```
┌─────────────────────────────────────────────────┐
│  Docker-Container (Port 8000)                   │
│                                                 │
│  FastAPI (Python/uv)                            │
│  ├── /api/*          REST-Endpunkte             │
│  ├── /api/stream/*   SSE-Streaming              │
│  └── /*              Ausliefern statischer      │
│                      Dateien (Next.js-Export)   │
│                                                 │
│  SQLite-Datenbank (per Volume eingebunden)      │
│  Hintergrund-Task: Marktdaten-Polling/-Sim      │
└─────────────────────────────────────────────────┘
```

- **Frontend**: Next.js mit TypeScript, gebaut als statischer Export (`output: 'export'`), von FastAPI als statische Dateien ausgeliefert
- **Backend**: FastAPI (Python), verwaltet als `uv`-Projekt
- **Datenbank**: SQLite, einzelne Datei unter `db/finally.db`, für Persistenz per Volume eingebunden
- **Echtzeitdaten**: Server-Sent Events (SSE) — einfacher als WebSockets, unidirektionaler Push Server→Client, funktioniert überall
- **KI-Integration**: LiteLLM → OpenRouter (Cerebras für schnelle Inferenz), mit Structured Outputs für die Trade-Ausführung
- **Marktdaten**: über Umgebungsvariablen gesteuert — standardmäßig Simulator, echte Daten über die Massive-API, falls ein Schlüssel hinterlegt ist

### Warum diese Entscheidungen

| Entscheidung | Begründung |
|---|---|
| SSE statt WebSockets | Unidirektionaler Push genügt vollkommen; einfacher, keine bidirektionale Komplexität, universelle Browser-Unterstützung |
| Statischer Next.js-Export | Single Origin, keine CORS-Probleme, ein Port, ein Container, einfaches Deployment |
| SQLite statt Postgres | Keine Authentifizierung = kein Mehrbenutzerbetrieb = kein Datenbankserver nötig; autark, ohne Konfiguration |
| Ein einzelner Docker-Container | Studierende führen einen Befehl aus; kein docker-compose in der Produktion, keine Service-Orchestrierung |
| uv für Python | Schnelles, modernes Python-Projektmanagement; reproduzierbare Lockfile; genau das, was Studierende lernen sollten |
| Nur Market-Orders | Entfällt Orderbuch, Limit-Order-Logik, Teilausführungen — drastisch einfachere Portfolio-Mathematik |

---

## 4. Verzeichnisstruktur

```
finally/
├── frontend/                 # Next.js-TypeScript-Projekt (statischer Export)
├── backend/                  # FastAPI-uv-Projekt (Python)
│   └── db/                   # Schemadefinitionen, Seed-Daten, Migrationslogik
├── planning/                 # Projektweite Dokumentation für Agenten
│   ├── PLAN.md               # Dieses Dokument
│   └── ...                   # Weitere Referenzdokumente für Agenten
├── scripts/
│   ├── start_mac.sh          # Docker-Container starten (macOS/Linux)
│   ├── stop_mac.sh           # Docker-Container stoppen (macOS/Linux)
│   ├── start_windows.ps1     # Windows-PowerShell-Pendant zum Starten
│   └── stop_windows.ps1      # Windows-PowerShell-Pendant zum Stoppen
├── test/                     # Playwright-E2E-Tests + docker-compose.test.yml
├── db/                       # Volume-Mount-Ziel (SQLite-Datei liegt hier zur Laufzeit)
│   └── .gitkeep              # Verzeichnis existiert im Repo; finally.db ist gitignored
├── Dockerfile                # Mehrstufiger Build (Node → Python)
├── docker-compose.yml        # Optionaler Komfort-Wrapper
├── .env                      # Umgebungsvariablen (gitignored, .env.example ist eingecheckt)
└── .gitignore
```

### Wichtige Abgrenzungen

- **`frontend/`** ist ein eigenständiges Next.js-Projekt. Es weiß nichts über Python. Es kommuniziert mit dem Backend über `/api/*`-Endpunkte und `/api/stream/*`-SSE-Endpunkte. Die interne Struktur liegt beim Frontend-Engineer-Agenten.
- **`backend/`** ist ein eigenständiges uv-Projekt mit eigener `pyproject.toml`. Es verantwortet die gesamte Serverlogik einschließlich Datenbankinitialisierung, Schema, Seed-Daten, API-Routen, SSE-Streaming, Marktdaten und LLM-Integration. Die interne Struktur liegt bei den Backend-/Marktdaten-Agenten.
- **`backend/db/`** enthält SQL-Schemadefinitionen und Seed-Logik. Das Backend initialisiert die Datenbank verzögert beim ersten Request — es legt Tabellen an und spielt Standarddaten ein, falls die SQLite-Datei nicht existiert oder leer ist.
- **`db/`** auf oberster Ebene ist der Volume-Mount-Punkt zur Laufzeit. Die SQLite-Datei (`db/finally.db`) wird hier vom Backend erzeugt und übersteht Container-Neustarts dank Docker-Volume.
- **`planning/`** enthält die projektweite Dokumentation, einschließlich dieses Plans. Alle Agenten beziehen sich auf Dateien hier als gemeinsamen Vertrag.
- **`test/`** enthält Playwright-E2E-Tests und die zugehörige Infrastruktur (z. B. `docker-compose.test.yml`). Unit-Tests liegen jeweils in `frontend/` bzw. `backend/`, gemäß den Konventionen des jeweiligen Frameworks.
- **`scripts/`** enthält Start-/Stopp-Skripte, die die Docker-Befehle kapseln.

---

## 5. Umgebungsvariablen

```bash
# Erforderlich: OpenRouter-API-Schlüssel für die LLM-Chat-Funktionalität
OPENROUTER_API_KEY=dein-openrouter-api-schluessel-hier

# Optional: Massive-(Polygon.io-)API-Schlüssel für echte Marktdaten
# Ohne Angabe wird der eingebaute Marktsimulator verwendet (für die meisten Nutzer empfohlen)
MASSIVE_API_KEY=

# Optional: auf "true" setzen für deterministische Mock-LLM-Antworten (Tests)
LLM_MOCK=false
```

### Verhalten

- Ist `MASSIVE_API_KEY` gesetzt und nicht leer → das Backend nutzt die Massive-REST-API für Marktdaten
- Fehlt `MASSIVE_API_KEY` oder ist leer → das Backend nutzt den eingebauten Marktsimulator
- Bei `LLM_MOCK=true` → das Backend liefert deterministische Mock-LLM-Antworten (für E2E-Tests)
- Das Backend liest `.env` aus dem Projekt-Root (in den Container eingebunden oder per `docker --env-file` gelesen)

---

## 6. Marktdaten

### Zwei Implementierungen, eine Schnittstelle

Sowohl der Simulator als auch der Massive-Client implementieren dieselbe abstrakte Schnittstelle. Das Backend wählt anhand der Umgebungsvariable aus, welche verwendet wird. Der gesamte nachgelagerte Code (SSE-Streaming, Kurs-Cache, Frontend) ist unabhängig von der Datenquelle.

### Simulator (Standard)

- Erzeugt Kurse mittels geometrischer brownscher Bewegung (GBM) mit konfigurierbarer Drift und Volatilität je Ticker
- Aktualisiert in Intervallen von ca. 500 ms
- Korrelierte Bewegungen über Ticker hinweg (z. B. bewegen sich Tech-Aktien gemeinsam)
- Gelegentliche zufällige „Ereignisse" — plötzliche Bewegungen von 2–5 % bei einem Ticker, für Dramatik
- Startet von realistischen Ausgangskursen (z. B. AAPL ca. 190 $, GOOGL ca. 175 $ usw.)
- Läuft als In-Process-Hintergrund-Task — keine externen Abhängigkeiten

### Massive-API (optional)

- REST-API-Polling (kein WebSocket) — einfacher, funktioniert in allen Tarifstufen
- Fragt die Vereinigungsmenge aller beobachteten Ticker in einem konfigurierbaren Intervall ab
- Kostenlose Stufe (5 Aufrufe/Minute): Abfrage alle 15 Sekunden
- Bezahlte Stufen: Abfrage alle 2–15 Sekunden je nach Stufe
- Parst die REST-Antwort in dasselbe Format wie der Simulator

### Gemeinsamer Kurs-Cache

- Ein einzelner Hintergrund-Task (Simulator oder Massive-Poller) schreibt in einen In-Memory-Kurs-Cache
- Der Cache hält den letzten Kurs, den vorherigen Kurs und den Zeitstempel für jeden Ticker
- SSE-Streams lesen aus diesem Cache und pushen Aktualisierungen an verbundene Clients
- Diese Architektur unterstützt künftige Mehrbenutzerszenarien ohne Änderungen an der Datenschicht

### SSE-Streaming

- Endpunkt: `GET /api/stream/prices`
- Langlebige SSE-Verbindung; der Client nutzt die native `EventSource`-API
- Der Server pusht Kursaktualisierungen für alle dem System bekannten Ticker in regelmäßiger Taktung (ca. 500 ms) — im Einzelbenutzermodell entspricht das der Watchlist des Nutzers
- Jedes SSE-Ereignis enthält Ticker, Kurs, vorherigen Kurs, Zeitstempel und Änderungsrichtung
- Der Client übernimmt die Wiederverbindung automatisch (`EventSource` bringt einen eingebauten Retry mit)

---

## 7. Datenbank

### SQLite mit verzögerter Initialisierung

Das Backend prüft die SQLite-Datenbank beim Start (oder beim ersten Request). Existiert die Datei nicht oder fehlen Tabellen, legt es das Schema an und spielt Standarddaten ein. Das bedeutet:

- Kein separater Migrationsschritt
- Kein manuelles Datenbank-Setup
- Frische Docker-Volumes starten automatisch mit einer sauberen, vorbefüllten Datenbank

### Schema

Alle Tabellen enthalten eine Spalte `user_id` mit dem Standardwert `"default"`. Dieser ist vorerst fest verdrahtet (Einzelbenutzer), ermöglicht aber künftigen Mehrbenutzerbetrieb ohne Schemamigration.

**users_profile** — Nutzerzustand (Barguthaben)
- `id` TEXT PRIMARY KEY (Standard: `"default"`)
- `cash_balance` REAL (Standard: `10000.0`)
- `created_at` TEXT (ISO-Zeitstempel)

**watchlist** — Ticker, die der Nutzer beobachtet
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (Standard: `"default"`)
- `ticker` TEXT
- `added_at` TEXT (ISO-Zeitstempel)
- UNIQUE-Constraint auf `(user_id, ticker)`

**positions** — Aktuelle Bestände (eine Zeile je Ticker und Nutzer)
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (Standard: `"default"`)
- `ticker` TEXT
- `quantity` REAL (Bruchteile von Aktien werden unterstützt)
- `avg_cost` REAL
- `updated_at` TEXT (ISO-Zeitstempel)
- UNIQUE-Constraint auf `(user_id, ticker)`

**trades** — Handelshistorie (rein anfügendes Protokoll)
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (Standard: `"default"`)
- `ticker` TEXT
- `side` TEXT (`"buy"` oder `"sell"`)
- `quantity` REAL (Bruchteile von Aktien werden unterstützt)
- `price` REAL
- `executed_at` TEXT (ISO-Zeitstempel)

**portfolio_snapshots** — Portfoliowert über die Zeit (für den G&V-Chart). Wird alle 30 Sekunden von einem Hintergrund-Task erfasst sowie unmittelbar nach jeder Trade-Ausführung.
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (Standard: `"default"`)
- `total_value` REAL
- `recorded_at` TEXT (ISO-Zeitstempel)

**chat_messages** — Gesprächsverlauf mit dem LLM
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (Standard: `"default"`)
- `role` TEXT (`"user"` oder `"assistant"`)
- `content` TEXT
- `actions` TEXT (JSON — ausgeführte Trades, Watchlist-Änderungen; null bei Nutzernachrichten)
- `created_at` TEXT (ISO-Zeitstempel)

### Standard-Seed-Daten

- Ein Nutzerprofil: `id="default"`, `cash_balance=10000.0`
- Zehn Watchlist-Einträge: AAPL, GOOGL, MSFT, AMZN, TSLA, NVDA, META, JPM, V, NFLX

---

## 8. API-Endpunkte

### Marktdaten
| Methode | Pfad | Beschreibung |
|--------|------|-------------|
| GET | `/api/stream/prices` | SSE-Stream mit Live-Kursaktualisierungen |

### Portfolio
| Methode | Pfad | Beschreibung |
|--------|------|-------------|
| GET | `/api/portfolio` | Aktuelle Positionen, Barguthaben, Gesamtwert, unrealisierte G&V |
| POST | `/api/portfolio/trade` | Trade ausführen: `{ticker, quantity, side}` |
| GET | `/api/portfolio/history` | Portfoliowert-Snapshots über die Zeit (für den G&V-Chart) |

### Watchlist
| Methode | Pfad | Beschreibung |
|--------|------|-------------|
| GET | `/api/watchlist` | Aktuelle Watchlist-Ticker mit den neuesten Kursen |
| POST | `/api/watchlist` | Ticker hinzufügen: `{ticker}` |
| DELETE | `/api/watchlist/{ticker}` | Ticker entfernen |

### Chat
| Methode | Pfad | Beschreibung |
|--------|------|-------------|
| POST | `/api/chat` | Nachricht senden, vollständige JSON-Antwort erhalten (Nachricht + ausgeführte Aktionen) |

### System
| Methode | Pfad | Beschreibung |
|--------|------|-------------|
| GET | `/api/health` | Health-Check (für Docker/Deployment) |

---

## 9. LLM-Integration

Beim Schreiben von Code für LLM-Aufrufe ist die Skill `cerebras-inference` zu verwenden, um über LiteLLM via OpenRouter das Modell `openrouter/openai/gpt-oss-120b` mit Cerebras als Inferenzanbieter anzusprechen. Zur Interpretation der Ergebnisse sind Structured Outputs zu verwenden.

Im Projekt-Root liegt eine `.env`-Datei mit einem `OPENROUTER_API_KEY`.

### Funktionsweise

Wenn der Nutzer eine Chat-Nachricht sendet, führt das Backend Folgendes aus:

1. Lädt den aktuellen Portfoliokontext des Nutzers (Bargeld, Positionen mit G&V, Watchlist mit Live-Kursen, Portfolio-Gesamtwert)
2. Lädt den jüngsten Gesprächsverlauf aus der Tabelle `chat_messages`
3. Baut einen Prompt aus Systemnachricht, Portfoliokontext, Gesprächsverlauf und der neuen Nachricht des Nutzers
4. Ruft das LLM über LiteLLM → OpenRouter auf, fordert Structured Output an und nutzt dabei die Skill `cerebras-inference`
5. Parst die vollständige strukturierte JSON-Antwort
6. Führt alle in der Antwort angegebenen Trades oder Watchlist-Änderungen automatisch aus
7. Speichert die Nachricht und die ausgeführten Aktionen in `chat_messages`
8. Gibt die vollständige JSON-Antwort an das Frontend zurück (kein Token-für-Token-Streaming — die Cerebras-Inferenz ist schnell genug, sodass ein Ladeindikator genügt)

### Schema für Structured Output

Das LLM wird angewiesen, mit JSON gemäß diesem Schema zu antworten:

```json
{
  "message": "Deine dialogorientierte Antwort an den Nutzer",
  "trades": [
    {"ticker": "AAPL", "side": "buy", "quantity": 10}
  ],
  "watchlist_changes": [
    {"ticker": "PYPL", "action": "add"}
  ]
}
```

- `message` (erforderlich): Der dialogorientierte Text, der dem Nutzer angezeigt wird
- `trades` (optional): Array automatisch auszuführender Trades. Jeder Trade durchläuft dieselbe Validierung wie manuelle Trades (ausreichend Bargeld bei Käufen, ausreichend Aktien bei Verkäufen)
- `watchlist_changes` (optional): Array von Watchlist-Änderungen

### Automatische Ausführung

Vom LLM angegebene Trades werden automatisch ausgeführt — ohne Bestätigungsdialog. Das ist eine bewusste Designentscheidung:

- Es handelt sich um eine simulierte Umgebung mit Spielgeld, es steht also nichts auf dem Spiel
- Es erzeugt ein beeindruckendes, flüssiges Demo-Erlebnis
- Es demonstriert agentische KI-Fähigkeiten — das Kernthema des Kurses

Scheitert ein Trade an der Validierung (z. B. unzureichendes Guthaben), wird der Fehler in die Chat-Antwort aufgenommen, damit das LLM den Nutzer informieren kann.

### Leitlinien für den System-Prompt

Das LLM soll als „FinAlly, ein KI-Trading-Assistent" geprompted werden, mit folgenden Anweisungen:

- Portfoliozusammensetzung, Klumpenrisiko und G&V analysieren
- Trades mit Begründung vorschlagen
- Trades ausführen, wenn der Nutzer darum bittet oder zustimmt
- Die Watchlist proaktiv verwalten
- Knapp und datengetrieben antworten
- Stets mit gültigem strukturiertem JSON antworten

### LLM-Mock-Modus

Bei `LLM_MOCK=true` liefert das Backend deterministische Mock-Antworten, statt OpenRouter aufzurufen. Das ermöglicht:

- Schnelle, kostenlose, reproduzierbare E2E-Tests
- Entwicklung ohne API-Schlüssel
- CI/CD-Pipelines

---

## 10. Frontend-Design

### Layout

Das Frontend ist eine Single-Page-Anwendung mit dichtem, terminal-inspiriertem Layout. Die konkrete Komponentenarchitektur und das Layoutsystem liegen beim Frontend-Engineer, die Oberfläche sollte jedoch folgende Elemente umfassen:

- **Watchlist-Panel** — Raster/Tabelle der beobachteten Ticker mit: Tickersymbol, aktuellem Kurs (grün/rot blinkend bei Änderung), täglicher prozentualer Veränderung und einem Sparkline-Minichart (seit dem Laden der Seite aus SSE akkumuliert)
- **Hauptchart-Bereich** — größerer Chart für den aktuell ausgewählten Ticker, mindestens mit Kursverlauf über die Zeit. Ein Klick auf einen Ticker in der Watchlist wählt ihn hier aus.
- **Portfolio-Heatmap** — Treemap-Visualisierung, bei der jedes Rechteck eine Position darstellt, dimensioniert nach Portfoliogewicht und eingefärbt nach G&V (grün = Gewinn, rot = Verlust)
- **G&V-Chart** — Liniendiagramm des Portfolio-Gesamtwerts über die Zeit, basierend auf Daten aus `portfolio_snapshots`
- **Positionstabelle** — tabellarische Ansicht aller Positionen: Ticker, Stückzahl, Durchschnittskosten, aktueller Kurs, unrealisierte G&V, prozentuale Veränderung
- **Trade-Leiste** — einfacher Eingabebereich: Tickerfeld, Mengenfeld, Kauf-Button, Verkauf-Button. Market-Orders, sofortige Ausführung.
- **KI-Chat-Panel** — angedockte/einklappbare Seitenleiste. Nachrichteneingabe, scrollender Gesprächsverlauf, Ladeindikator während des Wartens auf die LLM-Antwort. Trade-Ausführungen und Watchlist-Änderungen werden inline als Bestätigungen angezeigt.
- **Header** — Portfolio-Gesamtwert (live aktualisiert), Verbindungsstatus-Anzeige, Barguthaben

### Technische Hinweise

- `EventSource` für die SSE-Verbindung zu `/api/stream/prices` verwenden
- Canvas-basierte Charting-Bibliothek bevorzugt (Lightweight Charts oder Recharts) wegen der Performance
- Kurs-Blink-Effekt: Beim Empfang eines neuen Kurses kurz eine CSS-Klasse mit Hintergrundfarb-Transition anwenden und wieder entfernen
- Alle API-Aufrufe gehen an denselben Origin (`/api/*`) — keine CORS-Konfiguration nötig
- Tailwind CSS für das Styling mit einem eigenen dunklen Theme

---

## 11. Docker & Deployment

### Mehrstufiges Dockerfile

```
Stufe 1: Node 20 slim
  - frontend/ kopieren
  - npm install && npm run build (erzeugt statischen Export)

Stufe 2: Python 3.12 slim
  - uv installieren
  - backend/ kopieren
  - uv sync (Python-Abhängigkeiten aus der Lockfile installieren)
  - Frontend-Build-Ergebnis in ein static/-Verzeichnis kopieren
  - Port 8000 freigeben
  - CMD: uvicorn, das die FastAPI-App bereitstellt
```

FastAPI liefert die statischen Frontend-Dateien und alle API-Routen auf Port 8000 aus.

### Docker-Volume

Die SQLite-Datenbank wird über ein benanntes Docker-Volume persistiert:

```bash
docker run -v finally-data:/app/db -p 8000:8000 --env-file .env finally
```

Das Verzeichnis `db/` im Projekt-Root wird auf `/app/db` im Container abgebildet. Das Backend schreibt `finally.db` an diesen Pfad.

### Start-/Stopp-Skripte

**`scripts/start_mac.sh`** (macOS/Linux):
- Baut das Docker-Image, falls noch nicht vorhanden (oder wenn das Flag `--build` übergeben wird)
- Startet den Container mit Volume-Mount, Port-Mapping und `.env`-Datei
- Gibt die URL zum Aufruf der App aus
- Öffnet optional den Browser

**`scripts/stop_mac.sh`** (macOS/Linux):
- Stoppt den laufenden Container und entfernt ihn
- Entfernt NICHT das Volume (Daten bleiben erhalten)

**`scripts/start_windows.ps1`** / **`scripts/stop_windows.ps1`**: PowerShell-Pendants für Windows.

Alle Skripte sollten idempotent sein — mehrfaches Ausführen muss unbedenklich sein.

### Optionales Cloud-Deployment

Der Container ist so ausgelegt, dass er auf AWS App Runner, Render oder einer beliebigen Container-Plattform deploybar ist. Eine Terraform-Konfiguration für App Runner kann als Zusatzziel in einem Verzeichnis `deploy/` bereitgestellt werden, gehört aber nicht zum Kern-Build.

---

## 12. Teststrategie

### Unit-Tests (innerhalb von `frontend/` und `backend/`)

**Backend (pytest)**:
- Marktdaten: Der Simulator erzeugt valide Kurse, die GBM-Mathematik stimmt, das Parsen der Massive-API-Antworten funktioniert, beide Implementierungen erfüllen die abstrakte Schnittstelle
- Portfolio: Logik der Trade-Ausführung, G&V-Berechnungen, Grenzfälle (mehr verkaufen als vorhanden, Kauf bei unzureichendem Guthaben, Verkauf mit Verlust)
- LLM: Das Parsen von Structured Output verarbeitet alle gültigen Schemata, fehlerhafte Antworten werden sauber abgefangen, Trade-Validierung innerhalb des Chat-Flows
- API-Routen: korrekte Statuscodes, Antwortstrukturen, Fehlerbehandlung

**Frontend (React Testing Library o. Ä.)**:
- Komponenten-Rendering mit Mock-Daten
- Die Kurs-Blink-Animation wird bei Kursänderungen korrekt ausgelöst
- CRUD-Operationen auf der Watchlist
- Portfolio-Anzeigeberechnungen
- Rendering von Chat-Nachrichten und Ladezustand

### E2E-Tests (in `test/`)

**Infrastruktur**: Eine separate `docker-compose.test.yml` in `test/`, die den App-Container zusammen mit einem Playwright-Container hochfährt. Damit bleiben Browser-Abhängigkeiten aus dem Produktions-Image heraus.

**Umgebung**: Tests laufen standardmäßig mit `LLM_MOCK=true`, für Geschwindigkeit und Determinismus.

**Zentrale Szenarien**:
- Frischer Start: Die Standard-Watchlist erscheint, ein Guthaben von 10.000 $ wird angezeigt, Kurse werden gestreamt
- Ticker zur Watchlist hinzufügen und wieder entfernen
- Aktien kaufen: Guthaben sinkt, Position erscheint, Portfolio aktualisiert sich
- Aktien verkaufen: Guthaben steigt, Position aktualisiert sich oder verschwindet
- Portfolio-Visualisierung: Die Heatmap rendert mit korrekten Farben, der G&V-Chart hat Datenpunkte
- KI-Chat (gemockt): Nachricht senden, Antwort erhalten, Trade-Ausführung erscheint inline
- SSE-Resilienz: Verbindung trennen und Wiederverbindung prüfen
