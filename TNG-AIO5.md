# TNG-AiO5 – klassische AiO mit 5 Buttons (auf TonUINO-TNG)

Diese Datei dokumentiert den Fork-/Konfigurationsstand für die **klassische AiO-Platine mit 5 Buttons**
auf Basis der offiziellen **TonUINO-TNG**-Firmware.

## 1. Ziel, Upstream, Abgrenzung

| Punkt | Wert |
|---|---|
| Zielpfad | `/home/stef/github/TonuinoTNG` |
| Upstream | `https://github.com/tonuino/TonUINO-TNG.git` |
| Upstream-Basis | Branch `main`, Commit `d527ca5` (V3.3.3, 17.09.2026) |
| Fork-Branch | `tng-aio5` |
| Herkunft der Anforderung | Vergleichsrepo `/home/stef/github/TonUINO-Affenbox` |

Abgrenzung:
- `/home/stef/github/TonUINO-Affenbox` wurde **nicht verändert** – nur lesend als Vergleichsquelle verwendet.
- Keine Firmware wurde geflasht.
- Kein `git push`, kein Remote, keine anderen Repos angefasst.
- `main` bleibt unberührt = reiner Upstream-Stand. Alle TNG-AiO5-Anpassungen liegen auf `tng-aio5`.

## 2. Hardware-Zielbild

- MCU: **LGT8F328P** (AiO-Baugruppe)
- Takt: **16 MHz aus externem 32-MHz-Quarz**, Teiler 2 → `clock_source = 2`
- Buttons: **klassische AiO mit 5 Buttons** (`FIVEBUTTONS`)
- PlatformIO-Umgebung: `TNG_AiO5` (elektrisch identisch zu `ALLinONE_5`)

`ALLinONE=1` aktiviert in `src/constants.hpp` automatisch `FIVEBUTTONS`
(Abschnitt *AiO*: `#if not defined(THREEBUTTONS) and not defined(BUTTONS3X3) → #define FIVEBUTTONS`).
Deshalb braucht die Umgebung **kein** zusätzliches `-D FIVEBUTTONS`.

## 3. Build

```bash
export PATH="$HOME/.local/bin:$PATH"
cd /home/stef/github/TonuinoTNG
pio run -e TNG_AiO5             # eigener TNG-AiO5-Target
pio run -e ALLinONE_5           # Upstream-Referenz (identische Elektrik)
```

Ergebnis (2026-09-27):

```
Linking .pio/build/TNG_AiO5/firmware.elf
RAM:   [=======   ]  71.7% (used 1468 bytes from 2048 bytes)
Flash: [==========]  98.3% (used 29194 bytes from 29696 bytes)
========================= [SUCCESS] =========================
```

> **`platform.local.txt`:** Für PlattformIO nicht nötig – `platformio.ini` setzt
> `-std=gnu++17 -fconcepts` bereits als `build_flags`. Für die **Arduino IDE** muss
> `platform.local.txt` laut Upstream-README in den LGT8fx-HW-Ordner
> (`~/.arduino15/packages/LGT8fx Boards/hardware/avr/1.0.7`) kopiert werden.

## 4. Vorgenommene Anpassungen (bewusst minimal)

| Datei | Änderung |
|---|---|
| `platformio.ini` | Neue, additive Umgebung `[env:TNG_AiO5]` – identisch zu `ALLinONE_5` (lgt8f / LGT8F328P / `f_cpu=16000000L` / `clock_source=2` / `framework-lgt8fx@1.0.6` / `-D ALLinONE=1`). Benannter, reproduzierbarer Target für den Fork; bestehende Umgebungen unverändert. |
| `TNG-AIO5.md` | Diese Dokumentation. |
| **keine** `src/`-Änderung | Siehe Quellcodevergleich in Abschnitt 5: kein Fall war *eindeutig* so, dass ein Eingriff in den Quellcode zwingend nötig ist. |

## 5. Quellcodevergleich Affenbox ↔ TonUINO-TNG (AiO, 5 Buttons)

Physische AiO-Belegung (kanonisch, aus dem ASCII-Layout in `src/constants.hpp`):
`A0 = Pause`, `A1 = Down ▼`, `A2 = Up ▲`, `A3 = Vol−`, `A4 = Vol+`.

### 5.1 Buttons

| Physischer Pin | Affenbox (`Configuration.h` + `Affenbox.ino`, Stand `d83bf0a`) | TonUINO-TNG (`ALLinONE_5`) |
|---|---|---|
| A0 | `buttonPause` → Pause | `buttonPausePin` → Pause |
| A1 (▼) | `buttonFive` → **next** | `buttonDownPin` → **previous** |
| A2 (▲) | `buttonFour` → **previous** | `buttonUpPin` → **next** |
| A3 (Vol−) | `buttonDown` → `volumeDown` | `buttonFivePin` → `volume_down` |
| A4 (Vol+) | `buttonUp` → `volumeUp` | `buttonFourPin` → `volume_up` |

