# TNG-AiO5 – klassische AiO mit 5 Buttons (TonUINO-TNG Fork)

Diese Datei dokumentiert den verifizierten Stand der **klassischen AiO-Platine mit 5 Buttons**
auf Basis der offiziellen **TonUINO-TNG**-Firmware in diesem Fork.

## 1. Ziel, Upstream, Abgrenzung

| Punkt | Wert |
|---|---|
| Fork | `https://github.com/steff-sson/TonUINO-TNG.git` |
| Upstream | `https://github.com/tonuino/TonUINO-TNG.git` |
| Upstream-Basis | Branch `main`, V3.3.3 (Commit `d527ca5`) |
| Fork-Branch | `tng-aio5` |
| PlatformIO-Umgebung | `TNG_AiO5` |

Abgrenzung:

- `main` bleibt unberührt = reiner Upstream-Stand. Alle TNG-AiO5-Anpassungen liegen auf `tng-aio5`.
- Andere Targets/Platinen werden durch die AiO5-Anpassungen nicht verändert
  (Ausnahme: der compile-time deaktivierte Watchdog greift für `ALLinONE`, siehe Abschnitt 8;
  `ALLinONE_Plus` und alle übrigen Targets bleiben unverändert).

## 2. Hardware-Zielbild

- MCU: **LGT8F328P** (AiO-Baugruppe)
- Takt: **16 MHz aus externem 32-MHz-Quarz**, Teiler 2 → `clock_source = 2`
- Buttons (`FIVEBUTTONS`): **A0 Pause, A1 Next, A2 Previous, A3 Volume−, A4 Volume+**
- `ALLinONE=1` aktiviert in `src/constants.hpp` automatisch `FIVEBUTTONS`; die Umgebung
  braucht **kein** zusätzliches `-D FIVEBUTTONS`.

## 3. Build

```bash
export PATH="$HOME/.local/bin:$PATH"
pio run -e TNG_AiO5
```

Ergebnis (2026-09-27, verifiziert):

```
RAM:   [=======   ]  71.7% (used 1468 bytes from 2048 bytes)
Flash: [========= ]  97.4% (used 28924 bytes from 29696 bytes)
========================= [SUCCESS] =========================
```

- `platform.local.txt` ist für PlatformIO nicht nötig – `platformio.ini` setzt
  `-std=gnu++17 -fconcepts` bereits als `build_flags`. Für die Arduino IDE muss
  `platform.local.txt` laut Upstream-README in den LGT8fx-HW-Ordner
  (`~/.arduino15/packages/LGT8fx Boards/hardware/avr/1.0.7`) kopiert werden.
- PlatformIO kann Warnungen zu einem alten PIO-Core und zu `lib_deps_esp32` ausgeben.
  Diese betreffen den AiO-Build nicht und sind **keine Funktionsfehler**.

## 4. Vorgenommene Anpassungen

| Datei | Änderung |
|---|---|
| `platformio.ini` | Additive Umgebung `[env:TNG_AiO5]`, elektrisch identisch zu `ALLinONE_5` (lgt8f / LGT8F328P / `f_cpu=16000000L` / `clock_source=2` / `framework-lgt8fx@1.0.6` / `-D ALLinONE=1`). Bestehende Umgebungen unverändert. |
| `src/constants.hpp` | AiO-Buttonbelegung: `buttonUpPin = A1` (Next), `buttonDownPin = A2` (Previous). |
| `src/chip_card.cpp` | Gelesene `album_vb`-Karten werden intern auf `hoerbuch_vb` abgebildet (Abschnitt 5). |
| `src/settings.cpp` | PCR, Speaker- und Standby-Werte als EEPROM-Defaults (nicht mehr im RAM erzwungen; Abschnitte 6/7). |
| `src/mp3.cpp` | `setVolume()` fire-and-forget; `missing OnPlayFinished`-Watchdog für classic AiO compile-time deaktiviert (Abschnitte 7/8); doppelte reine Log-Abfrage entfernt und Readiness-Probe begrenzt (Abschnitt 12). |
| `src/mp3.hpp` | `setVolume()` Rückgabetyp `bool` → `void`. |
| `src/state_machine.cpp` | Kartenstart-Pling in `StartPlay::entry()` übersprungen, wenn von einer Karte gestartet wird; Boot-/Shutdown-Pling unverändert (Abschnitt 12). |
| `src/tonuino.cpp` | Mode16/`hoerbuch_vb`: `getFolderTrackCount()` im `playFolder()` übersprungen, Grenzen aus `special`/`special2` (Abschnitt 12). |
| `test/libs/DFMiniMp3.h` | Test-Stub um `setComRetries()` ergänzt (UNIT_TESTS-Build, Abschnitt 12). |
| `TNG-AIO5.md` | Diese Dokumentation. |

