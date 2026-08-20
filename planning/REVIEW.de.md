# PLAN.md — Dokumentationsreview

*Review von `planning/PLAN.md`. Nichts im Folgenden ist eine Änderung an der Spezifikation — es sind offene Fragen und Empfehlungen, die der Autor des Plans annehmen oder ablehnen kann. Zu jedem Punkt ist eine Empfehlung angegeben, damit er per Entscheidung statt per Diskussion aufgelöst werden kann.*

**Zustand des Repositories zum Zeitpunkt des Reviews:** Grüne Wiese. `backend/`, `frontend/`, `test/` und `db/` sind allesamt leer; `PLAN.md` ist der einzige substanzielle Inhalt. Dieses Review bewertet den Plan daher als **Vertrag, gegen den parallel arbeitende Agenten bauen**, nicht als Beschreibung existierenden Codes.

Der Plan ist stark bei Vision, Technologiebegründung und Verzeichnisabgrenzung. Die Lücken liegen fast alle bei den **Verträgen zwischen den Agenten**: an Stellen, an denen der Frontend- und der Backend-Agent jeweils eine plausible, aber inkompatible Annahme treffen würden. Diese stehen zuerst, weil sie parallele Arbeit blockieren.

---

## 1. Vertragslücken, die parallele Agenten blockieren

**F1. Die Struktur der SSE-Ereignisse ist nicht spezifiziert.** §6 sagt, „jedes SSE-Ereignis enthält Ticker, Kurs, vorherigen Kurs, Zeitstempel und Änderungsrichtung", aber nicht, ob es ein Ereignis pro Ticker oder ein gebündeltes Ereignis für alle Ticker pro Takt ist, auch nicht den SSE-`event:`-Namen oder die exakten JSON-Schlüssel. Zwei Agenten werden unterschiedlich raten, und die Integration wird scheitern.
*Empfehlung:* ein gebündeltes Ereignis pro Takt festschreiben — weniger Nachrichten, ein Render-Durchlauf im Client, und es macht selbstbeschreibend, welche Ticker existieren:
```
event: prices
data: {"ts":"2026-08-11T14:00:00.000Z","prices":[{"ticker":"AAPL","price":190.12,"prev":190.05,"dir":"up"}]}
```

**F2. Es gibt keinen Endpunkt zum Lesen des Chatverlaufs.** `chat_messages` wird mit einer `actions`-Spalte persistiert, §8 stellt aber nur `POST /api/chat` bereit. Beim Neuladen der Seite verschwindet die Konversation aus der Oberfläche, obwohl sie weiterhin in der Datenbank liegt.
*Empfehlung:* `GET /api/chat` ergänzen, das die jüngsten Nachrichten liefert — oder die Tabelle streichen und die Konversation ausschließlich im Frontend-State halten. Ersteres ist ein zweizeiliger Endpunkt und passt zur ohnehin spezifizierten Persistenz; daher vorzuziehen.

**F3. Die Struktur des `actions`-JSON ist undefiniert.** Das Frontend rendert „Trade-Ausführungen und Watchlist-Änderungen inline als Bestätigungen" (§10), die Struktur ist also ein harter Vertrag — und sie muss *Ergebnisse* transportieren (einschließlich Fehlschlägen, siehe F17), nicht bloß die vom LLM angeforderten Aktionen.
*Empfehlung:* im Plan festlegen, z. B. `[{"type":"trade","ticker":"AAPL","side":"buy","quantity":10,"price":190.12,"status":"ok"}, {"type":"trade",...,"status":"error","error":"insufficient cash"}]`.

**F4. Der Abschnitt zum Docker-Volume widerspricht dem Abschnitt zur Verzeichnisstruktur.** §4 sagt, das oberste `db/` sei „der Volume-Mount-Punkt zur Laufzeit", mit einer `.gitkeep`; §11 zeigt `docker run -v finally-data:/app/db`, ein *benanntes* Volume, das das Host-Verzeichnis `db/` überhaupt nicht berührt. Nur eines von beiden kann zutreffen.
*Empfehlung:* einen Bind-Mount `-v "$(pwd)/db:/app/db"` verwenden. Das entspricht §4, und für einen Kurs ist es viel wert, dass Studierende `db/finally.db` direkt sehen, inspizieren und löschen können.

