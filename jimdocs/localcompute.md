# Bauauftrag für Coding-Agent: Windows-Desktop + Windows-Laptop als lokales Compute-Cluster

**Auftrag:** Implementiere ein lokal betriebenes, ressourcenbewusstes System, das viele unabhängige Claude-Code-Coding-Aufgaben automatisch auf einen Windows-Haupt-PC und einen leistungsfähigen Windows-Laptop verteilt. Alles wird vom Haupt-PC aus bedient. Ziel ist **merklich bessere Desktop-Reaktionsfähigkeit und höherer Gesamtdurchsatz**, nicht das Verschmelzen von CPU oder RAM.

**Kontext:** Haupt-PC: Ryzen 9 (ca. 2020/21), RTX 3080, neun Monitore. Laptop: neuer Ryzen 9, RTX 5090, bislang meist ungenutzt. Ein separater Ubuntu-Server existiert bereits und kann optional Nomad-Server betreiben. SSH/Remote-Zugriff wird bereits verwendet. Beide Windows-Geräte nach Möglichkeit über dasselbe kabelgebundene LAN. Bestehende Systeme/Projekte nicht verändern, bevor die neue Lösung unabhängig funktioniert.

## Leitlinien

1. **Technologie:** Rust für einen schlanken lokalen Windows-Agenten; Go für den zentralen Controller und die Anbindung an HashiCorp Nomad. Nomad übernimmt Job-Lifecycle und Ausführung auf den Nodes; der Go-Controller ergänzt die **Live-Auslastungsentscheidung**. Kein eigenentwickelter allgemeiner Cluster-Scheduler, wenn Nomad-Funktionen reichen.
2. **Einheit der Verteilung:** **Ganze, ausdrücklich freigegebene Jobs bzw. Claude-Code-Sitzungen**; keine transparenten CPU-Befehle, keine gemeinsamen RAM-Seiten, keine automatische Migration bereits laufender Prozesse. Nicht behaupten, beliebige laufende Windows-Prozesse zuverlässig auf Unabhängigkeit prüfen zu können.
3. **Erstes Ergebnis:** Funktionsfähiges MVP für **nichtinteraktive Claude-Code-Aufgaben** (`claude -p`) und allgemeine erlaubte Background-Builds. **Interaktive Claude-Code-Sitzungen** danach separat ergänzen, z. B. mit einer zuverlässigen Remote-PTY/ConPTY-Lösung oder WSL2 + tmux; vor einer Zusage tatsächliche Interaktivität und Wiederaufnahme nach Verbindungsabbruch testen. Nicht nur Remote-Logs als „interaktiv“ ausgeben.
4. **Sicherheit:** Lokales Netz, authentifizierte Zugriffe, keine öffentlich erreichbaren Admin-Endpunkte. Keine Admin-/SYSTEM-Ausführung von Claude-Code-Jobs. Keine Geheimnisse in Logs oder Kommandozeilen. Niemals `--dangerously-skip-permissions` pauschal aktivieren. Überwachung nur eigener/erlaubter Prozesse. Bestehende Installationen nicht löschen oder umbauen.

## Architektur

```text
Haupt-PC: Terminal / CLI / lokale Web-UI
       │ Job anmelden / Logs ansehen / Ergebnis abrufen
       ▼
Go-Controller (Haupt-PC oder optional Ubuntu-Server)
       │ liest aktuelle Metriken, kalkuliert Last und priorisiert Laptop
       │ registriert/dispatcht geeignete Jobs über Nomad HTTP API
       ▼
Nomad-Server (optional Ubuntu-Server)
       ├── Nomad-Client Haupt-PC (Windows)
       │    └── Rust-Agent: CPU, verfügbarer RAM, Job-Status
       └── Nomad-Client Laptop (Windows)
            └── Rust-Agent: CPU, verfügbarer RAM, Job-Status
                 └── isoliertes Projektverzeichnis + Claude Code / Build
```

Die Rust-Agenten liefern ausschließlich Metriken und lokale Job-Hilfsfunktionen; weder Nomad noch der Go-Controller erkennen automatisch eine beliebige Programmanweisung als auslagerbar. Jede ausgelagerte Aufgabe muss als eigenständig definierter Job vorliegen.

## Arbeitspakete — in dieser Reihenfolge

### Phase 0: Inventur und Machbarkeitsnachweis
- Repo `local-compute-fabric` neu anlegen; bestehende AWG-, Projekt-, Claude- und Server-Konfiguration nicht ungefragt modifizieren.
- Betriebssystem-Versionen, RAM, CPU, Netzwerkgeschwindigkeit, verfügbare SSD, Windows-Benutzer, `claude --version`, Authentifizierung, Git, Rust/Go, Nomad und ggf. WSL2 **lesend prüfen**.
- Vor der Installation eine minimale PoC-Demo zeigen: Nomad-Server, beide Windows-Clients und ein auf dem **Laptop** gestarteter Dummy-Job, der seine Host-ID protokolliert. Wenn die Nomad-Installation am Ubuntu-Server bestehende Services gefährdet, zunächst separate lokale Testumgebung benutzen.
- Dokumentiere Windows-Treiberrestriktionen: Nomad-`exec` funktioniert nur auf Linux. Nomad-`raw_exec` läuft unter Windows, ist standardmäßig deaktiviert und bietet **keine Dateisystem-Isolation**. Für Tests nur freigegebene Jobs unter dediziertem unprivilegiertem Benutzer ausführen; alternativ WSL2-Linux-Nomad-Client mit passenden Isolationsoptionen evaluieren. Vor produktivem Aktivieren von `raw_exec` die Risiken dokumentieren.