## 5. Mode-8-Karten (`album_vb`) werden intern als Mode 16 (`hoerbuch_vb`) behandelt

- Karten, die physisch als Mode 8 (`album_vb`) geschrieben sind, werden **beim Lesen**
  intern auf Mode 16 (`hoerbuch_vb`) umgesetzt (`src/chip_card.cpp`, in `readCard()`).
- Wirkung: App-Kompatibilität (die Karten lesen sich als bekanntes Von-bis-Hörbuch),
  Von-bis-Bereich und **Fortschritt pro Ordner**.
- Die Karte bleibt **physisch Mode 8**; es wird nichts auf den Tag zurückgeschrieben.
- Gegenstelle: `hoerbuch_vb = 16` in `src/chip_card.hpp`.

## 6. PCR / `pauseWhenCardRemoved`

- `pauseWhenCardRemoved` ist ein **normaler EEPROM-Default**: In
  `src/constants.hpp` steht `AIO_PAUSE_WHEN_CARD_REMOVED = 1`, gesetzt in
  `resetSettings()` und wie die anderen Settings ins EEPROM geschrieben.
- Nach dem Laden aus dem EEPROM wird der Wert **nicht** mehr im RAM erzwungen:
  ein gültiger EEPROM-Wert gewinnt, ein gespeichertes `PCR=0` bleibt erhalten.
  Ein leeres/ungültiges EEPROM (fehlender Cookie) wird mit `PCR=1` initialisiert.
- Änderbar über das Admin-Menü (`PCR`), dort `case 1/2` in `src/state_machine.cpp`.
- Verhalten bei `PCR=1`: Karte entfernen **pausiert**, Karte wieder auflegen
  **resumiert**.

## 7. Lautstärke und Standby-Timer

- Speakerwerte sind **normale EEPROM-Defaults**: `min 1`, `max 25`, `init 8`
  (`src/constants.hpp`, gesetzt in `resetSettings()`).
- Der Standby-Timer ist ebenfalls ein Default: `15` Minuten (`0` = aus).
- Nach dem Laden aus dem EEPROM werden diese Werte **nicht** im RAM erzwungen:
  ein gültiger EEPROM-Wert gewinnt und ist über das Admin-Menü änderbar.
- Zum Zurücksetzen auf die Defaults das EEPROM mit der vorhandenen
  Tastenkombination leeren (alle Tasten beim Einschalten gedrückt halten) oder
  im Admin-Menü „EEPROM reset“ wählen.
- `Mp3::setVolume()` ist **fire-and-forget**: Wert einmal senden, **keine**
  `getVolume()`-Verifikation/ACK-Abfrage mehr.

## 8. `missing OnPlayFinished` (nur classic AiO)

- Für die classic AiO (`ALLinONE`, **nicht** `ALLinONE_Plus`) ist der
  `missing OnPlayFinished`-Watchdog samt Fallback **compile-time deaktiviert**
  (`src/mp3.cpp`, `#if !defined(ALLinONE) || defined(ALLinONE_Plus)`).
- Grund: Die gemessene DFPlayer-Startlatenz (~3,3 s beim Nutzertrack, deutlich länger
  beim Jingle) überschritt das ca. 2,4-s-Fenster des Watchdogs und löste **falsche**
  Track-Wechsel aus.
- Ohne Fallback laufen echte `Track end`-Meldungen und die Queue korrekt.
- **Andere Targets bleiben unverändert** (Originalverhalten).

## 9. Verifizierte Tests

- SD-Karte mit **7360 Tracks** wird nach langer Initialisierung gelesen.
- Kleine FAT32-Testkarte mit **797 Tracks**: Boot bis Idle ca. **2,1 s**,
  Mode16-Kartenstart bis `isPlaying` ca. **1,1 s** (Abschnitt 12.1).