**F5. Die E2E-Tests sind in der beschriebenen Form nicht wiederholbar.** Jedes aufgeführte Szenario verändert persistenten Zustand (kauft Aktien, fügt Ticker hinzu). Ein zweiter Lauf startet mit abweichendem Guthaben und abweichender Watchlist, sodass Zusicherungen wie „ein Guthaben von 10.000 $ wird angezeigt" fehlschlagen.
*Empfehlung:* `docker-compose.test.yml` ein anonymes bzw. tmpfs-Volume auf `/app/db` einhängen lassen, sodass jeder Lauf eine frische, verzögert initialisierte Datenbank erhält. Das ist sauberer als ein Reset-Endpunkt und übt bei jedem Testlauf zugleich den Lazy-Init-Pfad.

**F6. Die Antwortstruktur von `POST /api/portfolio/trade` ist unspezifiziert.** Die Oberfläche benötigt den tatsächlichen Ausführungskurs (der Cache kann seit der angezeigten Kursanzeige weitergetickt sein) sowie das resultierende Guthaben bzw. die Position.
*Empfehlung:* den ausgeführten Trade zusammen mit dem aktualisierten Portfolio zurückgeben, damit der Client aus einer einzigen Antwort neu rendern kann, statt ein nachgelagertes `GET /api/portfolio` abzusetzen.

---

## 2. Marktdaten

**F7. Woher bezieht der *Hauptchart* seine Historie?** §2 ist explizit, dass Sparklines clientseitig seit dem Laden der Seite aus SSE akkumuliert werden — der „größere Chart für den aktuell ausgewählten Ticker" aus §10 hat jedoch keine angegebene Datenquelle, und es gibt weder einen History-Endpunkt noch eine Kurstabelle. In der vorliegenden Fassung zeigt ein Klick auf einen Ticker einen leeren Chart, der sich erst über die folgenden Minuten füllt — ein schwacher erster Eindruck für das visuelle Aushängeschild.
*Empfehlung:* einen serverseitigen Ringpuffer vorhalten (etwa die letzten 500 Ticks je beobachtetem Ticker, im Speicher neben dem Kurs-Cache) und `GET /api/history/{ticker}` bereitstellen. Das kostet sehr wenig und sorgt dafür, dass der Chart im Moment des Öffnens gefüllt ist. Optional den Puffer beim Start mit einer synthetischen Rückschau vorbefüllen, damit der Chart nie leer ist.

**F8. Die „tägliche prozentuale Veränderung" ist aus dem spezifizierten Cache nicht berechenbar.** §10 verlangt für die Watchlist die Anzeige der täglichen prozentualen Veränderung, der Cache hält aber nur den letzten Kurs, den vorherigen Kurs (der letzte ~500-ms-Tick) und den Zeitstempel. Die Veränderung gegenüber dem vorherigen Tick liegt im Bereich von Bruchteilen eines Prozents — visuell bedeutungslos.
*Empfehlung:* einen **Sitzungs-Referenzkurs** je Ticker definieren (den Ausgangskurs des Simulators bzw. im Massive-Modus den ersten nach dem Start beobachteten Kurs), im Cache ablegen und die prozentuale Veränderung dagegen berechnen. Falls „täglich" irreführend ist, die Bezeichnung „Sitzungsveränderung" verwenden.

**F9. Was passiert, wenn ein unbekannter Ticker hinzugefügt wird?** Im Simulatormodus gibt es kein Symboluniversum — `POST /api/watchlist {"ticker":"ZZZZ"}` hat weder Ausgangskurs noch Drift oder Volatilität. Auch das LLM kann Ticker hinzufügen, über denselben Pfad.
*Empfehlung:* die Regel explizit festhalten. Am einfachsten und dennoch beeindruckend: eine kleine Tabelle mit ca. 30 bekannten Symbolen und realistischen Ausgangskursen; jedes unbekannte Symbol erhält einen generierten Ausgangskurs (z. B. gleichverteilt 20–400 $) mit Standardwerten für Drift und Volatilität. Ebenso festlegen: Normalisierung (Großschreibung, Trimmen), maximale Watchlist-Größe und die Antwort beim doppelten Hinzufügen (409 vs. idempotentes 200).