### Phase 1: Rust-Node-Agent
- Baue Windows-Binary mit strukturierten Host-Metriken: CPU-Auslastung (Zeitfenster 10/30/60 s), gesamter und tatsächlich verfügbarer RAM, Prozessbaum/Prozess-RAM **der selbst gestarteten Jobs**, freier Speicherplatz, Erreichbarkeit, laufende Jobzahl und Heartbeat.
- Datenerfassung alle 2 Sekunden, gleitende Glättung; `GET /health` und authentifizierte `GET /metrics` im LAN oder Push zum Controller. Datenerfassung leichtgewichtig und mit eigenem Overhead-Monitoring.
- Starte und beende nur vom Controller/Nomad zugelassene Jobs; sichere PID-/Kindprozess-Verwaltung. Keine fremden Prozesse automatisch beenden. Bei fehlendem Heartbeat gilt der Node als nicht auswählbar.

### Phase 2: Go-Controller + Nomad
- Kommuniziere über dokumentierte Nomad-HTTP-API (oder offiziell gepflegten Go-Client). Unterstütze `submit`, `list`, `status`, `logs`, `cancel` und `dry-run`.
- Lege vorab Job-Typen fest: `claude-batch`, `build`, `test`. Jeder Job enthält `job_id`, `repo`, `revision`, `worktree`, erlaubten Befehl, geschätztes CPU-/RAM-Budget, Timeout, erlaubte Nodes, Rückgabeort und Priorität.
- Schätze Job-Ressourcen anhand historischer Messungen oder konservativer Defaults. Korrigiere anhand gemessener Peaks. **Reservierter** RAM für Nomad ist nicht dasselbe wie **tatsächlich freier** RAM; nutze beide Angaben.
- Auswahlalgorithmus (alles konfigurierbar): Laptop bevorzugen; Node nur zulassen, wenn online, korrekte Runtime/Repo-Daten vorhanden, erwartetes RAM-Budget plus Sicherheitspuffer passt und CPU-Last unter dem Grenzwert liegt. Haupt-PC mit strengerem Schutz reservieren (z. B. mindestens 8 GiB RAM frei und 25–30 % CPU-Headroom). Keine Überbuchung bei unklaren Messwerten; Job sonst **in Warteschlange**.
- Nutze Nomad-Node-Constraints, um den vom Controller gewählten Node tatsächlich festzulegen, statt darauf zu vertrauen, dass Nomad live gemessene CPU automatisch berücksichtigt. Für Batch-Aufgaben können parameterisierte Nomad-Jobs verwendet werden; kläre, wann eine individuelle Jobregistrierung besser passt.
- Neue Jobs werden intelligent platziert. **Laufende Jobs werden bei CPU-Spitzen nicht automatisch verschoben**, um Datenverlust und Neuanlauf zu vermeiden. Cooldown/Hysterese gegen ständiges Umschalten.
- Fehlerroutinen: Laptop offline → vorhandene Sessions nicht gefährden; neue Jobs nur nach Freigabe auf Haupt-PC oder Warteschlange; Jobs idempotent starten; bei Neustarts keine Doppel-Ausführung oder stilles Verlieren ungespeicherter Arbeit.

### Phase 3: Dateien, Zustände und Credentials
- **Pro Job eigener Git-Klon oder eigener Worktree** am ausführenden Node. Repo bei Start auf die exakte Revision bringen; Änderungen in gesondertem Branch oder Patch zurückgeben. Kein gleichzeitiges Schreiben mehrerer Agenten in dasselbe Projektverzeichnis; keine automatische Überschreibung laufender Änderungen.
- Erst Git + lokale SSD als Datenstrategie. Nicht für jeden Dateizugriff große Daten über SMB hin- und herschicken. Falls Worktree-Synchronisierung nötig ist, nur differenzielle Daten kopieren und Konflikte erkennen.
- Claude Code auf dem **ausführenden** Rechner unter dem dafür konfigurierten Nutzer installieren und dort ordnungsgemäß authentifizieren; keine ungesicherte Credential-Kopie vom Haupt-PC. API-/Account-Limits getrennt beachten. Interaktive Sitzungsdaten müssen auf dem Ausführungsrechner persistent bleiben, wenn die Sitzung fortgesetzt werden soll.
- Pro Job stdout/stderr, Exit-Code, Start/Ende, Commit/Patch, Ressourcenspitzen und Logs zentral abrufbar machen. Sensible Daten redigieren.

