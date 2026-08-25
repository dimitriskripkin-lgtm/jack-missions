# JACK ROADMAP — Haertung, Bugfixes, Optimierung
Erstellt: 2026-08-25 von Claude nach Vollanalyse (lokaler Projekt-Snapshot + GitHub live)
Ziel: JACK arbeitet diese Liste autonom ueber den Missions-Loop ab (jack-missions Repo).

Hinweis vorab: Ein Teil der Befunde stammt aus dem Projekt-Snapshot (kann stale sein
gegen Honor live). Jede Mission unten ist erst DIAG (misst echten Zustand), bevor
irgendwas FIX gemacht wird. Kein Blindpatchen.

---

## KATEGORIE A — KRITISCH (moegliche Live-Bugs, Halluzinations-Ursache)

### A1. Tote Xiaomi-IP 10.234.166.131 in Kernmodulen
Befund: sechs Module zeigen im Snapshot noch die alte IP statt 10.58.220.131:
jack_cortex.py, jack_gemini_bridge.py (2x, davon einmal im Gemini-SYSTEM-PROMPT!),
jack_sensors.py, jack_audit.py, jack_snapshot.py, jack_autonomous.py.
Wenn das noch live ist: Gemini bekommt als Fakt eine falsche IP mitgegeben und
antwortet dann falsch/erfindet Zustaende dazu -> direkte Halluzinations-Quelle.
Mission: grep auf Honor live nach 10.234.166 in allen jack_*.py, jeden Treffer
mit Datei+Zeile melden. Danach EIN zentrales Netz-Config-Feld (jack_config.py /
config.ini), alle Module lesen daraus, keine IP mehr hartcodiert.

### A2. Kein echter Circuit Breaker in jack_gemini_bridge.py
Befund: Snapshot zeigt nur Retry+Backoff bei HTTP 429 (3 Versuche, sleep).
Kein Fehler-Counter ueber Zeit, kein automatischer Ollama-Fallback bei
wiederholtem Scheitern. Status live pruefen.
Mission: Live-Datei dumpen, verifizieren ob Fallback existiert. Wenn nicht:
Fehler-Counter (z.B. 3 Fails in 5 Min) -> automatisch auf llama3.2:3b umschalten,
Telegram-Alarm bei Umschaltung, automatischer Rueckschalt-Versuch alle 10 Min.

### A3. Sandbox-Luecke in jack_approval.py
Befund: SAFE_DIRS enthaelt "~/jack" komplett - JACK kann damit eigene
Kernmodule als "sicher" ueberschreiben (jack_cortex.py, jack_telegram.py
selbst). Live-Stand pruefen, nicht Snapshot trauen.
Mission: Live-Version dumpen. Wenn SAFE_DIRS noch ~/jack komplett erlaubt:
auf ~/jack_werkstatt eingrenzen, Kernmodule explizit in FORBIDDEN_DIRS.

### A4. 13 blanke except: Klauseln (kein Typ, kein Log)
Fundorte: jack_autonomous.py, jack_config.py, jack_cortex.py,
jack_gemini_bridge.py, jack_handshake_gen.py, jack_operator.py,
jack_personality.py, jack_telegram.py.
Fehler verschwinden lautlos statt zu loggen - moegliche Ursache fuer
Halluzinationen, weil JACK nicht weiss dass etwas gescheitert ist.
Mission: jede Fundstelle einzeln haerten (except Exception as e + Log),
Modul fuer Modul, mit compile+Import-Test nach jedem Fix.

---

## KATEGORIE B — TELEGRAM-MODUL (Hauptfrust-Quelle)

Snapshot 498 Zeilen, live laut Onboarding ~1549 Zeilen Spaghetti. Kein
Rewrite - Strangler-Pattern: Faehigkeiten einzeln raus, telegram.py wird
nur noch Transport.

### B1. Doppelter Callback-Handler
handle_callback() wird im main()-Loop zweimal behandelt - einmal mit
Chat-ID-Check, einmal ohne. Zweiter Block koennte Callbacks von JEDEM
Chat verarbeiten, nicht nur Dimas.
Mission: Live pruefen ob Duplikat noch da ist. Toten/unsicheren Pfad raus.

### B2. Toter Code if False and raw.strip()=="/test_button_disabled"
Mission: entfernen wenn seit 30+ Tagen ungenutzt.

### B3. /shell halb-deaktiviert, toter Code danach
Return kommt vor unreachable code (out=..., return f"$ {cmd}...").
Mission: toten Code raus ODER /shell mit Telegram-User-ID-Whitelist fertig
bauen - Dima entscheiden lassen.

