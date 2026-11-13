# coffeejunkie

Zählt die Bezüge einer einfachen Espressomaschine über die Leistungswerte smarter Steckdosen. Trennt Espresso und Cappuccino,
führt eine Statistik, erinnert an Rückspülen, Entkalken und Mühlenreinigung und rechnet den Bohnenvorrat mit.

coffeejunkie ist ein Home-Assistant-**Paket** (eine YAML-Datei) und kommt mit Bordmitteln aus: Template-Sensoren, Helfer, Skripte, Automationen,
`utility_meter`. Eine eigene Integration, Python oder `shell_command` braucht es nicht. Ein **Blueprint** schickt auf Wunsch stille Erinnerungen ans Handy,
eine **Beispiel-Ansicht** nutzt nur eingebaute Karten.

Die Entitäten und Texte im Paket sind englisch. Die ausführliche Anleitung steht in der [README.md](README.md), hier das Wichtigste.

## So funktioniert es

- **Bezug:** Die Mühle läuft (über 100 W), danach läuft innerhalb von 5 Minuten die Pumpe der Maschine (60 bis 130 W, dazwischen Heizimpulse). Ein Cappuccino zählt auch als Bezug.
- **Cappuccino:** Kurz nach dem Bezug heizt die Maschine länger als 90 s am Stück (Dampf).
- **Mühle gereinigt:** Ein langer Mahlgang (ab 45 s) ohne Bezug danach bucht die Mühlenreinigung. Das passt nur, wenn die Mühle ausschließlich für Espresso mahlt; Schwelle 0 schaltet es ab.

| Betriebsart | Stecker | Zuverlässigkeit |
|---|---|---|
| `with_grinder` (Standard) | Maschine und Mühle | sehr gut: im Test über 9 Tage 22 von 22 Bezügen und 4 von 4 Cappuccinos, keine Fehlzählung |
| `machine_only` | nur Maschine | gut, mit bekannter Schwäche: Was die Pumpe so lang laufen lässt wie ein Bezug, zählt mit (Rückspülen, Entkalken, Spülen). Im Test 10 von 10 Bezügen und einmal Spülen als Bezug gezählt |

Alle Schwellen sind Helfer und lassen sich im Dashboard ändern. Die Standardwerte stammen von einer Ascaso mit Thermoblock.

## Voraussetzungen

- Home Assistant ab 2024.10.
- Eine Steckdose mit Leistungsmessung an der Maschine, am besten eine zweite an der Mühle.
- **Die Steckdose muss oft melden**, mindestens alle paar Sekunden, denn ein Bezug dauert rund 30 s. Shelly meldet oft sekündlich. Bei Tasmota `TelePeriod 10` und `PowerDelta 101` setzen. Zigbee je nach Gerät, Tuya, Kasa/Tapo und andere Cloud-Stecker sind mit 30 bis 60 s meist zu langsam. Im Verlauf des Leistungssensors während eines Bezugs nachsehen.
- Die Steckdose muss die Last tragen: Eine Espressomaschine zieht über 1000 W.

## Einrichtung

1. Pakete in der `configuration.yaml` einschalten, falls noch nicht geschehen:
   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```
2. [`packages/coffeejunkie.yaml`](packages/coffeejunkie.yaml) nach `<config>/packages/coffeejunkie.yaml` kopieren.
3. In Abschnitt 1 der Datei `sensor.espresso_machine_power` und `sensor.grinder_power` durch die eigenen Leistungssensoren (W) ersetzen. Nur dort stehen eigene Entitäten.
4. Konfiguration prüfen (Entwicklerwerkzeuge > YAML) und Home Assistant **neu starten**. Beim ersten Start bekommen alle Einstellungen ihre Standardwerte, die Pflege-Uhren starten, der Bohnenvorrat steht auf einer vollen Packung.
5. Ohne Stecker an der Mühle: **Coffeejunkie mode** auf `machine_only` stellen.
6. Wahlweise die [Beispiel-Ansicht](dashboards/coffeejunkie_view.yaml) und den [Erinnerungs-Blueprint](#erinnerungen-aufs-handy) einrichten.
7. Einen Espresso machen und den Zähler ansehen. Zählt nichts: kalibrieren.

## Kalibrieren

Einen Espresso machen (gern auch einen Cappuccino) und im Abschnitt **Live and calibration** der Beispiel-Ansicht oder im Verlauf von `sensor.coffeejunkie_machine_power` nachsehen:

- **Pumpenband:** niedrigster und höchster Wert, solange nur die Pumpe läuft (die ersten Sekunden des Bezugs).
- **Heizen über:** unter der Heizleistung, über der Pumpe.
- **Heizen muss dauern:** kürzer als das Milchschäumen, länger als die Heizimpulse beim Warmhalten.
- **Nur Maschine, Bezug mindestens/höchstens:** kürzester und längster Bezug, Spülen (kurz) und Entkalken (lang) bleiben draußen.

Die Erkennungs-Sensoren (Mühle läuft, Pumpe läuft, Pumpenaktivität, Siebträger aktiv, heizt) zeigen live, was das Paket sieht.
**Coffeejunkie: reset settings to defaults** stellt die Standardwerte wieder her, die Zähler bleiben.

## Pflege und Bohnen

Eine Pflege ist fällig nach einer Zahl von Bezügen **oder** Tagen, je nachdem, was zuerst eintritt (0 = diese Grenze zählt nicht). Standard: Rückspülen alle 40 Bezüge oder 14 Tage,
Entkalken alle 90 Tage, Mühle reinigen alle 100 Bezüge oder 56 Tage. Die Anleitung der eigenen Maschine geht vor. Gebucht wird mit den Skripten `…_backflush_done`, `…_descale_done`,
`…_grinder_done`, eine neue Packung mit `script.coffeejunkie_new_bag`.

## Erinnerungen aufs Handy

[![Blueprint importieren](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/create-link/?redirect=blueprint_import&blueprint_url=https%3A%2F%2Fgithub.com%2Fmatthiasjobst87%2Fcoffeejunkie%2Fblob%2Fmain%2Fblueprints%2Fautomation%2Fcoffeejunkie%2Fcoffeejunkie_reminders.yaml)

Eine stille Meldung (Companion-App), wenn eine Pflege fällig wird oder die Bohnen knapp werden, einmal täglich eine Erinnerung, solange etwas offen ist. Ist alles erledigt, verschwindet sie.
Einstellbar: Handys, Zeitfenster, Uhrzeit der Erinnerung, Bohnengrenze und wahlweise Personen, von denen jemand zu Hause sein muss. Die Meldungstexte sind englisch.

## Grenzen

- Getestet an einer Maschine (Ascaso, Thermoblock, Vibrationspumpe) mit einer Steckerleiste (Shelly Power Strip 4 Gen4). Kesselmaschinen, Zweikreiser und Rotationspumpen brauchen andere Werte.
- `machine_only` kann einen Bezug nicht von Rückspülen, Entkalken oder Spülen gleicher Länge unterscheiden.
- Ohne Mahlgang (Pulver, Pads) zählt `with_grinder` nichts.
- Zwei Bezüge aus einem Mahlgang zählen als einer.
- Die Steckdose wird nur gemessen, nie geschaltet.

## Lizenz

MIT, siehe [LICENSE](LICENSE). Von matthiasj.
