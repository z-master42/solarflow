## SolarFlow in Home Assistant einbinden
### Einleitung
Zendure betreibt für den Abruf von Informationen für die Produkte SuperBase V, Satellite Battery und SolarFlow einen MQTT-Broker. Dies stellt aktuell auch die einzige offizielle Möglichkeit dar, Informationen außerhalb der von Zendure bereitgestellten App abzugreifen. Da Zendure die Daten über einen eigenen Broker auspielt läuft die Verbindung zwangsweise durchs Internet. Eine rein lokale Steuerung ist aktuell und offiziell noch nicht möglich.
[MQTT](https://de.wikipedia.org/wiki/MQTT) ist ein offenes Netzwerkprotokoll für eine Maschine-zu-Maschine-Kommunikation. Dabei stehen in der Regel mehrere Clients in Verbindung zu einem Broker. Die ausgetauschten Nachrichten werden hierachisch abgestuft durch Topics definiert. Um entsprechende Informationen abzugreifen oder Befehle zu senden muss das entprechende Topic abonniert werden.
### Vorbereitung
Voraussetzung zur Nutzung ist ein Account bei Zendure (den ja jeder dadurch haben sollte, dass er sich die App zur Steuerung des SolarFlow installiert hat).
Für den Abruf der MQTT-Daten des eigenen SolarFlow benötigt man im weiteren einen `appKey` und ein `appSecret`.
Um diese beiden Werte zu erhalten, benötigt ihr, neben der Emailadresse mit der ihr euch bei Zendure registriert habt, die Seriennummer eures PV-Hubs.
Ich habe zum Abruf das Kommandozeilen-Tool _curl_ verwendet.

**Vorgehen auf einem Microsoft Betriebssystem**
+ Öffnet mit `Windows-Taste + R` die Eingabeaufforderung.
+ Gebt `cmd` ein.
+ Gebt folgenden Befehl in der Kommandozeile ein:
  
  Regionseinstellung in der Zendure-App auf _"Global"_
  ```
  curl -i -v --json "{'snNumber': 'EureHubSeriennummer', 'account': 'EureEmailadresse'}" https://app.zendure.tech/v2/developer/api/apply
  ```
  Regionseinstellung in der Zendure-App auf einem _"Europäischen Land"_
  ```
  curl -i -v --json "{'snNumber': 'EureHubSeriennummer', 'account': 'EureEmailadresse'}" https://app.zendure.tech/eu/developer/api/apply
  ```
+ Zuvor habt ihr natürlich eure Seriennummer und eure verwendete Emailadresse anstelle der Platzhalter eingetragen.

**Vorgehen auf einem Linux Betriebssystem**
+ Öffnet mit `Strg+Alt+T` ein Terminalfenster.
+ Gebt folgenden Befehl in der Kommandozeile ein:
    
  Regionseinstellung in der Zendure-App auf _"Global"_
  ```
  curl -i -X POST -H 'Content-Type: application/json' -d '{"snNumber": "EureHubSeriennummer", "account": "EureEmailadresse"}' https://app.zendure.tech/v2/developer/api/apply
  ```
  Regionseinstellung in der Zendure-App auf einem _"Europäischen Land"_
  ```
  curl -i -X POST -H 'Content-Type: application/json' -d '{"snNumber": "EureHubSeriennummer", "account": "EureEmailadresse"}' https://app.zendure.tech/eu/developer/api/apply
  ```
+ Zuvor habt ihr natürlich eure Seriennummer und eure verwendete Emailadresse anstelle der Platzhalter eingetragen.

**Antwort**

Habt ihr keine Fehler gemacht, sollte eine Antwort wie folgt erscheinen:
```
{"code":200,"success":true,"data":{"appKey":"EuerAppKey","secret":"EuerAppSecret","mqttUrl":"mqtt.zen-iot.com","port":1883},"msg":"Successful operation"}
```

Anstelle der Platzhalter findet ihr dann euren `appKey` und euer `appSecret`. Beides sind Buchstaben-Zahlen-Kombinationen.
### Einrichtung von MQTT in Home Assistant
Je nachdem wie weit ihr euch in Home Assistant schon ausgetobt habt, gibt es nun mehrere Wege zum Ziel.

1. **Option: Noch kein MQTT-Gerät mit Home Assistant verbunden**
   
   Wenn dies eure erste Berührung mit MQTT ist, geht die Sache recht schnell.
   + Fügt über _Einstellungen - Geräte & Dienste_ eine neue Integration hinzu. Sucht dort nach MQTT und wählt die ohne irgendwelche weiteren Bezeichnungen.
   + Der Benutzername ist euer `appKey` und das Passwort euer `appSecret`. Die URL des Brokers und der Port wurden euch ebenfalls mit der o.a. Antwort geliefert: `mqtt.zen-iot.com` mit Port `1883`.
   + Damit auch Daten reinkommen müsst ihr wie eingangs erwähnt, noch ein Topic abonnieren. Dies geschieht hier in dem ihr auf der Konfigurationsseite _Enable Discovery_ aktiviert und als _Discovery prefix_ euren `appKey` eintragt.
     
   + ![grafik](https://github.com/z-master42/solarflow/assets/66380371/769a98f3-8786-42c3-8cc5-0d761df5aee7)
2. **Option: Ihr nutzt bereits einen MQTT-Broker in Home Assistant**
   
   Wenn ihr bereits Erfahrungen mit MQTT gesammelt habt, weil ihr z. B. Sensoren oder Steckdosen darüber mit Home Assistant verbunden habt, ist die Wahrscheinlichkeit sehr groß, dass ihr das Mosquitto broker-Addon installiert und die MQTT-Integration für die Verbindung mit diesem konfiguriert habt.
   
   Problem: In Home Assistant kann nur eine MQTT-Integration installiert werden.
   
   Lösung: Ihr müsst eine Brücke zum MQTT-Broker von Zendure bauen.
   Hierbei gibt es dann zudem zwei Möglichkeiten des weiteren Vorgehens. Entweder ist lasst euch durch Home Assistant alle verfügbaren Sensoren automatisch anlegen oder ihr fügt diese manuell hinzu. Direkt vorweg: Ich habe meine manuell hinzugefügt, so hatte ich die Möglichkeit diese dabei noch anzupassen.
   1. **Check vorweg**
      + Um zu überprüfen, ob seitens des Zendure-Brokers überhaupt Daten eures SolarFlows ausgespielt werden, bietet sich das Programm [MQTT-Explorer](http://mqtt-explorer.com/) an, welches es für die gängisten Betriebssysteme gibt.
      + Erstellt dort eine neue Connection mit euren Zugangsdaten (`appKey`, `appSecret`) wie im Screenshot.

        ![grafik](https://github.com/z-master42/solarflow/assets/66380371/f4555568-a65c-43a1-8c8a-e6fdbb46bb96)
      + Unter Advanced (Button) müsst ihr dann noch euren `appKey` als Topic abonnieren, also als neue Subscription hinzufügen --> `appKey/#`.

        ![grafik](https://github.com/z-master42/solarflow/assets/66380371/59e3ea65-2927-4e6f-9bd9-815e761ed3d9)
      + Es sollten dann ziemlich zeitig Werte reinkommen. Wenn ihr alles aufklappt sieht es ungefähr so aus:

        ![grafik](https://github.com/z-master42/solarflow/assets/66380371/cb2d9c67-40ce-4bcb-bd18-054219adf43d)
        Unter der Broker-Adresse erscheint ein Eintrag der wie euer `appKey` lautet (alles in 🔴). Darunter gibt es drei weitere Einträge; `switch`, `sensor`, und einen der euren SolarFlow bezeichnet. Wir nennen ihn daher ab jetzt `deviceID` (alles in 🔵).
        + `switch` enthält als Einträge die Bauanleitungen für die bisher verfügbaren Schalter.
        + `sensor` enthält als Einträge die Bauanleitungen für die bisher verfügbaren Sensoren.
        + `deviceID` enthält als Eintrag die Status der Sensoren, jedoch immer nur diejenigen, deren Wert sich geändert hat.
          
        Aktuell verfügbar sind folgende Sensoren und Schalter:

          | Field | Description | device_class | Automatic discovery (2. iii.)
          | -------- | -------- | -------- | -------- |
          | electricLevel | Device battery percentage | sensor | yes |
          | remainOutTime | Remaining discharge time | sensor | yes |
          | remainInputTime | Remaining charging time | sensor | yes |
          | socSet | Charge Capacity Limitation | sensor | yes |
          | inputLimit | input limit | sensor | no |
          | outputLimit | output limit | sensor | yes |
          | solarInputPower | solar input power | sensor | yes |
          | packInputPower | pack input power | sensor | yes |
          | outputPackPower | output pack power | sensor | yes |
          | outputHomePower | output home power | sensor | yes |
          | packNum | pack num | sensor | yes |
          | packState | pack state(0: standby 1: input 2: output) | sensor | yes |
          | buzzerSwitch | buzzer switch | switch | yes |
          | masterSwitch | master switch | switch | yes |
          | wifiState | wifi state | sensor | no |
          | hubState | Hub output status (0: stop output standby 1: stop output and shut down) | sensor | no |
          | inverseMaxPower | inverter max power | sensor | no |
          | packData | pack Data (maxVol, minVol, maxTemp, socLevel, sn) | sensor | yes |
          | solarPower1 | Solar1 Input Power | sensor | yes |
          | solarPower2 | Solar2 Input Power | sensor | yes |
          | passMode | Bypass Mode (0: auto 1: always off 2: always on) | sensor | yes |
          | autoRecover | Automatic recovery of bypass mode settings (0: off 1: on) | switch | yes |
          | sn | Hub serial number | sensor | no |
        
        **Hinweis**: Die Sensoren, welche als Schalter angeben sind, nutzen dies nur zur Informationsdarstellung. Über den Schalter lässt sich die Funktion nicht ein- oder ausschalten.


   2. **Gleicher Start**
      + Der Anfang ist bei beiden Möglichkeiten gleich.
      + Zunächst muss die Brücke zum Zendure-Broker aufgebaut werden. Hiefür müsst ihr auf das `share`-Verzeichnis eures Home Assistant zugreifen. Über das File Editor-Addon ist dies z. B. nicht möglich. Über dieses habt ihr nämlich nur Zugriff auf das `config`-Verzeichnis. Ihr könnt dies über das Samba-Addon machen. Wir gehen an dieser Stelle aber den Weg über das Terminal & SSH-Addon. Welches ihr hierfür natürlich installieren müsst. Erstellt zunächst eine Datei mit folgendem Inhalt. Wie die Datei heißt ist völlig egal und welchen Editor oder welches Textverarbeitungsprogramm ihr dafür nutzt ebenfalls.
      + Erstellt eine neue Textdatei.
      + Fügt folgenden [Inhalt](zendure.conf) ein:
        
        ```
        connection zendure-broker
        address mqtt.zen-iot.com:1883
        remote_username <appKey>
        remote_password <appSecret>
        remote_clientid <appKey>
        topic <appKey>/# in
        ```
      + Alles zwischen <> ersetzt ihr **inklusive, der <>** natürlich wieder durch eure eigenen Daten. Es dürfen keine <> mehr vorhanden sein.
      + Installiert das Terminal & SSH-Addon im Addon-Bereich, startet es danach und öffnet die Benutzeroberfläche.
      + Wechselt mit dem Befehl `cd share` ins `share`-Verzeichnis.
      + Überprüft mit `ls` ob es schon einen Ordner `mosquitto` gibt. Erscheint nach dem Befehl einfach eine neue Eingabezeile, existiert der Ordner noch nicht, ansonsten erscheint ein Eintrag `mosquitto`.
      + Ist der Ordner nicht vorhanden, erstellt ihn mit `mkdir mosquitto`.
      + Wechselt mit `cd mosquitto` in das Verzeichnis.
      + Nun erstellen wir eine neue Datei und bearbeiten diese mit dem Editor _nano_. Gebt dazu `nano zendure.conf` ein.
      + Kopiert nun den Inhalt aus eurer zuvor angelegten Textdatei und fügt ihn in das Terminalfenster ein. **Achtung:** Hierfür müsst ihr die `Umschalttaste` eurer Tastatur gedrückt halten und mit der `rechten Maustaste` ins Terminalfenster klicken und dort _Einfügen_ auswählen. Nun sollte der Inhalt eurer Textdatei im Terminalfenster stehen.
      + Wollt ihr, dass Home Assistant euch die Sensorentitäten automatisch anlegt überspringt die weiteren Schritte zunächst und macht bei iii. weiter.
      + Schließt mit `Strg + X` den _nano_-Editor.
      + Bestätigt die am unteren Terminalrand erscheinende Frage mit `y`.
      + Bestätigt die nächste Frage mit der `Eingabetaste`.
      + In der Konfiguration des Mosquitto-Addons überprüft ihr jetzt noch ob unter _Customize_ `active` auf `true` gesetzt ist.
      + Abschließend ist das Mosquitto-Addon neu zu starten.
      + Im Log sollte dann ein Eintrag ähnlich `Connecting bridge external-bridge (mqtt.zen-iot.com:1883)` auftauchen. Ggf. müsst ihr das Log mehrmals aktualisieren (Geduld). Sollte hingehen irgendwas mit Timeout oder so kommen, einfach das Addon noch mal neu starten.
   3. **Automatische Einfügung**
      
      Damit Home Assistant die Sensoren und Schalter automatisch erstellt, muss es wissen wie diese aufgebaut sind. Zudem gibt es seitens Home Assistant Vorgaben, wie die Topics aussehen müssen damit dies funktioniert.
      Ergänzt hierzu in euer `zendure.conf` einfach noch eine Zeile mit:
      
      ```
      topic # in 0 homeassistant/sensor/<appKey>/ <appKey>/sensor/device/
      ```
      Macht nun oben da weiter, wo ihr eben hierhin abgesprungen seit.
   5. **Manuelle Einfügung**

      Das manuelle, also händische, Anlegen von Emtitäten erfolgt in Home Assistant über die `configuration.yaml`. Diese liegt im `config`-Verzeichnis. Auf dieses könnt ihr ebenso via Samba zugreifen. Genau so gut ist aber der Weg über das File Editor Addon. In diesem wird der eingefügte Code zur besseren Lesbarkeit farblich markiert und sollten Formatierungs- oder Syntaxfehler vorliegen wird dies direkt angezeigt. An für sich könnt ihr alles in die `configuration.yaml` schreiben. Mit der Zeit wird diese dann aber etwas unübersichtlich, da alles untereinander in einer Textdatei steht. Schöner ist es hier, entsprechende Konfigurationen in neue Dateien auszulagern.
      + Öffnet eure `configuration.yaml`.
      + Ergänzt unter dem nachstehenden Block, welcher sich ziemlich am Anfang der Datei befindet eine neue Zeile mit: `mqtt: !include mqtt.yaml`.
        
        ```yaml
        group: !include groups.yaml
        automation: !include automations.yaml
        script: !include scripts.yaml
        scene: !include scenes.yaml
        ```
        **Hinweis**: Der Block muss bei euch nicht genau so aussehen. Hier ist das ebenfalls davon abhängig, wie weit ihr euch in Home Assistant schon ausgetobt habt. Wenn ich das richtig in Erinnerung habe, sollte aber wenigstens eine `!include`-Zeile schon vorhanden sein.
      + Erstellt eine neue Datei `mqtt.yaml` und fügt nachstehenden [Inhalt](mqtt.yaml) ein.
      + Alles zwischen <> ersetzt ihr **inklusive, der <>** natürlich wieder durch eure eigenen Daten. Es dürfen keine <> mehr vorhanden sein.
        
        ```yaml
           # Die finalen Entitätsnamen setzen sich aus dem Sensornamen und dem Gerätenamen zusammen
           # Beim ersten Sensor hier also SolarFlow Hub State (sensor.solarflow_hub_state)
           # HINWEIS: Die u. a. weiteren Kommentare müssen vor Benutzung wieder entfernt werden! 
           sensor:
             - name: "Hub State"
               unique_id: "<deviceID>hubState"
               state_topic: "<appKey>/<deviceID>/state"
               value_template: "{{ value_json.hubState | int }}"
               device: 
                 name: "SolarFlow"
                 identifiers: "<EurePVHubSeriennummer>"
                 manufacturer: "Zendure"
                 model: "SmartPV Hub 1200 Controller"

             - name: "Solar Input Power"
               unique_id: "<deviceID>solarInputPower"
               state_topic: "<appKey>/<deviceID>/state"
               unit_of_measurement: "W"
               device_class: "power"
               value_template: >
                 {% if states('sensor.solarflow_solar_input_power') not in ['unknown'] %} # Muss ggf. an euren Entitätsnamen angepasst werden
                   {{ int(value_json.solarInputPower, 0) }}
                 {% else %}
                   {{ int(0) }}
                 {% endif %}
               state_class: "measurement"
               device: 
                 name: "SolarFlow"
                 identifiers: "<EurePVHubSeriennummer>"
                 manufacturer: "Zendure"
                 model: "SmartPV Hub 1200 Controller"
      
             - name: "Pack Input Power"
               unique_id: "<deviceID>packInputPower"
               state_topic: "<appKey>/<deviceID>/state"
               unit_of_measurement: "W"
               device_class: "power"
               value_template: >
                 {% if states('sensor.solarflow_pack_input_power') not in ['unknown'] %} # Muss ggf. an euren Entitätsnamen angepasst werden
                   {{ int(value_json.packInputPower, 0) }}
                 {% else %}
                   {{ int(0) }}
                 {% endif %}
               state_class: "measurement"
               device: 
                 name: "SolarFlow"
                 identifiers: "<EurePVHubSeriennummer>"
                 manufacturer: "Zendure"
                 model: "SmartPV Hub 1200 Controller"
      
             - name: "Output Pack Power"
               unique_id: "<deviceID>outputPackPower"
               state_topic: "<appKey>/<deviceID>/state"
               unit_of_measurement: "W"
               device_class: "power"
               value_template: >
                 {% if states('sensor.solarflow_output_pack_power') not in ['unknown'] %} # Muss ggf. an euren Entitätsnamen angepasst werden
                   {{ int(value_json.outputPackPower, 0) }}
                 {% else %}
                   {{ int(0) }}
                 {% endif %}
               state_class: "measurement"
               device: 
                 name: "SolarFlow"
                 identifiers: "<EurePVHubSeriennummer>"
                 manufacturer: "Zendure"
                 model: "SmartPV Hub 1200 Controller"
      
             - name: "Output Home Power"
               unique_id: "<deviceID>outputHomePower"
               state_topic: "<appKey>/<deviceID>/state"
               unit_of_measurement: "W"
               device_class: "power"
               value_template: >
                 {% if states('sensor.solarflow_output_home_power') not in ['unknown'] %} # Muss ggf. an euren Entitätsnamen angepasst werden
                   {{ int(value_json.outputHomePower, 0) }}
                 {% else %}
                   {{ int(0) }}
                 {% endif %}
               state_class: "measurement"
               device: 
                 name: "SolarFlow"
                 identifiers: "<EurePVHubSeriennummer>"
                 manufacturer: "Zendure"
                 model: "SmartPV Hub 1200 Controller"
      
             - name: "Output Limit"
               unique_id: "<deviceID>outputLimit"
               state_topic: "<appKey>/<deviceID>/state"
               value_template: "{{ value_json.outputLimit | int }}"
               unit_of_measurement: "W"
               device: 
                 name: "SolarFlow"
                 identifiers: "<EurePVHubSeriennummer>"
                 manufacturer: "Zendure"
                 model: "SmartPV Hub 1200 Controller"
    
             - name: "Input Limit"
               unique_id: "<deviceID>inputLimit"
               state_topic: "<appKey>/<deviceID>/state"
               value_template: "{{ value_json.inputLimit | int }}"
               unit_of_measurement: "W"
               device: 
                 name: "SolarFlow"
                 identifiers: "<EurePVHubSeriennummer>"
                 manufacturer: "Zendure"
                 model: "SmartPV Hub 1200 Controller"
    
             - name: "Remain Out Time"
               unique_id: "<deviceID>remainOutTime"
               state_topic: "<appKey>/<deviceID>/state"
               value_template: "{{ value_json.remainOutTime | int }}"
               device_class: "duration"
               unit_of_measurement: "min"
               device: 
                 name: "SolarFlow"
                 identifiers: "<EurePVHubSeriennummer>"
                 manufacturer: "Zendure"
                 model: "SmartPV Hub 1200 Controller"
    
             - name: "Remain Input Time"
               unique_id: "<deviceID>remainInputTime"
               state_topic: "<appKey>/<deviceID>/state"
               value_template: "{{ value_json.remainInputTime | int }}"
               device_class: "duration"
               unit_of_measurement: "min"
               device: 
                 name: "SolarFlow"
                 identifiers: "<EurePVHubSeriennummer>"
                 manufacturer: "Zendure"
                 model: "SmartPV Hub 1200 Controller"
    
             - name: "Pack State"
               unique_id: "<deviceID>packState"
               state_topic: "<appKey>/<deviceID>/state"
               value_template: "{{ value_json.packState | int }}"
               device: 
                 name: "SolarFlow"
                 identifiers: "<EurePVHubSeriennummer>"
                 manufacturer: "Zendure"
                 model: "SmartPV Hub 1200 Controller"
    
             - name: "Pack Num"
               unique_id: "<deviceID>packNum"
               state_topic: "<appKey>/<deviceID>/state"
               value_template: "{{ value_json.packNum | int }}"
               device: 
                 name: "SolarFlow"
                 identifiers: "<EurePVHubSeriennummer>"
                 manufacturer: "Zendure"
                 model: "SmartPV Hub 1200 Controller"
    
             - name: "Electric Level"
               unique_id: "<deviceID>electricLevel"
               state_topic: "<appKey>/<deviceID>/state"
               unit_of_measurement: "%"
               device_class: "battery"
               value_template: "{{ value_json.electricLevel | int }}"
               device: 
                 name: "SolarFlow"
                 identifiers: "<EurePVHubSeriennummer>"
                 manufacturer: "Zendure"
                 model: "SmartPV Hub 1200 Controller"
    
             - name: "SOC Set"
               unique_id: "<deviceID>socSet"
               state_topic: "<appKey>/<deviceID>/state"
               unit_of_measurement: "%"
               value_template: "{{ value_json.socSet | int / 10 }}"
               device: 
                 name: "SolarFlow"
                 identifiers: "<EurePVHubSeriennummer>"
                 manufacturer: "Zendure"
                 model: "SmartPV Hub 1200 Controller"
        
            - name: "Inverse Max Power"
              unique_id: "<deviceID>inverseMaxPower"
              state_topic: "<appKey>/<deviceID>/state"
              value_template: "{{ value_json.inverseMaxPower | int }}"
              unit_of_measurement: "W"
              device: 
                name: "SolarFlow"
                identifiers: "<EurePVHubSeriennummer>"
                manufacturer: "Zendure"
                model: "SmartPV Hub 1200 Controller"
      
            - name: "WiFi State"
              unique_id: "<deviceID>wifiState"
              state_topic: "<appKey>/<deviceID>/state"
              value_template: >
                {% if (value_json.wifiState | is_defined) %}
                  {{ value_json.wifiState | abs() }}
                {% endif %}
              device: 
                name: "SolarFlow"
                identifiers: "<EurePVHubSeriennummer>"
                manufacturer: "Zendure"
                model: "SmartPV Hub 1200 Controller"

            - name: "Solar Power 1"
              unique_id: "<deviceID>solarPower1"
              state_topic: "<appKey>/<deviceID>/state"
               value_template: >
                 {% if states('sensor.solarflow_solar_power_1') not in ['unknown'] %}
                   {{ int(value_json.solarPower1, 0) }}
                 {% else %}
                   {{ int(0) }}
                 {% endif %}
              unit_of_measurement: "W"
              device_class: "power"
              state_class: "measurement"
              device: 
                name: "SolarFlow"
                identifiers: "<EurePVHubSeriennummer>"
                manufacturer: "Zendure"
                model: "SmartPV Hub 1200 Controller"

            - name: "Solar Power 2"
              unique_id: "<deviceID>solarPower2"
              state_topic: "<appKey>/<deviceID>/state"
               value_template: >
                 {% if states('sensor.solarflow_solar_power_2') not in ['unknown'] %}
                   {{ int(value_json.solarPower2, 0) }}
                 {% else %}
                   {{ int(0) }}
                 {% endif %}
              unit_of_measurement: "W"
              device_class: "power"
              state_class: "measurement"
              device: 
                name: "SolarFlow"
                identifiers: "<EurePVHubSeriennummer>"
                manufacturer: "Zendure"
                model: "SmartPV Hub 1200 Controller"

            - name: "Pass Mode"
              unique_id: "<deviceID>passMode"
              state_topic: "<appKey>/<deviceID>/state"
              value_template: "{{ value_json.passMode | int }}"
              device: 
                name: "SolarFlow"
                identifiers: "<EurePVHubSeriennummer>"
                manufacturer: "Zendure"
                model: "SmartPV Hub 1200 Controller"

            - name: "Batterie <Nr> maxTemp"
              unique_id: "<deviceID>Batterie<Nr>maxTemp"
              state_topic: "<appKey>/<deviceID>/state"
              value_template: >
                {% if (value_json.packData | is_defined) %}
                  {% for i in value_json.packData %}
                    {% if i.sn == "<EureBatterieSeriennummer>" %}
                      {{ (i.maxTemp | float - 273.15) | round(2) }}
                    {% endif %}
                  {% endfor %}
                {% endif %}
              unit_of_measurement: "°C"
              device_class: "temperature"
              device: 
                name: "SolarFlow"
                identifiers: "<EurePVHubSeriennummer>"
                manufacturer: "Zendure"
                model: "SmartPV Hub 1200 Controller"

            - name: "Batterie <Nr> maxVol"
              unique_id: "<deviceID>Batterie<Nr>maxVol"
              state_topic: "<appKey>/<deviceID>/state"
              value_template: >
                {% if (value_json.packData | is_defined) %}
                  {% for i in value_json.packData %}
                    {% if i.sn == "<EureBatterieSeriennummer>" %}
                      {{ i.maxVol | float / 100 }}
                    {% endif %}
                  {% endfor %}
                {% endif %}
              unit_of_measurement: "V"
              device_class: "voltage"
              device: 
                name: "SolarFlow"
                identifiers: "<EurePVHubSeriennummer>"
                manufacturer: "Zendure"
                model: "SmartPV Hub 1200 Controller"

            - name: "Batterie <Nr> minVol"
              unique_id: "<deviceID>Batterie<Nr>minVol"
              state_topic: "<appKey>/<deviceID>/state"
              value_template: >
                {% if (value_json.packData | is_defined) %}
                  {% for i in value_json.packData %}
                    {% if i.sn == "<EureBatterieSeriennummer>" %}
                      {{ i.minVol | float / 100 }}
                    {% endif %}
                  {% endfor %}
                {% endif %}
              unit_of_measurement: "V"
              device_class: "voltage"
              device: 
                name: "SolarFlow"
                identifiers: "<EurePVHubSeriennummer>"
                manufacturer: "Zendure"
                model: "SmartPV Hub 1200 Controller"

            - name: "Batterie <Nr> socLevel"
              unique_id: "<deviceID>Batterie<Nr>socLevel"
              state_topic: "<appKey>/<deviceID>/state"
              value_template: >
                {% if (value_json.packData | is_defined) %}
                  {% for i in value_json.packData %}
                    {% if i.sn == "<EureBatterieSeriennummer>" %}
                      {{ i.socLevel | int }}
                    {% endif %}
                  {% endfor %}
                {% endif %}
              unit_of_measurement: "%"
              device_class: "battery"
              device: 
                name: "SolarFlow"
                identifiers: "<EurePVHubSeriennummer>"
                manufacturer: "Zendure"
                model: "SmartPV Hub 1200 Controller"
      
          switch:
            - unique_id: "<deviceID>masterSwitch"
              state_topic: "<appKey>/<deviceID>/state"
              state_off: false
              command_topic: "<appKey>/<deviceID>/masterSwitch/set"
              name: "Master Switch"
              device_class: "switch"
              value_template: "{{ value_json.masterSwitch | default('') }}"
              payload_on: true
              payload_off: false
              state_on: true
              device: 
                name: "SolarFlow"
                identifiers: "<EurePVHubSeriennummer>"
                manufacturer: "Zendure"
                model: "SmartPV Hub 1200 Controller"

            - unique_id: "<deviceID>buzzerSwitch"
              state_topic: "<appKey>/<deviceID>/state"
              state_off: false
              command_topic: "<appKey>/<deviceID>/buzzerSwitch/set"
              name: "Buzzer Switch"
              device_class: "switch"
              value_template: "{{ value_json.buzzerSwitch | default('') }}"
              payload_on: true
              payload_off: false
              state_on: true
              device: 
                name: "SolarFlow"
                identifiers: "<EurePVHubSeriennummer>"
                manufacturer: "Zendure"
                model: "SmartPV Hub 1200 Controller"

            - unique_id: "<deviceID>autoRecover"
              state_topic: "<appKey>/<deviceID>/state"
              state_off: false
              command_topic: "<appKey>/<deviceID>/autoRecover/set"
              name: "Auto Recover"
              device_class: "switch"
              value_template: "{{ value_json.autoRecover | default('') }}"
              payload_on: true
              payload_off: false
              state_on: true
              device: 
                name: "SolarFlow"
                identifiers: "<EurePVHubSeriennummer>"
                manufacturer: "Zendure"
                model: "SmartPV Hub 1200 Controller"
        ```
      + Speichert die Datei.
      + Öffnet die Entwicklerwerkzeuge und überprüft, ob eure Konfiguration fehlerfrei ist, danach startet Home Assistant neu.
      + Wenn ihr im weiteren Änderungen an dieser Sensoren- und Schalter-Konfiguration vornehmt, etwas hinzufügt oder entfernt, reicht es wenn ihr im Bereich _Neuladen der YAML-Konfiguration_ auf _Manuell konfigurierte MQTT-Entitäten_ klickt.
        
        ![grafik](https://github.com/z-master42/solarflow/assets/66380371/ac61106b-200e-44ab-bce8-e9c7fc3ff0ef)
      + Nun solltet ihr unter _Geräte & Dienste_ in eurer MQTT-Integration ein neues Gerät namens `SolarFlow` vorfinden, welches unter den o. a. Namen die entsprechenden Sensoren und Schalter enthält. Die wenigsten werden von vorne herein bereits einen Wert haben, da wie anfänglich erwähnt der Zendure-Broker nur Werteänderungen ausspielt, sich also der jeweilige Wert im Vergleich zu seinem vorherigen Status geändert haben muss. Um eine Aktualisierung aller Sensoren zu erzwingen, könnt ihr einfach mal für ein paar Augenblicke in der Zendure-App auf der Detailansicht der Batterien, dort wo ihr auch die Temperaturen sehen könnt, verweilen.
      + Abschließend noch ein paar Einlassungen zu den von mir gemachten Anpassungen, in Ergänzung zu der Zendure Sensorbauanleitung:
        + Ich habe alle Sensoren um einen Default-Wert in der `value_template`-Zeile ergänzt. Durch den Umstand, dass nur Werteänderungen übermittelt werden, schreibt euch Home Assistant für jeden Sensor bei dem beim letzten Datenaustausch kein Wert mit dabei war eine Warning (`Template variable warning: 'dict object' has no attribute 'blablabla' when rendering '{{ value_json.blablabla }}'`) in euer Log, da seitens Home Assistant die Erwartung vorhanden ist zu jedem Sensor einen Wert zu erhalten. Alle Power-Sensoren habe ich mit einer zuzätzlichen Überprüfung versehen, ob der Sensor auch ein Wert hat. Ist dem nämlich nicht so, wird der Sensor initial auf 0 W gesetzt. Da gerade die Batteriesensoren aktuell nicht definiert auf 0 W gesetzt werden, habe ich zwei kleine Automatisierungen erstellt, die dies umsetzen, da die Batterien ja nur geladen oder entladen werden können. Zuvor hatte ich diese Sensoren nur mit dem Default-Wert `int(0)` versehen. Dies sorgte dafür, dass Home Assistant weiterhin den letzten Wert angezeigte, bis ein neuer übermittelt wurde bzw., dass ein Sensor auch auf 0 gesetzt wurde, wenn er nicht mehr aktualisiert wurde. Dies klappt aber mittlerweile nicht mehr, was gerade bei den Batteriesensoren stört. Daher der u. a. neue Ansatz mit den Automatisierungen. Sicherlich nicht das Ende der Fahnenstange. Zudem habe ich eine Automatisierung erstellt, die einen jeweiligen Power-Sensor nach drei Minuten ohne Werteänderung auf 0 W setzt.
        + Ihr findet die Automatisierungen auch als einzelne Dateien links in der Übersichtsleiste.
        + Beachtet zudem, dass ich dort die "Standard Entitätennamen" verwende. Solltet ihr eure Sensoren also anders benennen, müsst ihr dies in der Automatisierung entsprechend anpassen, sonst wird sie nicht funktionieren.
          
          Batterieladung
          https://github.com/z-master42/solarflow/blob/269ed142c047246bf4079e2a69490e6f63022656/automation_charging_to_discharging.yaml#L1-L21
          Batterieentladung
          https://github.com/z-master42/solarflow/blob/269ed142c047246bf4079e2a69490e6f63022656/automation_discharging_to_charging.yaml#L1-L21
          Power-Sensoren nullen
          https://github.com/z-master42/solarflow/blob/486cacf589003cc239ca24f2bceea673e3126e97/automation_reset_power_sensors.yaml#L1-L121
          
        + _Solar Input Power_, _Pack Input Power_, _Output Pack Power_ und _Output Home Power_ habe ich um `state_class: measurement` ergänzt, damit Home Assistant mit diesen auch rechnen kann. Ob das wirklich nötig ist weiß ich gerade gar nicht. Um die Werte aber im Energie Dashboard nutzen zu können müssen sie noch zu einem Verbrauchswert (Leistung mal Zeit) integriert werden. Dafür gibt es in der Helfersektion von Home Assistant den _Riemann Summenintegralsensor_. Die Methode habe ich auf `Links` gestellt und das metrische Präfix auf `kilo` gestellt. Wenn dann ein paar Werte durchgelaufen sind könnt ihr diese dann im Energie Dashboard verwenden.
        + _Output Limit_ und _Input Limit_ haben die `unit_of_measurement: "W"` erhalten.
        + _Remain Input Time_ und _Remain Out Time_ haben die `device_class: "duration"` bekommen. Der übermittelte Wert ist die jeweilige Dauer in Minuten, sodass die `unit_of_measurement: "min"` ist. Durch die `device_class` rechnet Home Assistant das automatisch in eine Zeitangabe in h:min:s um.
        + _SOC Set_ ist von Zendure schon mit `unit_of_measurement: "%"` angegeben, allerdings liefert der Sensor dann z.B. 1000 %, wenn die obere Ladegrenze 100 % ist. Keine Ahnung warum das so sein soll. Ich habe den Wert entsprechend noch durch 10 geteilt.
        + Ich habe alle Einträge um einen `device`-Block ergänzt und als eindeutigen `identifier` meine Seriennummer genommen. Durch diesen Block weiß Home Assistant, dass es sich um Entitäten des selben Gerätes handelt und erstellt entsprechend dieses Gerät, sodass ihr es unter _Geräte & Dienste_ auch vorfinden könnt.
        + Durch den Zendure-Broker werden auch Werte ausgespielt, für die es keine Bauanleitung gibt. Ich habe sie einfach mitangelegt.
        + Natürlich könnt ihr jeder Entität hier schon ein eigenes Symbol verpassen. Ergänzt dafür für die jeweilige Entität eine `icon`-Zeile, z. B. `icon: mdi:flash`.
  4. **Vor- und Nachteile**
     | Methode | Vorteile | Nachteile |
     | -------- | -------- | -------- |
     | (1.) Keine vorherige Nutzung von MQTT in Home Assistant | Schnell hinzugefügt und wenig Aufwand | Nachträgliche Anpassung der Entitäten möglich aber ggf. umständig. Warnings im Log über fehlende `dict object`[^1] |
     | (2. iii.) Bereits MQTT in der Nutzung. Automatisches Anlegen der Entitäten durch Home Assistant | Geringfügiger Mehraufwand ggü. (1.), aber in Home Assistant nicht anders möglich | Nachträgliche Anpassung der Entitäten möglich aber ggf. umständig. Warings im Log über fehlende `dict object`[^1]|
     | (2. iv.) Bereits MQTT in der Nutzung. Manuelles Anlegen der Entitäten | Vollständige Anpassungmöglichkeiten gegeben. Ausschließen von Warnings | Ggf. umständlich und aufwendiger als (2. iii.). Nicht selbsterklärend, schwierig nachzuvollziehen für jemanden der nicht so in der Materie steckt |



[^1]: Als "Lösung" ist mir hier bisher nur bekannt, die entsprechende [Warning](warning.yaml) zu unterdrücken. 
