# TNG-AiO5 – Status (Branch `tng-aio5`)

Dauerhafte Statusdatei für den Stand des AiO5-Forks.
Stand: **2026-09-30** · Branch: **`tng-aio5`** · PlatformIO-Umgebung: **`TNG_AiO5`**

Diese Datei fasst den aktuellen, verifizierten Stand zusammen. Sie ersetzt die
frühere Arbeitsplan-/Übergabedokumentation (`ToDo-Boot-Optimierung.txt`,
`ToDo.md`, `Kontext.md`). Die ausführliche Fork-Dokumentation steht in
[TNG-AIO5.md](TNG-AIO5.md).

## 1. Branch / Umgebung

| Punkt | Wert |
|---|---|
| Fork | `https://github.com/steff-sson/TonUINO-TNG.git` |
| Upstream | `https://github.com/tonuino/TonUINO-TNG.git`, Branch `main`, V3.3.3 (`d527ca5`) |
| Fork-Branch | `tng-aio5` |
| PlatformIO-Umgebung | `TNG_AiO5` (electric identisch zu `ALLinONE_5`) |
| Build | `pio run -e TNG_AiO5` |
| letzter relevanter Commit | `23b5251` (TNG AiO5: EEPROM-Defaults und Boot-/Start-Optimierungen) |

`main` bleibt unberührt = reiner Upstream-Stand; alle AiO5-Anpassungen liegen auf
`tng-aio5`. Andere Targets bleiben unverändert (Ausnahme: der compile-time
deaktivierte `missing OnPlayFinished`-Watchdog greift für `ALLinONE`).

## 2. EEPROM-Defaults

Alle Werte sind **normale EEPROM-Defaults**: Sie werden nur beim Zurücksetzen
(fabrikneues bzw. geleertes EEPROM, fehlender Cookie) gesetzt und danach wie jede
andere Einstellung im EEPROM gespeichert. Nach dem Laden aus dem EEPROM werden sie
**nicht** im RAM erzwungen – **ein gültiger EEPROM-Wert gewinnt** und bleibt über
das Admin-Menü änderbar.

| Einstellung | Default | Definition |
|---|---|---|
| Speaker min / max / init | **1 / 25 / 8** | `AIO_SPK_MIN_VOLUME` / `AIO_SPK_MAX_VOLUME` / `AIO_SPK_INIT_VOLUME` |
| Standby-Timer | **15** Minuten (`0` = aus) | `AIO_STANDBY_TIMER` |
| PCR (`pauseWhenCardRemoved`) | **1** | `AIO_PAUSE_WHEN_CARD_REMOVED` |

- PCR-Default **1** gilt nur, wenn kein gültiges EEPROM vorliegt. Ein gespeichertes
  `PCR=0` bleibt erhalten (kein RAM-Erzwingen).
- `Mp3::setVolume()` ist **fire-and-forget**: Wert einmal senden, keine
  `getVolume()`-Verifikation/ACK-Abfrage.
- Zum Zurücksetzen auf die Defaults das EEPROM mit der vorhandenen
  Tastenkombination leeren (alle Tasten beim Einschalten gedrückt halten) oder im
  Admin-Menü „EEPROM reset“ wählen.

## 3. Aktuelle Implementierungen (Boot-/Start-Optimierungen)

| Maßnahme | Datei | Status |
|---|---|---|
| **Karten-Pling-Skip:** Beim Kartenstart wird das Pling (`t_262_pling`) in `StartPlay::entry()` übersprungen (`if (not tonuino.playingCard())`). Boot- und Shutdown-Pling bleiben unverändert. | `src/state_machine.cpp` | umgesetzt |
| **Mode16 ohne FolderCount:** Bei `hoerbuch_vb` (Mode 16) entfällt die teure `getFolderTrackCount()`-Abfrage in `playFolder()`; die Von-Bis-Grenzen kommen aus `special`/`special2`, `numTracksInFolder = special2` bleibt gültige Wrap-Grenze. Mode 8 (`album_vb`) wird intern auf Mode 16 remappt und profitiert mit. | `src/tonuino.cpp` | umgesetzt |
| **Deduplizierte Trackcount-Logabfrage:** `getTotalTrackCount()` wird in `mp3.init()` einmal ermittelt, in einer lokalen Variable (`trackCount`) gehalten und nur diese geloggt (`track_count: <n>`). Kein doppelter Aufruf mehr. | `src/mp3.cpp` | umgesetzt |
| **Begrenzte Readiness-Probe:** `setComRetries(1)` nur für die Probe, max. **2 Versuche**, **500 ms** Abstand, danach Restore auf Bibliotheks-Default **3**. Der **6-s-`startTrackTimer`** bleibt als Rückfallebene – das Gate wurde **bewusst nicht entfernt**. | `src/mp3.cpp` | umgesetzt |
| Test-Stub um `setComRetries()` ergänzt (UNIT_TESTS-Build) | `test/libs/DFMiniMp3.h` | umgesetzt |

**Wichtig:** Damit ist **nicht** belegt, dass alle Boot-Optimierungen abgeschlossen
sind. Die Readiness-Probe wurde bewusst nur begrenzt, nicht entfernt. Das
verbleibende Worst-Case-Budget pro Query bestimmt das feste Template-`C_ACK_TIMEOUT`
(4 s) und ist nicht lokal senkbar; eine globale Senkung wäre ein separater,
separat zu messender Schritt.