**F10. Welche Ticker verfolgt der Hintergrund-Task tatsächlich?** §6 sagt „alle dem System bekannten Ticker … entspricht der Watchlist des Nutzers". Eine Position in einem Ticker, den der Nutzer inzwischen aus der Watchlist entfernt hat, benötigt aber weiterhin einen Live-Kurs, sonst friert deren G&V stillschweigend ein.
*Empfehlung:* die verfolgte Menge als `Vereinigung(Watchlist, Ticker mit offenen Positionen)` definieren und das Entfernen eines gehaltenen Tickers aus der Watchlist zulassen (eine Blockade wäre überraschend).

**F11. Massives Abfrageintervall und die SSE-Taktung passen nicht zusammen.** Die kostenlose Stufe fragt alle 15 s ab, während SSE alle 500 ms pusht — derselbe Kurs wird also ca. 30-mal erneut gesendet, und der Client darf nicht bei jedem Mal blinken. Auch der Massive-Request/Response-Vertrag (Basis-URL, Auth-Header, Endpunkt, JSON-Struktur) ist vollständig unspezifiziert — der Agent müsste raten oder ihn erfinden.
*Empfehlung:* (a) nur Ticker pushen, deren Kurs sich tatsächlich geändert hat, zuzüglich eines SSE-Kommentar-Heartbeats alle ca. 15 s, um die Verbindung durch Proxys offen zu halten; (b) ein kurzes `planning/MARKET_DATA.md` mit dem realen Massive-Request/Response-Vertrag ergänzen, bevor der Marktdaten-Agent startet.

---

## 3. Portfolio & Handel

**F12. Die Buchhaltungsregeln sind impliziert, aber nirgends ausformuliert.** Konkret: Ist die Kostenbasis der gewichtete Durchschnitt (die Spalte `avg_cost` legt das nahe)? Bleibt `avg_cost` bei einem Verkauf unverändert? Wird realisierte G&V irgendwo erfasst (es existiert keine Spalte dafür)? Wird eine Positionszeile bei Stückzahl 0 gelöscht oder als Nullzeile behalten? §2 sagt „Position aktualisiert sich oder verschwindet", was auf Löschen hindeutet.
*Empfehlung:* klar festhalten: gewichtete Durchschnittskosten, `avg_cost` bei Verkäufen unangetastet, keine Erfassung realisierter G&V und keine Steuerlose (Tax Lots), Positionszeile wird gelöscht, sobald die Stückzahl 0 erreicht.

**F13. Fließkomma-Geldbeträge werden driften.** Guthaben und Kurse sind `REAL`. Eine Folge von Bruchteil-Trades kann das Guthaben bei `-0,0000001` hinterlassen, und eine naive Prüfung `cash >= cost` würde einen legitimen Ablauf „alles verkaufen / mit vollem Guthaben kaufen" ablehnen.
*Empfehlung:* Guthaben und Kosten bei jedem Schreibvorgang auf zwei Nachkommastellen runden und mit einem kleinen Epsilon vergleichen. Ein Satz im Plan verhindert eine ganze Klasse verwirrender Fehler.

**F14. Die Regeln zur Trade-Validierung sind nicht aufgeführt.** Die Menge muss > 0 sein; werden gebrochene Mengen über die Trade-Leiste akzeptiert (das Schema unterstützt sie)? Was wird zurückgegeben, wenn für den Ticker noch kein Kurs im Cache liegt? Was, wenn der Ticker nicht in der Watchlist steht — ist der Kauf eines nicht beobachteten Tickers erlaubt?
*Empfehlung:* die Ablehnungsfälle und ihre Statuscodes (400 mit Meldung) einmal auflisten, damit manuelle und LLM-Trades nachweislich denselben Codepfad teilen — was §9 bereits zusichert.

**F15. `GET /api/portfolio/history` hat keine Parameter.** Snapshots wachsen um 2.880 Zeilen pro Tag, und die Antwort wächst unbegrenzt.
*Empfehlung:* `?limit=` akzeptieren (Standard z. B. 500) und die neuesten Einträge zuletzt zurückgeben. Aufbewahrungsfristen oder Downsampling sind in Kursgröße unnötig, das Limit lohnt sich aber.

**F16. Der G&V-Chart ist in den ersten 30 Sekunden leer und bis zum ersten Trade flach.** Das sollte anerkannt werden.
*Empfehlung:* unmittelbar beim Start einen Snapshot schreiben (zusätzlich zu alle 30 s und nach jedem Trade), damit der Chart stets mindestens einen Punkt hat — und die Darstellung des Leerzustands festlegen.