### B4. send() schluckt Fehler ohne Alarm
Nur lokaler print(), kein jack_log, kein Telegram-Alarm bei Fehlschlag.
Mission: Fehlschlag loggen, bei N Fails in Folge Fallback-Alarm-Datei die
jack_waechter aufgreift.

### B5. Kein Rate-Limit im Update-Loop
Mission: pruefen ob time.sleep(1) am Loop-Ende ausreichend puffert oder ob
getUpdates-Timeout das umgeht.

### B6. Voice-Datei-Cleanup nur bei Erfolg
Cleanup im try statt finally - bei Fehler bleibt .ogg liegen, Speichermuell.
Mission: Cleanup in finally-Block verschieben.

---

## KATEGORIE C — DEADMAN / OVERMIND HAERTUNG

### C1. Deadman-Alarm nie unter echten Bedingungen getestet
Mission: kuenstlich alte mtime setzen, pruefen ob ECHTER Telegram-Alarmpfad
feuert (nicht nur Mission-Act FAIL), danach mtime zuruecksetzen.

### C2. Overmind-Zahl driftet, Ursache unbekannt
Mission: DIAG-Act der 5x im 60s-Abstand die mtime-Differenz misst und
loggt, um das Muster sichtbar zu machen bevor geraten wird.

---

## KATEGORIE D — SICHERHEIT / SECRETS

### D1. Secrets-Laden ohne Fehlerbehandlung
load_secrets() / load_api_key() crashen den Dienst wenn ~/.jack_secrets
fehlt oder Rechte falsch sind, statt sauber zu melden.
Mission: try/except um Secrets-Laden, klare Fehlermeldung statt Crash-Loop.

### D2. no_secret Act system-weit ausrollen
Bisher nur 2 Dateien geprueft (Mission 056/057).
Mission: eine no_secret-Mission pro jack_*.py Modul generieren, Vollscan
gegen Klartext-Keys statt Stichprobe.

---

## KATEGORIE E — PERFORMANCE

### E1. jack_identity.json In-Memory-Cache
Wird bei jedem Gemini-Call neu von Disk gelesen.
Mission: Cache mit Invalidierung bei Schreibzugriff, Zeitmessung vorher/nachher.

### E2. voice_service_v2.py Polling -> inotifywait
sleep(1)-Polling frisst Akku.
Mission: inotifywait-Variante, A/B-Vergleich Akkuverbrauch 1h.

### E3. Lazy Imports fuer Startzeit <50ms
jack_telegram.py importiert alles global oben (jack_coder, jack_skills,
jack_claude selten gebraucht).
Mission: seltene Imports in die jeweiligen if-Bloecke verschieben.

---

## KATEGORIE F — CODE-HYGIENE

### F1. Doppelte Fix-Loading-Logik
handle_callback() approve:/reject: und /approve_fix_ machen fast Identisches.
Mission: gemeinsame Helper execute_fix(fix_id), beide Stellen rufen sie auf.

### F2. Inkonsistente Pfad-Konstruktion
Mix aus os.path.expanduser und hardcoded /data/data/com.termux/...
Mission: einheitlich auf expanduser + eine JACK_HOME-Konstante in
jack_config.py umstellen.

### F3. jack_write.py Test-Payload im Produktivcode
commit_write("../../HACK.txt",...) als Inline-Test.
Mission: pruefen ob in if __name__=="__main__" (ok) oder im Import-Pfad
(dann raus in tests/).

---

## MISSIONS-REIHENFOLGE

Phase 1 - DIAG only (062-080): Alle A-Punkte + C2 nur MESSEN, nichts patchen.
Phase 2 - Kritische Fixes (081-100): A1 A3 A4 D1, jede einzeln mit Rollback.
Phase 3 - Telegram Strangler (101-130): B1-B6 einzeln, 001-061 muessen PASS bleiben.
Phase 4 - Deadman + Performance (131-150): C1 E1-E3.
Phase 5 - Hygiene (151+): F1-F3, D2 als Breitenscan.

---

## WICHTIG FUER JACK BEIM ABARBEITEN

1. Diese Liste ist die Rohfassung - naechster Schritt ist pro Punkt eine
   echte Mission-JSON in pending/ mit passendem Act.
2. FAIL stoppt die Queue - Absicht, kein Bug.
3. Nichts wird blind gepatcht. Erst DIAG (Live-Zustand messen), dann
   entscheiden ob und wie gefixt wird.
4. Reihenfolge ist Vorschlag, Dima kann jederzeit umsortieren.