## 4. Build / Tests

- Build verifiziert (2026-09-27, `pio run -e TNG_AiO5`):
  `RAM 71.7%` (1468/2048 B), `Flash 97.4%` (28924/29696 B), `[SUCCESS]`.
- PlatformIO kann Warnungen zu altem PIO-Core und `lib_deps_esp32` ausgeben – für
  den AiO-Build irrelevant, keine Funktionsfehler.
- Funktionsprüfungen: große Karte mit 7360 Tracks wird gelesen; Mode-8-Bereich und
  Fortschritt funktionieren; Pause/Resume funktioniert; Buttons A0–A4 funktionieren.
- **Offen:** vollständige A/B-Messreihe (≥ 5 Boots/Starts pro Variante) sowie
  Hardware-Regressionstests für Admin-Menü, Quiz und Webservice.

## 5. Messergebnisse

| Szenario | Karte | Messwert |
|---|---|---|
| Boot bis Idle | kleine FAT32-Testkarte, **797 Tracks** | ca. **2,1 s** |
| Mode16-Kartenstart bis `isPlaying` | kleine Karte, 797 Tracks | ca. **1,1 s** |
| Boot | große Karte, ca. **7360 Tracks** | typischerweise ca. **5,5 s** |
| Kartenstart | große Karte | ca. **3–6 s**, je nach SD-/DFPlayer-Indexierung teils deutlich mehr |

- Referenzlog (2026-09-30, große Karte): nach `track_count: 7360` ca. 1 s später
  `Volume: 8`, dann `enter Idle`; Boot bis Idle ≈ 5,5 s.
- Die Latenz skaliert sichtbar mit der **Track-Anzahl** (vermutlich SD-/DFPlayer-
  Indexierung), nicht linear mit der reinen Kartengröße.
- `DfPl Err: 1` = **transienter Busy-Zustand** des DFPlayer während des Starts; in
  den Tests **ohne Funktionsausfall**, das Gerät erholt sich.

## 6. Erkenntnis SD-Karte / Dateilayout

- **Keine count.txt**, **keine SD-/Dateisystem-Änderung**. Es wird weder ein
  Zähler erzeugt noch gelesen; „Track-Anzahl in count.txt ablegen“ ist verworfen.
- Nutzer-Tracks bleiben im bekannten numerischen Layout (`33/003.mp3` …); nur
  `mp3/` und `advert/` werden gegen das TNG-Ansagen-Set getauscht.
- **Kartengröße (64 GB / 32 GB):** Eine 64-GB-Karte ist als **plausible, aber
  nicht endgültig bewiesene** Mitursache der langen Indexierung dokumentiert. Eine
  32-GB-Karte dient als weitere große Referenz; beide großen Karten zeigen die
  track-abhängige Startlatenz. Ein sauberer A/B-Nachweis (gleiche Tracks, nur
  Kartengröße/Format geändert) steht noch aus.
- Fortschritt wird pro **Ordennummer** via
  `settings.readFolderSettingFromFlash` / `writeFolderSettingToFlash` gehalten,
  nicht auf der RFID-Karte und nicht an die physische UID gebunden.

## 7. Offene Punkte und Risiken

| Punkt | Schwere / Status | Beschreibung |
|---|---|---|
| `DfPl Err:1` transient | niedrig / beobachtet | DFPlayer ist beim Start kurz busy; in Tests ohne Funktionsausfall, danach erfolgreiche Erholung. |
| Readiness nicht vollständig optimiert | mittel / offen | Probe nur begrenzt (`setComRetries(1)`, max. 2 Versuche, 500 ms), 6-s-Gate bleibt; Worst-Case-Budget durch festes `C_ACK_TIMEOUT` (4 s) bestimmt. Kaltstart-/SD-Wechsel-Tests offen. |
| Normale Hörbuchpfade bewusst unverändert | niedrig / gewollt | `getFolderTrackCount()` bleibt für `hoerbuch` (ohne `_vb`), Admin-Menü, Quiz und Webservice erhalten. Hardware-Regression für diese Pfade offen. |
| `special=0` nicht abgesichert | mittel / offen | `numTracksInFolder = special2` ohne FolderCount-Abfrage; bei fehlerhaft beschriebenen Karten (`special=0`/inkonsistenter Bereich) gibt es keinen Count-basierten Guard mehr. |
| A/B-Messreihe unvollständig | niedrig / offen | Rohdaten-/Median-Protokoll und 5×-Serie pro Variante stehen aus. |
| Fortschritt/Wrap | niedrig / teilweise | Wrap über `special2` gegeben, aber keine neuen Persistenz-/Resume-Tests auf Hardware. |
| Kartengröße als Ursache | niedrig / offen | 64 GB vs. 32 GB nicht durch sauberen A/B-Nachweis belegt. |

## 8. Referenzen

- Ausführliche Fork-Dokumentation: [TNG-AIO5.md](TNG-AIO5.md)
- EEPROM-Defaults im Code: `src/constants.hpp`, `src/settings.cpp`
- Boot-/Start-Optimierungen im Code: `src/mp3.cpp`, `src/tonuino.cpp`,
  `src/state_machine.cpp`