---

## 4. Chat & LLM

**F17. Die Reihenfolge der automatischen Ausführung widerspricht der zugesicherten Fehlerbehandlung.** §9 Schritt 4 erzeugt `message`, Schritt 6 führt dann die Trades aus, und Schritt 8 gibt die Antwort zurück — der Text wurde also *vor* der Ausführung verfasst. Scheitert ein Trade, kann die Nachricht des Assistenten fröhlich einen Trade bestätigen, der nie stattgefunden hat. „Der Fehler wird in die Chat-Antwort aufgenommen, damit das LLM den Nutzer informieren kann" ist ohne einen zweiten LLM-Aufruf, den der Ablauf nicht vorsieht, schlicht nicht möglich.
*Empfehlung:* keinen zweiten Aufruf hinzufügen. Die Ergebnisse stattdessen deterministisch in der Oberfläche aus dem `actions`-Array (F3) rendern — ein roter Inline-Chip mit `✗ BUY 10 AAPL — unzureichendes Guthaben` neben der Nachricht des Assistenten. Günstiger, schneller und nicht halluzinierbar. Der Plan sollte das explizit festhalten.

**F18. „Jüngster Gesprächsverlauf" ist nicht quantifiziert.** Unbegrenztes Wachstum sprengt irgendwann das Kontextfenster.
*Empfehlung:* auf die letzten N Nachrichten festlegen (N ≈ 20) und das auch so dokumentieren.

**F19. Das Verhalten des Mock-Modus ist undefiniert, aber die E2E-Tests hängen davon ab.** §12 verlangt, dass der gemockte Chat-Test „Trade-Ausführung erscheint inline" zusichert — der Mock muss also eine Trade-Aktion ausgeben können, während §9 lediglich von „deterministischen Mock-Antworten" spricht.
*Empfehlung:* den Mock-Vertrag im Plan spezifizieren, z. B.: eine Nachricht, die auf `/(buy|sell) (\d+) ([A-Z]+)/i` passt, liefert eine vorgefertigte Nachricht plus diesen Trade; alles andere liefert eine feste Portfolio-Zusammenfassung ohne Aktionen. Das ist ein Vertrag zwischen Backend- und E2E-Agent und gehört schriftlich festgehalten.

**F20. Was passiert, wenn `OPENROUTER_API_KEY` fehlt oder der LLM-Aufruf scheitert?** §5 bezeichnet ihn als „erforderlich", dabei funktioniert alles außer dem Chat auch ohne ihn.
*Empfehlung:* sanft degradieren — die App startet, der Chat liefert eine freundliche Fehlermeldung in der Nachrichtenblase statt eines 500ers, und die Startskripte warnen beim Start.

---

## 5. Frontend & Entwicklungsablauf

**F21. Es gibt keinen Entwicklungsablauf.** Der Plan beschreibt ausschließlich das produktive Ein-Container-Setup. Agenten, die am Frontend iterieren, werden nicht pro Änderung ein Docker-Image neu bauen; sie brauchen `next dev` auf :3000 im Zusammenspiel mit `uvicorn` auf :8000 — womit genau das Cross-Origin-Problem zurückkehrt, das die Architektur vermeiden sollte.
*Empfehlung:* einen Dev-Modus dokumentieren, der Next.js-`rewrites` nutzt, um `/api/*` an `localhost:8000` weiterzuleiten. Gleicher Origin in der Entwicklung, keine CORS-Konfiguration, keine Codeunterschiede zwischen Dev und Prod. Das ist die wertvollste Ergänzung für die Entwicklungsgeschwindigkeit.

