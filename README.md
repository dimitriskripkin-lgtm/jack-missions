# jack-missions

Queue-Kanal fuer JACK Mission Runner.

## Struktur
- pending/  KI schreibt JSON hier rein
- done/     Runner verschiebt erledigte
- logs/     Ergebnisse als JSON
- STOP      Datei anlegen = Runner stoppt Queue

## Regeln
Erlaubte Acts: fact diag no_chrome_src ui_none classify_is compile_ok explain_ok sv_ok mtime_fresh json_valid no_secret
Verboten: ssh_exec git_push chrome groq_as_actor
FAIL = Queue stoppt
Honor pullt, KI pusht - nie umgekehrt
