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
| `src/settings.cpp` | PCR hart aktiv, Speakerwerte hart im RAM (Abschnitte 6/7). |
| `src/mp3.cpp` | `setVolume()` fire-and-forget; `missing OnPlayFinished`-Watchdog für classic AiO compile-time deaktiviert (Abschnitte 7/8). |
| `src/mp3.hpp` | `setVolume()` Rückgabetyp `bool` → `void`. |
| `TNG-AIO5.md` | Diese Dokumentation. |

## 5. Mode-8-Karten (`album_vb`) werden intern als Mode 16 (`hoerbuch_vb`) behandelt

- Karten, die physisch als Mode 8 (`album_vb`) geschrieben sind, werden **beim Lesen**
  intern auf Mode 16 (`hoerbuch_vb`) umgesetzt (`src/chip_card.cpp`, in `readCard()`).
- Wirkung: App-Kompatibilität (die Karten lesen sich als bekanntes Von-bis-Hörbuch),
  Von-bis-Bereich und **Fortschritt pro Ordner**.
- Die Karte bleibt **physisch Mode 8**; es wird nichts auf den Tag zurückgeschrieben.
- Gegenstelle: `hoerbuch_vb = 16` in `src/chip_card.hpp`.

## 6. PCR / `pauseWhenCardRemoved`

- `pauseWhenCardRemoved` ist für diese Firmware **hart aktiv** (`PCR:1`).
- Beim Laden wird der Wert in `src/settings.cpp` nach dem EEPROM-Read im RAM auf `1`
  erzwungen; ein zuvor gültiger EEPROM-Wert kann ihn nicht überschreiben.
- Es erfolgt **kein** Flash-Schreibvorgang → kein zusätzlicher EEPROM-Verschleiß.
- Verhalten: Karte entfernen **pausiert**, Karte wieder auflegen **resumiert**.

## 7. Lautstärke

- Speakerwerte sind für diesen Endstand **hart im RAM**: `min 1`, `max 20`, `init 6`
  (`src/settings.cpp`, nach dem Laden erzwungen, ohne Flash-Schreiben).
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
- Mode-8-Bereich und Fortschritt funktionieren.
- Pause/Resume funktioniert.
- Buttons funktionieren (A0 Pause, A1 Next, A2 Previous, A3 Vol−, A4 Vol+).
- Transientes `DfPl Err: 1` bedeutet: DFPlayer ist während des Starts **busy**;
  danach erfolgreiche Erholung (kein Funktionsfehler).

## 10. Flashen / EEPROM / Fuses / Bootloader

- Geflasht wird **ausschließlich manuell** über VSCodium/PlatformIO.
- EEPROM, Fuses und Bootloader werden **nicht gezielt gelöscht oder geändert**.

## 11. Upstream-Pflege / Rollback

- `main` = unveränderter Upstream.
- Upstream-Updates: auf `main` pullen und `tng-aio5` rebasen/mergen; der Env-Block ist
  additiv und konfliktarm.
- Rollback: Branch verwerfen (`git checkout main`), Upstream bleibt sauber.
