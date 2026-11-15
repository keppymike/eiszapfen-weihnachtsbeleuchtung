# Eiszapfen-Lichterkette mit ESP32 und ESPHome steuern

Ersatz-Steuerung für eine 2-adrige LED-Lichterkette (Eiszapfen, 40 LEDs, blau), deren Original-Netzteil/Steuergerät defekt ist. Die Kette wird mit 5 V betrieben, über einen ESP32 mit Motortreiber umgepolt und per Home Assistant gesteuert. Die 8 Programme des Originals und die Dimmfunktion sind nachgebaut, die Dimmung ist stufenlos und die Effektgeschwindigkeit ist regelbar.

Getestet mit der VOLTRONIC 40 LEDs Lichterkette Eiszapfen, sollte aber mit jeder 2-adrigen Kette mit antiparallelen LED-Hälften funktionieren.

## Funktionsprinzip

Die Kette hat nur zwei Adern. Die LEDs sind in zwei Hälften mit entgegengesetzter Polung (antiparallel) geschaltet. Bei Gleichspannung leuchtet immer nur eine Hälfte, bei umgekehrter Polung die andere. Das Original-Steuergerät wechselt die Polung schnell und erzeugt daraus alle Effekte.

Der ESP32 macht dasselbe mit zwei Hardware-PWM-Kanälen (LEDC) mit 1 kHz, die um 180° versetzt laufen. Sie steuern die Eingänge AIN1 und AIN2 eines TB6612FNG (H-Brücke). Pro Hälfte ist die Helligkeit unabhängig einstellbar. Daraus entstehen alle Effekte.

## Hardware

| Teil | Hinweis | Testaufbau |
|---|---|---|
| ESP32 D1 Mini (Wemos-Format) | Pin-Beschriftung je nach Board prüfen, hier GPIO-Nummern | |
| TB6612FNG Motortreiber-Modul | Nicht den L298N verwenden (zu hoher Spannungsabfall bei 5 V) | |
| 5-V-Netzteil | Muss den Strom der Kette liefern (vorher messen) | LM2596 DC-DC-Konverter mit 12-V-Quelle am Eingang |
| Elko 100 µF / 10 V | Zwischen VM und GND am TB6612 | funktioniert auch ohne |

## Verdrahtung

| TB6612FNG | Anschluss |
|---|---|
| VM | +5 V |
| VCC | 3V3 des ESP32 |
| GND | GND (gemeinsam mit ESP32 und Netzteil) |
| PWMA | 3V3 (dauerhaft High) |
| STBY | GPIO25 |
| AIN1 | GPIO26 |
| AIN2 | GPIO27 |
| AO1 / AO2 | die zwei Adern der Lichterkette |

Der ESP32 wird ebenfalls mit 5 V versorgt (5V/VIN-Pin oder USB).

```
                +5 V ──┬───────────── ESP32 (5V/VIN)
                       │
                       └── VM ┐
 ESP32 3V3 ── VCC, PWMA ──────┤ TB6612FNG ├── AO1 ──┐
 ESP32 GPIO25 ── STBY ────────┤           │         │ Lichterkette
 ESP32 GPIO26 ── AIN1 ────────┤           │         │ (LED-Hälfte A
 ESP32 GPIO27 ── AIN2 ────────┤           ├── AO2 ──┘  und B, antiparallel)
 GND (gemeinsam) ── GND ──────┘
```

## Software

