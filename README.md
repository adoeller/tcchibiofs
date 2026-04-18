# TCChibiOFS — Total Commander File System Plugin

WFX-Plugin für **Total Commander** zur Anbindung von ChibiOS-basierten
USB-Geräten per serieller Shell. Erkannte Geräte:

| Gerät | Firmware | VID/PID | Plugin-Verhalten |
|---|---|---|---|
| HackRF One + PortaPack | **Mayhem ≥ 2.0** | `1D50 : 6018` | voll zugreifbar (Shell) |
| HackRF One (nackt) | Standard-FW | `1D50 : 6089` | nur Anzeige, keine Shell |
| Flipper Zero | Stock oder Custom | `0483 : 5740` | voll zugreifbar (Shell) |

> **Nackter HackRF One** (ohne PortaPack) wird erkannt und angezeigt, aber
> als `HackRF One (COMx, no shell)` markiert und ist nicht betretbar. Das
> Standard-Firmware-Interface ist `libhackrf`/USB-Bulk, nicht Serial.
> Der Eintrag ist rein informativ.

## Funktionen

- Root-Ebene listet **nur** HackRF- und Flipper-Geräte. Andere COM-Ports
  (CH340, FTDI, virtuelle Ports von Software) werden rausgefiltert:
  ```
  ChibiOS Devices\
  ├── PortaPack (COM5)                         <- betretbar, Mayhem-Shell
  ├── HackRF One (COM8, no shell)              <- nur Info, nicht betretbar
  └── Flipper Zero AM1TELTE (COM7)             <- betretbar, Name aus USB-ID
  ```
  Der Flipper-Name (`AM1TELTE` im Beispiel) stammt aus der Windows-
  Instance-ID (`FLIP_AM1TELTE` bei `VID_0483&PID_5740`). Ist mehr als
  ein Flipper angeschlossen, lassen sie sich so zuverlässig auseinander-
  halten.
- Verzeichnis wechseln = Verbindung öffnen (automatisch)
- TC-Disconnect-Button = Verbindung sauber schließen
- **TC-Dateibefehle**: Kopieren (F5), Verschieben (F6), Löschen (F8),
  Umbenennen (Shift+F6), Neues Verzeichnis (F7)
- Binärübertragung:
  - PortaPack: `fopen` + `frb` / `fwb` (schnelle Binär-I/O)
  - Flipper: `storage read_chunks` / `storage write_chunk`
- Dateigröße: aus `filesize` bzw. `storage stat`
- Zeitstempel: Flipper via `storage timestamp` (PortaPack liefert keine)
- Fortschrittsanzeige mit Abbruch-Funktion

## Voraussetzungen

- Windows 7 oder neuer
- Total Commander 9.0 oder neuer (64-bit empfohlen)
- Mayhem-Firmware ≥ 2.0 auf HackRF/PortaPack
- Flipper Zero mit aktueller Firmware (Stock, Unleashed, Momentum, Xtreme — alle okay)
- USB-Kabel (die üblichen)

## Installation (empfohlen: integrierter TC-Installer)

1. Build-Artefakte holen:
   - `tcchibiofs.wfx` (32-bit) und/oder
   - `tcchibiofs.wfx64` (64-bit)
2. Beide Dateien zusammen mit `pluginst.inf` in ein ZIP packen.
3. Das ZIP im Total Commander öffnen (einfach anklicken) — er erkennt
   die Plugin-Installation und fragt nach Zielverzeichnis.

## Manuelle Installation