- Mode-8-Bereich und Fortschritt funktionieren.
- Pause/Resume funktioniert.
- Buttons funktionieren (A0 Pause, A1 Next, A2 Previous, A3 Vol−, A4 Vol+).
- Transientes `DfPl Err: 1` bedeutet: DFPlayer ist während des Starts **busy**;
  danach erfolgreiche Erholung (kein Funktionsfehler, Abschnitt 12.1).

## 10. Flashen / EEPROM / Fuses / Bootloader

- Geflasht wird **ausschließlich manuell** über VSCodium/PlatformIO.
- EEPROM, Fuses und Bootloader werden **nicht gezielt gelöscht oder geändert**.

## 11. Upstream-Pflege / Rollback

- `main` = unveränderter Upstream.
- Upstream-Updates: auf `main` pullen und `tng-aio5` rebasen/mergen; der Env-Block ist
  additiv und konfliktarm.
- Rollback: Branch verwerfen (`git checkout main`), Upstream bleibt sauber.

## 12. Boot-/Start-Optimierungen (AiO5)

Ziel: Boot- und Kartenstart-Latenz senken, ohne Readiness-/Log-Semantik zu
beschädigen. Stand **2026-09-30**, Umgebung `TNG_AiO5`.

| Maßnahme | Datei | Status |
|---|---|---|
| Doppelte reine Log-Abfrage entfernt: `getTotalTrackCount()` einmal ermitteln, in lokaler Variable halten und nur diese loggen | `src/mp3.cpp` | umgesetzt |
| Readiness-Probe begrenzt: `setComRetries(1)` nur für die Probe, max. **2 Versuche**, **500 ms** Abstand, danach Restore auf den Bibliotheks-Default **3**; der **6-s-`startTrackTimer`** bleibt als Rückfallebene | `src/mp3.cpp` | umgesetzt |
| Kartenstart-Pling (`t_262_pling` in `StartPlay::entry()`) übersprungen, wenn die Wiedergabe von einer Karte kommt | `src/state_machine.cpp` | umgesetzt |
| Mode16/`hoerbuch_vb`: teure `getFolderTrackCount()`-Abfrage übersprungen; Von-Bis-Grenzen aus `special`/`special2`, `numTracksInFolder = special2` bleibt gültige Wrap-Grenze | `src/tonuino.cpp` | umgesetzt |
| Test-Stub um `setComRetries()` ergänzt (UNIT_TESTS-Build) | `test/libs/DFMiniMp3.h` | umgesetzt |
| Boot-Pling und Shutdown-Pling | `src/state_machine.cpp` | unverändert |

**Wichtig:** Damit ist **nicht** belegt, dass alle Boot-Optimierungen
abgeschlossen sind. Insbesondere wurde die Readiness-Probe bewusst **nicht**
entfernt, sondern nur begrenzt. Das verbleibende Worst-Case-Budget pro Query
wird durch das feste Template-`C_ACK_TIMEOUT` (4 s) bestimmt und ist nicht lokal
senkbar; eine globale Senkung wäre ein separater, separat zu messender Schritt.

### 12.1 Messresultate (2026-09-30)

| Szenario | Karte | Messwert |
|---|---|---|
| Boot bis Idle | kleine FAT32-Testkarte, **797 Tracks** | ca. **2,1 s** |
| Mode16-Kartenstart bis `isPlaying` | kleine Karte, 797 Tracks | ca. **1,1 s** |
| Boot | große Karte, ca. **7360 Tracks** | typischerweise ca. **5,5 s** |
| Kartenstart | große Karte | ca. **3–6 s**, je nach SD-/DFPlayer-Indexierung teils deutlich mehr |

- Die große Karte ist **deutlich langsamer**; die Latenz skaliert sichtbar mit
  der Track-Anzahl (vermutlich SD-/DFPlayer-Indexierung).
- Eine **64-GB-Karte** ist als **plausible, aber nicht endgültig bewiesene**
  Mitursache der langen Indexierung dokumentiert. Ein sauberer A/B-Nachweis
  (gleiche Tracks, nur Kartengröße/Format geändert) steht noch aus.
- `DfPl Err:1` ist ein **transienter Busy-Zustand** des DFPlayer während des
  Starts; in den Tests **ohne Funktionsausfall**, das Gerät erholt sich.
- Der dauerhafte Gesamtstand, offene Punkte und Risiken sind in
  [TNG-AIO5-STATUS.md](TNG-AIO5-STATUS.md) gepflegt.