1. [ESPHome](https://esphome.io) installieren (Dashboard oder Kommandozeile).
2. `eiszapfen.yaml` in das ESPHome-Verzeichnis legen.
3. In der `secrets.yaml` diese Einträge anlegen:
   ```yaml
   wifi_ssid: "MeinWLAN"
   wifi_password: "..."
   api_encryption_key: "..."   # 32 Byte, Base64-kodiert
   ota_password: "..."         # beliebiges Passwort
   ```
   Einen gültigen API-Schlüssel erzeugt die [ESPHome-Dokumentation zur API](https://esphome.io/components/api.html) (dort steht auf der Seite ein zufällig generierter Schlüssel) oder z. B. `openssl rand -base64 32`.
4. Das erste Mal per USB flashen, danach geht es per OTA (mit `ota_password`).
5. In Home Assistant: Einstellungen → Geräte & Dienste → ESPHome → Gerät "eiszapfen" hinzufügen. Home Assistant fragt dabei nach dem API-Schlüssel (`api_encryption_key`).

## Entitäten in Home Assistant

| Entität | Funktion |
|---|---|
| Eiszapfen Ein/Aus | Schaltet die Kette ganz aus (merkt sich den Zustand nach Neustart) |
| Programm | 1 Mixed, 2 In Waves, 3 Sequential, 4 Slo-Glo, 5 Chasing/Flash, 6 Slow Fade, 7 Twinkle/Flash, 8 Steady On |
| Helligkeit | 0 bis 100 %, gamma-korrigiert (Exponent 2,2) für gleichmäßig wirkende Stufen |
| Geschwindigkeit | −50 % (halb so schnell) bis +50 % (1,5-mal so schnell) für alle Effekte; Änderung wirkt ohne Neustart des Effekts |

Programm, Helligkeit und Geschwindigkeit bleiben nach einem Neustart oder Stromausfall erhalten.

## Programme

| Nr. | Name | Verhalten (Grundtempo bei Geschwindigkeit +/-0 %) |
|---|---|---|
| 1 | Mixed | wechselt alle 8 s durch die Programme 2 bis 8 |
| 2 | In Waves | A und B leuchten in Wellen auf (Periode 3 s, phasenversetzt) |
| 3 | Sequential | A an, dann A+B, dann beide aus (je 0,7 s) |
| 4 | Slo-Glo | beide Hälften glühen langsam auf und ab (Periode 4 s) |
| 5 | Chasing/Flash | A und B blitzen abwechselnd schnell (je 0,15 s) |
| 6 | Slow Fade | A blendet aus, während B einblendet (Periode 4 s) |
| 7 | Twinkle/Flash | A glüht langsam auf und ab wie Slo-Glo (Periode 4 s), B funkelt zufällig |
| 8 | Steady On | beide Hälften dauerhaft an |

Die Timings sind Annäherungen an die üblichen Standardprogramme von Lichterketten mit zwei Gruppen und lassen sich im `interval`-Lambda in `eiszapfen.yaml` anpassen.

## Hinweise

- `max_power: 0.5` an beiden LEDC-Kanälen ist wichtig. Es begrenzt jede Hälfte auf 50 % der Periode, damit sich die Phasen nie überlappen.
- Bei Überlappung (beide Eingänge High) bremst der TB6612 kurz, und die LEDs bleiben aus. Das ist unkritisch, aber die Helligkeit sinkt.
- Strom der Kette vor dem ersten Anschließen mit dem Multimeter messen. Er muss deutlich unter 1,2 A liegen (Grenze des TB6612FNG pro Kanal), und das Netzteil muss ihn liefern können.
- Die Kette wird mit Niederspannung betrieben. Am 230-V-Netz wird nichts angeschlossen.

## Fehlersuche

- Nichts leuchtet: Programm und Helligkeit in Home Assistant prüfen, "Eiszapfen Ein/Aus" einschalten. STBY und PWMA müssen High sein. Mit dem Multimeter an AO1/AO2 prüfen, ob der Treiber schaltet.
- Nur eine Hälfte leuchtet: Prüfen, ob `phase_angle: 180°` am zweiten Kanal gesetzt ist und beide Kanäle (0 und 1) im selben Timer laufen.
- Kompilierfehler bei `phase_angle`: ESPHome aktualisieren. Das Framework muss `esp-idf` sein.

## Dateien

- `eiszapfen.yaml`: komplette ESPHome-Konfiguration
- `README.md`: diese Beschreibung