1. `tcchibiofs.wfx` (+ `tcchibiofs.wfx64`) in einen Ordner kopieren,
   z. B. `%APPDATA%\GHISLER\tcchibiofs\`
2. Total Commander: **Konfiguration → Optionen → Plugins → Konfigurieren**
   (bei „FS-Plugins")
3. **Neue**, `tcchibiofs.wfx` auswählen, Namen vergeben (z. B.
   „ChibiOS Devices")
4. Über **Netzwerkumgebung** (bzw. `\\` eintippen) → „ChibiOS Devices"
   aufrufen

## Bauen aus dem Quellcode

### Lazarus / FreePascal (empfohlen, kostenfrei)

```cmd
build-lazarus.bat both
```

liefert `tcchibiofs.wfx` und `tcchibiofs.wfx64` im Projektverzeichnis.

### Delphi

```cmd
rem einmalig: Delphi-Umgebung laden
call "C:\Program Files (x86)\Embarcadero\Studio\23.0\bin\rsvars.bat"
build.bat both
```

## Pfad-Mapping

Der Plugin-Root `\` zeigt die Liste der erkannten Geräte. Darunter liegen
die Geräte-Wurzeln. Pfade werden 1:1 auf den Shell-Pfad abgebildet:

| TC-Pfad | Geräte-Shell-Pfad |
|---|---|
| `\PortaPack (COM5)\` | `/` |
| `\PortaPack (COM5)\apps\TPMS_RX.ppma` | `/apps/TPMS_RX.ppma` |
| `\Flipper Zero (COM7)\ext\subghz\` | `/ext/subghz` |
| `\Flipper Zero (COM7)\int\apps_data\` | `/int/apps_data` |

Flipper-Besonderheit: der Plugin-Root des Flippers zeigt synthetisch
`int` und `ext` — beide sind Shell-Pfade, keine echten Unterverzeichnisse
eines gemeinsamen `/`.

## Bekannte Grenzen und Hinweise

- **PortaPack Rename ist teuer.** Mayhem kennt keinen eigenen
  Rename-Befehl. Das Plugin emuliert ihn durch Download → Upload →
  Löschen. Auf großen Dateien lieber klassisch: herunterladen,
  umbenennen, wieder hochladen.
- **HackRF-Modus auf PortaPack vermeiden.** Sobald das Gerät im
  „HackRF Mode" (blauer Bildschirm) ist, funktioniert die Serial-Shell
  nicht mehr. Vorher aus diesem Modus zurückkehren.
- **Keine Unicode-Dateinamen.** Das Plugin exportiert ausschließlich
  ANSI-Varianten der WFX-Funktionen. Geräte-Dateinamen verwenden
  sowieso ASCII.
- **Keine Thumbnails, kein Executefile.** Würde erfordern, dass das
  Plugin auf dem Zielgerät Datei-Inhalte interpretiert — ausserhalb des
  Scopes dieses Plugins.
- **Flipper im Debug-/Build-Modus** kann gelegentlich das `Ready?` vor
  dem eigentlichen Datenstrom abweichend platzieren; das Plugin ist
  tolerant, bricht aber im Zweifel mit Fehlermeldung ab statt
  korrupte Daten zu schreiben.

## Architektur

```
┌────────────────────────────────────────────────────────┐
│ Total Commander <──> tcchibiofs.wfx / .wfx64           │
├────────────────────────────────────────────────────────┤
│ uWfxMain          (WFX-Exports, Plugin-Lifecycle)       │
│ uDeviceRegistry   (offene Verbindungen, Pfad-Split)     │
│ uDeviceBase       (abstrakte Geräteklasse + Factory)    │
│ uPortapack        (Mayhem ChibiOS-Shell-Dialekt)        │
│ uFlipper          (Flipper-Storage-CLI-Dialekt)         │
│ uSerial           (Windows Serial, Timeout-gesichert)   │
│ uComScan          (COM-Ports + VID/PID aus Registry)    │
│ uStrUtils         (AnsiString-sichere Helpers)          │
│ fsplugin          (offizieller WFX-Header 2.1 SE)       │
└────────────────────────────────────────────────────────┘
```

Geräteabstraktion: `TShellDevice` definiert das Geräte-Interface
(ListDir/GetFile/PutFile/…). Ein neues Gerät einbauen = neue Unterklasse
schreiben + in `CreateDeviceForPort` registrieren.

## Fehlerbehebung

### „ChibiOS Devices" erscheint leer
- Prüfen: Gerät im Windows-Geräte-Manager als COM-Port sichtbar?
- Kabel testen (manche „Charge-only"-Kabel haben keine Datenleitung)
- Rechte: Plugin muss die Registry lesen dürfen — läuft TC mit normalem
  User-Profil, passt das

### Gerät erscheint als „[?] COM… - USB Serial Device"
Die VID/PID-Erkennung hat kein passendes Profil gefunden. Das kann
passieren bei:
- Clone-Geräten mit abweichender VID/PID
- Inkorrekt installierten Treibern

Workaround: im Windows-Geräte-Manager nachsehen, ob die VID/PID
zum Gerät passt — ggf. neu oben in der Tabelle ergänzen.

### „fopen failed" auf PortaPack
- Das Gerät ist in einer App, die die SD-Karte bereits exklusiv nutzt.
  Zur PortaPack-Hauptansicht zurückkehren.
- SD-Karte ist schreibgeschützt oder fehlt.

### „write_chunk: no Ready" auf Flipper
- Die Firmware hat `storage write_chunk` nicht mit dem erwarteten
  Protokoll beantwortet. Aktuelle Flipper-Firmware tut das, sehr alte
  Builds (vor 2022) evtl. nicht — Firmware-Update.


# TCChibiOFS — Total Commander File System Plugin

WFX plugin for **Total Commander** for connecting ChibiOS-based
USB devices via serial shell. Detected devices:

| Device | Firmware | VID/PID | Plugin behavior |
|---|---|---|---|
| HackRF One + PortaPack | **Mayhem ≥ 2.0** | `1D50 : 6018` | fully accessible (shell) |
| HackRF One (bare) | Standard firmware | `1D50 : 6089` | display only, no shell |
| Flipper Zero | Stock or custom | `0483 : 5740` | fully accessible (shell) |

> A **bare HackRF One** (without PortaPack) is detected and displayed, but
> marked as `HackRF One (COMx, no shell)` and cannot be entered. The
> standard firmware interface is `libhackrf`/USB bulk, not serial.
> The entry is purely informational.

## Features

- The root level lists **only** HackRF and Flipper devices. Other COM ports
  (CH340, FTDI, virtual ports from software) are filtered out:
  ```
  ChibiOS Devices\\
  ├── PortaPack (COM5)                         <- enterable, Mayhem shell
  ├── HackRF One (COM8, no shell)              <- info only, not enterable
  └── Flipper Zero AM1TELTE (COM7)             <- enterable, name from USB ID
  ```
  The Flipper name (`AM1TELTE` in the example) comes from the Windows
  instance ID (`FLIP_AM1TELTE` for `VID_0483&PID_5740`). If more than
  one Flipper is connected, this makes them easy to distinguish reliably.
- Changing into a directory = open connection (automatically)
- TC Disconnect button = close connection cleanly
- **TC file commands**: Copy (F5), Move (F6), Delete (F8),
  Rename (Shift+F6), New directory (F7)
- Binary transfer:
  - PortaPack: `fopen` + `frb` / `fwb` (fast binary I/O)
  - Flipper: `storage read_chunks` / `storage write_chunk`
- File size: from `filesize` or `storage stat`
- Timestamps: Flipper via `storage timestamp` (PortaPack provides none)
- Progress display with cancel function

## Requirements

- Windows 7 or newer
- Total Commander 9.0 or newer (64-bit recommended)
- Mayhem firmware ≥ 2.0 on HackRF/PortaPack
- Flipper Zero with current firmware (Stock, Unleashed, Momentum, Xtreme — all okay)
- USB cable (the usual kind)

## Installation (recommended: integrated TC installer)

1. Get the build artifacts:
   - `tcchibiofs.wfx` (32-bit) and/or
   - `tcchibiofs.wfx64` (64-bit)
2. Put both files together with `pluginst.inf` into a ZIP archive.
3. Open the ZIP in Total Commander (just click it) — it recognizes
   the plugin installation and asks for the target directory.

## Manual installation

1. Copy `tcchibiofs.wfx` (+ `tcchibiofs.wfx64`) into a folder,
   e.g. `%APPDATA%\\GHISLER\\tcchibiofs\\`
2. In Total Commander: **Configuration → Options → Plugins → Configure**
   (under “File system plugins”)
3. Click **New**, select `tcchibiofs.wfx`, assign a name (e.g.
   “ChibiOS Devices”)
4. Open it via **Network Neighborhood** (or type `\\`) → “ChibiOS Devices”

## Building from source

### Lazarus / FreePascal (recommended, free)

```cmd
build-lazarus.bat both
```

produces `tcchibiofs.wfx` and `tcchibiofs.wfx64` in the project directory.

### Delphi

```cmd
rem load Delphi environment once
call "C:\Program Files (x86)\Embarcadero\Studio\23.0\bin\rsvars.bat"
build.bat both
```

## Path mapping

The plugin root `\\` shows the list of detected devices. Below that are
the device roots. Paths are mapped 1:1 to the shell path:

| TC path | Device shell path |
|---|---|
| `\\PortaPack (COM5)\\` | `/` |
| `\\PortaPack (COM5)\\apps\\TPMS_RX.ppma` | `/apps/TPMS_RX.ppma` |
| `\\Flipper Zero (COM7)\\ext\\subghz\\` | `/ext/subghz` |
| `\\Flipper Zero (COM7)\\int\\apps_data\\` | `/int/apps_data` |

Flipper special case: the Flipper plugin root synthetically shows
`int` and `ext` — both are shell paths, not real subdirectories
of one shared `/`.

## Known limitations and notes

- **PortaPack rename is expensive.** Mayhem has no dedicated
  rename command. The plugin emulates it via download → upload →
  delete. For large files, it is better to use the classic way:
  download, rename, upload again.
- **Avoid HackRF mode on PortaPack.** As soon as the device is in
  “HackRF Mode” (blue screen), the serial shell no longer works.
  Leave this mode first.
- **No Unicode filenames.** The plugin exports only
  ANSI variants of the WFX functions. Device filenames use
  ASCII anyway.
- **No thumbnails, no executefile.** That would require the
  plugin to interpret file contents on the target device — outside the
  scope of this plugin.
- **Flipper in debug/build mode** may occasionally place `Ready?`
  differently before the actual data stream; the plugin is
  tolerant, but when in doubt it aborts with an error message instead of
  writing corrupt data.

## Architecture

```
┌────────────────────────────────────────────────────────┐
│ Total Commander <──> tcchibiofs.wfx / .wfx64           │
├────────────────────────────────────────────────────────┤
│ uWfxMain          (WFX exports, plugin lifecycle)      │
│ uDeviceRegistry   (open connections, path split)       │
│ uDeviceBase       (abstract device class + factory)    │
│ uPortapack        (Mayhem ChibiOS shell dialect)       │
│ uFlipper          (Flipper storage CLI dialect)        │
│ uSerial           (Windows serial, timeout-safe)       │
│ uComScan          (COM ports + VID/PID from registry)  │
│ uStrUtils         (AnsiString-safe helpers)            │
│ fsplugin          (official WFX header 2.1 SE)         │
└────────────────────────────────────────────────────────┘
```

Device abstraction: `TShellDevice` defines the device interface
(ListDir/GetFile/PutFile/…). Adding a new device = write a new subclass
and register it in `CreateDeviceForPort`.

## Troubleshooting

### “ChibiOS Devices” appears empty
- Check: is the device visible as a COM port in Windows Device Manager?
- Test the cable (some “charge-only” cables have no data line)
- Permissions: the plugin must be allowed to read the registry — if TC runs with a normal
  user profile, that is fine

### Device appears as “[?] COM… - USB Serial Device”
VID/PID detection did not find a matching profile. This can happen with:
- Clone devices with different VID/PID
- Incorrectly installed drivers

Workaround: check in Windows Device Manager whether the VID/PID
matches the device — if needed, add it to the table above.

### “fopen failed” on PortaPack
- The device is in an app that is already using the SD card exclusively.
  Return to the PortaPack main view.
- SD card is write-protected or missing.

### “write_chunk: no Ready” on Flipper
- The firmware did not answer `storage write_chunk` with the expected
  protocol. Current Flipper firmware does; very old builds (before 2022)
  may not — update the firmware.