- Die **Pin-Namen sind in Affenbox und TNG vertauscht** (`buttonUp/Down` ↔ `buttonFour/Five`),
  die **Lautstärke-Belegung ist physisch identisch** (A3/A4).
- Unterschied bleibt nur die **next/prev-Richtung auf A1/A2**: TNG = Standardkonvention
  (`A2 ▲ = next`), die steff-sson-Affenbox dreht das per Commit `d83bf0a`
  (`A2 = previous`, `A1 = next`) bewusst/individuell um.
- **Bewertung:** kein *eindeutig* zwingender Grund, das im Fork nachzubauen. TNG bildet
  die dokumentierte Standard-AiO-Belegung ab; die Affenbox-Abweichung ist gerätespezifisch.
  → Als **offener Hardwaretest** dokumentiert (Abschnitt 6), nicht als Codeänderung.

### 5.2 Lautstärke

| | Affenbox | TonUINO-TNG |
|---|---|---|
| Ausgabe | `mp3.setVolume((volume / 2) + 1)` (Commit `8741065`, PAM8403-Doppelverstärkung) | `Base::setVolume(*volume)` direkt |
| Defaults | `maxVolume=10`, `minVolume=1`, `initVolume=3` (Commit `cc5b735`) | `spkMaxVolume=25`, `spkMinVolume=5`, `spkInitVolume=15` |

- **Bewertung:** Lautstärke ist eine **Laufzeit-Einstellung** (Admin-Menü „Maximal-/Minimal-/
  Initial-Lautstärke“), keine Build-Konfiguration. Eine `settings.cpp`-Änderung würde alle
  Hardware-Varianten treffen und wäre nicht minimal.
  → Im Admin-Menü anpassen (Empfehlung s. Abschnitt 6), nicht im Quellcode.

### 5.3 DFPlayer-Profil

| | Affenbox | TonUINO-TNG |
|---|---|---|
| Library | `Makuna/DFMiniMp3#1.0.7` | `makuna/DFPlayer Mini Mp3 by Makuna@1.2.3` |
| Seriell | SoftwareSerial `(2, 3)` | SoftwareSerial `dfPlayer_receivePin=2`, `dfPlayer_transmitPin=3` |
| Chip-Profil | keins (Default-Protokoll) | `#define DFMiniMp3_T_CHIP_Mp3ChipIncongruousNoAck` (Upstream-Default, gilt für alle Envs) |

- Pins/Anbindung sind identisch.
- Das Chip-Profil ist ein **globaler Upstream-Default**, keine Affenbox-spezifische Abweichung.
  → Nicht angetastet; als offener Hardwaretest dokumentiert (Abschnitt 6).

### 5.4 Fazit des Vergleichs

Aus dem Quellcodevergleich ergibt sich **keine eindeutig zwingende** `src/`-Änderung.
Einziger realer Verhaltensunterschied ist die next/prev-Richtung auf A1/A2, die gerätespezifisch
ist. Der Fork bleibt daher bewusst quellcodetreu und wird über den eigenen Build-Target geführt.

## 6. Offene Hardwaretests

Bevor diese Firmware produktiv auf der klassischen AiO-Platine läuft, am Gerät prüfen:

1. **next/prev-Richtung (A1/A2):** Reagiert ▼ (A1) auf *next* oder *previous*?
   Falls die Box die Affenbox-Konvention (`A1 = next`, `A2 = previous`) braucht, ist der minimale
   Patch in `src/constants.hpp` im AiO-Block:
   ```cpp
   inline constexpr uint8_t buttonUpPin   = A1;  // war A2
   inline constexpr uint8_t buttonDownPin = A2;  // war A1
   ```
2. **Lautstärke/PAM8403:** Startlautstärke testen. Bei zu laut/verzerrt im Admin-Menü
   Initial- und Maximal-Lautstärke senken (Affenbox fuhr effektiv ~halbe Werte).
3. **DFPlayer-Chip:** Tracks sauber starten/enden (`OnPlayFinished`)? Bei „missing OnPlayFinished“
   oder fehlenden ACKs alternatives Profil in `src/constants.hpp` wählen
   (z. B. Default-Profil ohne Chip-Define oder den konkret verbauten Chip).
4. **RFID-Gain:** Affenbox nutzt `NFCgain_avg` (mittlere Empfindlichkeit). TNG-Default ist
   `RxGain_33dB`. Bei Erkennungsproblemen `MRFC522_RX_GAIN RxGain_avg` in `constants.hpp` setzen.
5. **Bootloader/Fuses:** `clock_source=2` setzt externen 32-MHz-Takt/16 MHz. Vor dem ersten
   Flashen Bootloader-/Fuse-Zustand der AiO bestätigen (vgl. Bootloader-Recovery-Doku im
   Affenbox-Repo) – **nicht Teil dieses Auftrags, es wurde nicht geflasht**.
6. **Shutdown:** TNG nutzt `shutdownPin=7`, `activeLow` (AiO) – mit der vorhandenen
   Shutdown-Hardware (Pololu/Traeger) verifizieren.