### Phase 4: Bedienung vom Haupt-PC
- Starte mit einer schlanken CLI und optional einer lokalen Web-UI: `nodes`, `submit`, `jobs`, `logs`, `cancel`, `dry-run`, `set-policy`.
- Zeige pro Node CPU, freien RAM, laufende Jobs und den **konkreten Grund**, warum ein Job dort platziert wurde.
- Manuelle Überschreibung `--node laptop|desktop|auto`; globale Notbremse `pause-scheduling`; Stoppen nur der von diesem System verwalteten Jobs.
- Interaktive Claude-Code-Nutzung nach MVP gesondert realisieren, inklusive echter Tastatureingabe, Terminal-Ausgabe, Reconnect und dauerhaft gespeicherter Sitzung. Bei nicht gelöster PTY-Frage ehrliche technische Grenze dokumentieren und einen SSH-/WSL2-tmux-Fallback für interaktive Sessions liefern.

### Phase 5: Messung und Qualität
- Baseline vor/nach Lastverschiebung unter gleichen Bedingungen: (a) p50/p95 Zeit zum Öffnen eines Browser-Tabs und allgemeine Desktop-Latenz, (b) freie RAM-Kapazität und CPU-Auslastung am Haupt-PC, (c) gesamte Jobs pro Stunde, (d) Zeit vom Submit bis zum Jobstart, (e) End-to-End-Dauer und LAN-/Datei-Sync-Kosten. Keine Geschwindigkeitsgewinne behaupten, bevor gemessen wurde.
- Automatisierte Tests: Auswahl anhand realer Metriken, RAM-Grenzen, konkurrierende Jobs, Laptop-Ausfall, Controller-Neustart, Auth-Fehler, Konflikt bei Git-Änderungen, Doppelstarts, Cancellation und Logs.
- 5 parallele **Dummy-/Build-Jobs** auf dem Laptop testen; erst danach echte Claude-Code-Jobs in kleinen Zahlen. Keine kostenpflichtigen oder externen KI-Aufrufe in automatisierten Tests ohne explizites Budget.

## Definition of Done (MVP)

1. Auf Haupt-PC und Laptop laufen unprivilegierte Rust-Metrikagenten; Controller zeigt CPU/RAM live an.
2. Ein Job kann vom Haupt-PC eingereicht werden, wird aufgrund gemessener Ressourcen automatisch **auf dem Laptop gestartet** und endet mit abholbarem Ergebnis.
3. Läuft der Laptop voll, wird ein neuer Job korrekt eingereiht oder gemäß Policy auf den Desktop gelegt; der Desktop bleibt durch konfigurierbare Ressourcenreserven geschützt.
4. Jobs nutzen getrennte Verzeichnisse, sind überwachbar und abbrechbar, und laufen nicht versehentlich doppelt.
5. Ein Benchmarkbericht belegt oder widerlegt, ob sich die Benutzeroberfläche des Haupt-PCs und der Gesamtdurchsatz tatsächlich verbessern.
6. README liefert Windows- und optional Ubuntu-Schritte, sichere Konfiguration, Start-/Stop-Kommandos, Tests, Rückbau und bekannte Grenzen.

## Lieferformat

Liefere nach der Inventur **nicht nur eine Architekturzeichnung**, sondern: vollständigen Rust-/Go-Quellcode, reproduzierbare Builds, Beispielkonfiguration für beide Windows-Rechner und optional den Ubuntu-Nomad-Server, Nomad-Jobdefinitionen, ein Installationsskript nur für die neue Testumgebung, Tests sowie eine verständliche Bedienungsanleitung. Implementiere in kleinen Phasen, überprüfe die tatsächliche Ausführung jedes Meilensteins und berichte nach jeder Phase über Ergebnis und Blocker. Keine unbelegten Aussagen über Unterstützung, Isolation oder gemessene Performance.

## Verifizierte Referenzen

- Nomad Windows / `raw_exec`: https://developer.hashicorp.com/nomad/docs/deploy/task-driver/raw_exec
- Nomad `exec` nur Linux: https://developer.hashicorp.com/nomad/docs/deploy/task-driver/exec
- Nomad Live-Stats API: https://developer.hashicorp.com/nomad/api-docs/client
- Nomad Job-Dispatch / Register API: https://developer.hashicorp.com/nomad/api-docs/jobs
- Nomad Node-Constraints: https://developer.hashicorp.com/nomad/docs/job-specification/constraint
- Nomad parameterisierte Jobs: https://developer.hashicorp.com/nomad/docs/job-specification/parameterized
- Claude Code Windows & nichtinteraktiver Betrieb: https://support.claude.com/en/articles/14554922-claude-code-user-faq

**Priorität:** Erst end-to-end funktionsfähige Lastverteilung und messbare Entlastung des Haupt-PCs, dann Komfort, UI und ambitionierte automatische Prozessklassifikation. Zielgröße: einzelne Person mit typischerweise wenigen bis einigen Dutzend gleichzeitigen Sessions, kein hyperskalierender Cloud-Cluster.
