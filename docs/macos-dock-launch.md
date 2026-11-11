# macOS: xshell über das Dock gestartet ist fehlerhaft

## Symptome

Startet man xshell v1.5.0 per Klick im Dock, Finder oder Spotlight, gibt es Probleme, die beim Start aus dem Terminal (`nohup ~/.local/bin/xshell &`) nicht auftreten:

- Claude Code im Terminal-Tab ist schwarz-weiß, also ohne Farben.
- Backspace, Tab und Umlaute verhalten sich falsch.
- Agents in `~/.local/bin`, Homebrew usw. werden nicht gefunden.
- Projekt-Icons fehlen. Stattdessen steht der komplette Wert `img:/Users/.../Downloads/bild.png` als Text in Sidebar, Tab-Leiste und „Recent Projects“ und überlappt alles.

## Ursache

Eine per Dock gestartete App erbt **nicht** die Umgebung einer Terminal-Session, sondern nur die minimale Umgebung von `launchd`. Dazu kommt der macOS-Datenschutz (TCC).

| Was | Start aus dem Terminal | Start über das Dock |
|---|---|---|
| `PATH` | voller PATH aus `.zprofile`/`.zshrc` | nur `/usr/bin:/bin:/usr/sbin:/sbin` |
| `TERM` / `COLORTERM` | vom Terminal gesetzt | fehlen → TUIs fallen auf „dumb terminal“ zurück, keine Farben |
| `LANG` | setzt Terminal.app/iTerm (z. B. `de_DE.UTF-8`) | fehlt → kein UTF-8, Multibyte-Eingaben (Umlaute, Backspace) gehen kaputt |
| Arbeitsverzeichnis | `~` bzw. aktuelles Verzeichnis | `/` |
| Zugriff auf `~/Downloads`, `~/Desktop`, `~/Documents` | läuft über die Berechtigung des Terminals | xshell selbst hat keine Berechtigung → Lesen schlägt fehl |

Der Icon-Bug hat zwei Teile:

1. `read_image_base64` kann das Bild in `~/Downloads` nicht lesen, weil TCC den Zugriff blockiert.
2. Das Frontend fällt dann auf `{iconValue || initials}` zurück. `iconValue` ist aber der rohe String `img:<pfad>`, deshalb wird der Pfad als Text gerendert.

## Fix (Branch `fix/macos-dock-launch`)

**Backend (`src-tauri/src/lib.rs`)**

- `import_login_shell_path()`: holt beim Start einmal den `PATH` aus der Login-Shell des Users (`$SHELL -l -i -c`, mit 5 s Timeout) und übernimmt ihn.
- `normalize_gui_launch_env()`: Ist kein `LC_ALL`/`LC_CTYPE`/`LANG` gesetzt, wird `LANG` aus der macOS-Region (`defaults read -g AppleLocale`) + `.UTF-8` abgeleitet. Ist das Arbeitsverzeichnis `/`, wird nach `$HOME` gewechselt.
- Neuer Command `import_project_image`: kopiert ein im Dialog gewähltes Bild nach `<app_data_dir>/project-icons/`. Der Datei-Dialog gibt einmalig Zugriff auf die gewählte Datei. Die Kopie im App-Datenordner bleibt danach immer lesbar, egal wie die App gestartet wurde.
- `TERM=xterm-256color` / `COLORTERM=truecolor` für gespawnte Terminals sind schon upstream (#67).

**Frontend**

- `ProjectEditorDialog`: ruft nach der Bildauswahl `import_project_image` auf und speichert den neuen Pfad.
- `textIcon()` in `useProjectImage.ts`: Sidebar, TabBar, HomeView und QuickActions zeigen bei einem nicht ladbaren Bild die Initialen statt `img:<pfad>`.

## Bestehende Icons migrieren

Icons, die vor diesem Fix gesetzt wurden, verweisen weiter auf den Originalpfad. Entweder das Icon im Dialog „Edit project…“ neu wählen, oder die Dateien nach `~/Library/Application Support/com.xshell.app/project-icons/` kopieren und die Pfade in `settings.json` anpassen.

## Lokal bauen

```sh
npm install
npx tauri build --bundles app -c '{"bundle":{"createUpdaterArtifacts":false}}'
ditto src-tauri/target/release/bundle/macos/xshell.app /Applications/xshell.app
```