7. **Flash-Budget:** 98,3 % belegt → **keine** zusätzlichen Features/Debug definieren, sonst
   Linker-Overflow.

## 7. SD-Migrationsanforderungen

Die SD-Karte der Affenbox ist **nicht** direkt TNG-kompatibel:

1. **Prompt-/Sprachdateien ersetzen.** TNG liefert einen eigenen `sd-card`-Satz
   (`sd-card/mp3` + `sd-card/advert`, laut Upstream-README geändert gegenüber 3.3.2).
   Die Affenbox-Prompts (`sdCard/mp3`, `sdCard/advert`) haben andere Nummern/Inhalte
   (Affenbox: 397 mp3 / 266 advert; TNG: 383 mp3 / 275 advert – u. a. Admin-Prompts und
   Spiel-Modifier weichen ab). → Ordner `mp3`/`advert` durch die TNG-Version ersetzen.
2. **Hörspiel-/Musikordner bleiben** inhaltlich erhalten (Ordner 01–99 / `mp3`-Tracks),
   sie sind unabhängig vom Prompt-Satz.
3. **RFID-Karten neu schreiben.** Das Kartenformat ist inkompatibel:
   - Affenbox: `cardVersion = 1`, `folderSettings` = 6 Byte
     (`folder, mode, special, special2, special3, special4` inkl. Track-Memory).
   - TNG: `cardVersion = 2`, `folderSettings` = 4 Byte (`folder, mode, special, special2`).
   - `cardCookie` ist identisch (`0x1337B347`), aber Version/Struktur unterscheiden sich.
   → Karten im TNG-Admin-Menü **neu anlegen** (Create new card / Modifier card / Shortcut).
4. **„Hörbuch von–bis“-Karten** neu erzeugen (TNG `hoerbuch_vb`, s. Abschnitt 8); der
   Affenbox-`special3`-Fortschritt (Track-Memory) hat im TNG-Kartenformat keinen Platz und
   muss als TNG-Hörbuchfortschritt im EEPROM neu entstehen.
5. **EEPROW:** TNG nutzt für AiO 512 Byte emuliertes EEPROM (`framework-lgt8fx@1.0.6`).
   Nach Wechsel empfiehlt sich ein EEPROM-Reset (Admin → Reset EEPROM), damit keine
   Affenbox-Reste gelesen werden.

## 8. Nutzung von `hoerbuch_vb` + `pauseWhenCardRemoved`

Beides sind **TNG-Bordmittel** (keine Zusatz-Hardware):

### `pmode_t::hoerbuch_vb` (Hörbuch von–bis)
- Entspricht dem Affenbox-„von–bis“-Abspielmodus.
- Anlegen: Admin-Menü → **Create new card** → Modus **„Hörbuch von bis“** wählen.
  TNG fragt danach ersten und letzten Track ab (State-Flow `ChFirstTrack` → `ChNumAnswer`,
  `folder.special` = von, `folder.special2` = bis).
- Quelle: `src/chip_card.hpp` (`hoerbuch_vb = 16`); Abspiel-Logik in `src/tonuino.cpp:377`.

### `pauseWhenCardRemoved` (Pause, wenn Karte entfernt wird)
- Entspricht der Affenbox-Option „Stop Wenn Karte Weg“.
- Aktivieren: Admin-Menü → Option **13 „Pause, wenn Karte entfernt wird“**
  (`Admin_PauseIfCardRemoved`, `src/state_machine.cpp:1827/2392`; Menü-Prompt
  `t_913_pause_on_card_removed`).
  Auswahl: `1 = aus` (`=0`), `2 = ein` (`=1`).
- Verhalten: Bei entfernter Karte wird pausiert/gestoppt (u. a. `state_machine.cpp:910/959/982/1023`).
- Wird auf der Karte/im EEPROM-**Settings-Bereich** gespeichert (`settings.hpp`), nicht auf dem Tag.

## 9. Risiken / offene Punkte

- **Flash 98,3 %** – praktisch kein Spielraum für weitere Features.
- **Next/Prev-Richtung** und **Lautstärke-Skalierung** sind gerätespezifisch und ungetestet
  (Abschnitt 6) – erst nach Hardwaretest final entscheiden.
- **Kein Hardwaretest erfolgt** (Auftrag: nicht flashen). Alle Aussagen sind Quellcode-/Build-basiert.
- **`clock_source=2`/Bootloader** nicht verifiziert.
- **DFPlayer-Chip-Profil** kann je nach verbautem Player abweichen.

## 10. Upstream-Pflege / Rollback

- `main` = unveränderter Upstream → jederzeit `git diff main..tng-aio5` zeigt **nur**
  `platformio.ini` (neue Env) + diese Doku.
- Upstream-Updates: auf `main` pullen und `tng-aio5` rebasen/mergen; der Env-Block ist
  additiv und konfliktarm.
- Rollback: Branch verwerfen (`git checkout main`), Upstream bleibt sauber.