**F22. Es werden zwei Charting-Bibliotheken angeboten („Lightweight Charts oder Recharts"), die nicht austauschbar sind** — und weder Heatmap/Treemap noch Sparklines sind einer davon zugeordnet.
*Empfehlung:* Recharts für alles wählen. Es liefert eine `Treemap`-Komponente mit (Lightweight Charts nicht), deckt Liniendiagramme und Sparklines ab, und eine Bibliothek schlägt zwei. Falls die Canvas-Performance für den Hauptchart wichtiger ist, Lightweight Charts wählen *und* einen separaten Ansatz für die Treemap benennen.

**F23. Füllt ein Klick auf einen Watchlist-Ticker auch die Trade-Leiste?** Unspezifiziert — und es macht den Unterschied zwischen einer flüssigen und einer holprigen Demo aus.
*Empfehlung:* ja — ein einziger „ausgewählter Ticker"-State steuert sowohl den Hauptchart als auch das Tickerfeld der Trade-Leiste.

**F24. Leerzustände sind unspezifiziert** für Positionstabelle und Heatmap beim frischen Start — also genau das, was ein Erstnutzer sieht.
*Empfehlung:* je eine Zeile, die die Darstellung ohne Positionen beschreibt.

**F25. Wie erzwingt der E2E-Test zur „SSE-Resilienz" einen Verbindungsabbruch?** Playwright kann eine laufende EventSource nicht ohne Weiteres beenden.
*Empfehlung:* `page.route('**/api/stream/prices', r => r.abort())` verwenden, um sie zu unterbrechen, danach die Route wieder aufheben und prüfen, dass der Statuspunkt zu Grün zurückkehrt — und diese Technik im Plan benennen, damit der Test-Agent nicht improvisiert.

---

## 6. Infrastruktur & Betrieb

**F26. Die SQLite-Nebenläufigkeit wird nicht behandelt.** Ein Snapshot-Task im 30-Sekunden-Takt schreibt gleichzeitig mit den Request-Handlern; mit den Standardeinstellungen erzeugt das sporadische `database is locked`-Fehler, deren Diagnose mühsam ist.
*Empfehlung:* eine Zeile — den WAL-Modus aktivieren und `busy_timeout` beim Verbindungsaufbau setzen.

**F27. Zeitstempel sind `TEXT` ohne festgelegtes Format.** Die Sortierung von `portfolio_snapshots` und `chat_messages` beruht auf lexikographischem Vergleich, was nur funktioniert, wenn jeder Schreiber dasselbe UTC-Format verwendet.
*Empfehlung:* durchgängig ISO-8601 in UTC mit `Z`-Suffix und Millisekundengenauigkeit vorschreiben.

**F28. `docker run -p 8000:8000` bindet an alle Netzwerkschnittstellen** und stellt damit eine nicht authentifizierte Anwendung, die einen API-Schlüssel hält und Trades ausführt, dem lokalen Netzwerk bereit.
*Empfehlung:* in den Startskripten an `-p 127.0.0.1:8000:8000` binden. Für den lokalen Einsatz ohne jeden Nachteil.

**F29. `.env` wird von `--env-file` benötigt, ist aber gitignored,** sodass ein frischer Clone beim ersten Start mit einem Docker-Fehler scheitert statt mit einer hilfreichen Meldung.
*Empfehlung:* die Startskripte sollen `.env.example` → `.env` kopieren, falls die Datei fehlt, und ausgeben, was einzutragen ist. Außerdem laviert §5 zwischen dem Einhängen von `.env` und der Nutzung von `--env-file` — hier ausschließlich `--env-file` wählen.

**F30. Der In-Memory-Kurs-Cache bedeutet, dass ein Container-Neustart die simulierten Kurse auf die Ausgangswerte zurücksetzt,** was einen sichtbaren Sprung in der G&V und eine Unstetigkeit im Snapshot-Chart erzeugt, während `avg_cost` erhalten bleibt.
*Empfehlung:* das akzeptieren und in einer Zeile festhalten — oder die letzten Kurse beim Herunterfahren persistieren. Akzeptieren ist völlig in Ordnung, sollte aber eine dokumentierte Entscheidung sein, damit ein Agent das nicht ungefragt „repariert".

---

## 7. Vereinfachungsmöglichkeiten

**V1. Die beiden 500-ms-Timer zusammenlegen.** §6 beschreibt, dass der Simulator den Cache alle ca. 500 ms aktualisiert *und* SSE alle ca. 500 ms pusht — zwei unabhängige Schleifen, die auseinanderdriften und doppelte oder ausgelassene Ticks erzeugen können. Stattdessen eine Schleife: Modell takten, dann Ergebnis broadcasten. Weniger bewegliche Teile, kein Drift.

**V2. Die Sektor-Taxonomie durch einen einzelnen Marktfaktor ersetzen.** „Korrelierte Bewegungen über Ticker hinweg (z. B. bewegen sich Tech-Aktien gemeinsam)" setzt eine gepflegte Sektorzuordnung voraus — die ohnehin keine beliebig hinzugefügten Ticker abdecken kann. Ein Einfaktormodell (`Rendite = Beta × Marktfaktor + idiosynkratischer Term`, Beta standardmäßig 1,0) erzeugt optisch überzeugende Korrelation mit einem Bruchteil des Codes und behandelt unbekannte Symbole kostenlos mit.

**V3. Nur geänderte Kurse pushen.** In Kombination mit V1 entfällt damit die Prüfung „hat sich überhaupt etwas geändert?" im Client, die Blink-Logik wird trivial korrekt, und der Datenverkehr im Massive-Modus sinkt um etwa das 30-Fache.

**V4. Prüfen, ob Next.js seinen Platz verdient.** Die Anwendung ist eine einzige Seite, ohne Routing, ohne SSR, ohne Server-Komponenten, statisch exportiert und von FastAPI ausgeliefert — jedes Next.js-Feature ist abgeschaltet. Vite + React würde schneller bauen und hätte eine einfachere Konfiguration. *Allerdings:* Wenn der Kurs Next.js vermitteln soll, wiegt dieser didaktische Grund schwerer als die Vereinfachung — dann sollte das schlicht als bewusste Entscheidung festgehalten und nicht implizit gelassen werden.

**V5. Eine Chart-Bibliothek statt zwei.** Siehe F22.

**V6. `users_profile.created_at` streichen.** Die Spalte wird einmal geschrieben und nie gelesen. Eine Kleinigkeit — aber das Schema ist ein Vertrag, gegen den mehrere Agenten implementieren, und ungenutzte Spalten laden zu spekulativem Code ein.

**V7. Den „vorherigen Kurs" aus der SSE-Nutzlast streichen,** sofern `dir` ohnehin mitgesendet wird. Der Client kennt den vorherigen Kurs per Definition — er hat ihn gerendert. Drei Repräsentationen derselben Tatsache zu senden (`prev`, `dir` und den impliziten Vorzustand) schafft drei Möglichkeiten, sich zu widersprechen.

---

## 8. Kleinere Anmerkungen

- Der Verzeichnisbaum in §4 lässt `.env.example` aus, obwohl §5 und §11 beide davon abhängen.
- §8 führt nirgends Fehlerantworten auf. Ein einziger Satz — „alle Fehler liefern `{"error": "..."}` mit 4xx" — deckt die gesamte API ab.
- `GET /api/health` hat keinen spezifizierten Antwortkörper; falls er einen Docker-`HEALTHCHECK` speist, sollte das dort stehen.
- §2 sichert „kein Bestätigungsdialog" für Trades zu und §9 wiederholt das für LLM-Trades — konsistent, es lohnt aber, zusätzlich festzuhalten, dass es kein Rückgängigmachen gibt.
- CLAUDE.md sagt „Die Agenten interagieren über Dateien in `planning/`", doch `planning/` enthält nur PLAN.md und keinerlei Konvention dazu, was sonst dorthin gehört. Angesichts der obigen Lücken sind die naheliegenden Begleitdokumente `API_CONTRACT.md` (exakte Request/Response- und SSE-JSON-Strukturen), `MARKET_DATA.md` (Massive-Vertrag + Simulatorparameter) sowie ein fortlaufendes `PROGRESS.md`. Sie in PLAN.md zu benennen macht aus „Agenten koordinieren sich über Dateien" etwas, mit dem Agenten tatsächlich arbeiten können.

---

## 9. Vorgeschlagene Reihenfolge der Klärung

1. **F1, F3, F6** — die JSON-Verträge einfrieren; alles Weitere hängt davon ab.
2. **F4, F21** — den Volume-Widerspruch auflösen und den Dev-Ablauf definieren, sonst zahlt jeder Agent den Rebuild-Aufschlag.
3. **F7, F8** — über Historie und prozentuale Veränderung entscheiden; beides verändert das Datenmodell des Backends.
4. **F17, F19** — Berichterstattung über Chat-Aktionen und den Mock-Vertrag klären, bevor LLM- und E2E-Agent auseinanderlaufen.
5. Alles Übrige lässt sich im Verlauf des Builds nebenbei klären.
