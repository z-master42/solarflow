## SolarFlow in Home Assistant einbinden
### Einleitung
Zendure betreibt für den Abruf von Informationen für die Produkte SuperBase V, Satellite Battery und SolarFlow einen MQTT-Broker. Dies stellt aktuell auch die einzige offizielle Möglichkeit dar, Informationen außerhalb der von Zendure bereitgestellten App abzugreifen. Da Zendure die Daten über einen eigenen Broker auspielt läuft die Verbindung zwangsweise durch Internet. Eine rein lokale Steuerung ist aktuell noch nicht möglich.
[MQTT](https://de.wikipedia.org/wiki/MQTT) ist ein offenes Netzwerkprotokoll für eine Maschine-zu-Maschine-Kommunikation. Dabei stehen in der Regel mehrere Clients in Verbindung zu einem Broker. Die ausgetauschten Nachrichten werden hierachisch abgestuft durch Topics definiert. Um entsprechende Informationen abzugreifen oder Befehle zu senden muss das entprechende Topic abonniert werden.
### Vorbereitung
Voraussetzung zur Nutzung ist ein Account bei Zendure (den ja jeder dadurch haben sollte, dass er ich die App zur Steuerung des SolarFlow installiert hat).
Für den Abruf der MQTT-Daten des eigenen SolarFlow benötigt man im weiteren einen `appKey` und ein `appSecret`.
Um diese beiden Werte zu erhalten, benötigt ihr, neben der Emailadresse mit der ihr euch bei Zendure registriert habt, die Seriennummer eures PV-Hubs.
Ich habe zum Abruf das Kommandozeilen-Tool _curl_ verwendet.

**Vorgehen auf einem Microsoft Betriebssystem**
+ Öffnet mit `Windows-Taste + R` die Eingabeaufforderung.
+ Gebt `cmd` ein.
+ Gebt folgenden Befehl in der Kommandozeile ein:
  ```
  curl -i -v --json "{'snNumber': 'EureHubSeriennummer', 'account': 'EureEmailadresse'}" https://app.zendure.tech/v2/developer/api/apply
  ```
+ Zuvor habt ihr natürlich eure Seriennummer und eure verwendete Emailadresse anstelle der Platzhalter eingetragen.

**Vorgehen auf einem Linux Betriebssystem**
+ Öffnet mit `Strg+Alt+T` ein Terminalfenster.
+ Gebt folgenden Befehl in der Kommandozeile ein:
  ```
  curl -i -X POST -H 'Content-Type: application/json' -d '{"snNumber": "EureHubSeriennummer", "account": "EureEmailadresse"}' https://app.zendure.tech/v2/developer/api/apply
  ```
+ Zuvor habt ihr natürlich eure Seriennummer und eure verwendete Emailadresse anstelle der Platzhalter eingetragen.

**Antwort**

Habt ihr keine Fehler gemacht, sollte eine Antwort wie folgt erscheinen:
```
{"code":200,"success":true,"data":{"appKey":"EuerAppKey","secret":"EuerAppSecret","mqttUrl":"mqtt.zen-iot.com","port":1883},"msg":"Successful operation"}
```

Anstelle der Platzhalter findet ihr dann euren `appKey` und euer `appSecret`. Beides sind Buchstaben-Zahlen-Kombinationen.
### Einrichtung von MQTT in Home Assistant
Je nachdem wie weit ihr euch in Home Assistant schon ausgetobt habt, gibt es nun mehrere Optionen zum Ziel.

1. **Option: Noch kein MQTT-Gerät mit Home Assistant verbunden**
   
   Wenn dies eure erste Berührung mit MQTT ist geht die Sache recht schnell.
   + Fügt über _Einstellungen - Geräte & Dienste_ eine neue Integration hinzu. Sucht dort nach MQTT und wählt die ohne irgendwelche weiteren Bezeichnungen.
   + Der Benutzername ist euer `appKey` und das Passwort euer `appSecret`. Die URL des Brokers und der Port wurden euch ebenfalls mit der o.a. Antwort geliefert: `mqtt.zen-iot.com` mit Port `1883`.
   + Damit auch Daten reinkommen müsst ihr wie eingangs erwähnt, noch ein Topic abonnieren. Dies geschieht hier in dem ihr auf der Konfigurationsseite `Enable Discovery` aktiviert und als `Discovery prefix` euren `appKey` eintragt.
     
   + ![grafik](https://github.com/z-master42/solarflow/assets/66380371/769a98f3-8786-42c3-8cc5-0d761df5aee7)
2. **Option: Ihr nutzt bereits einen MQTT-Broker in Home Assistant**
   
   Wenn ihr bereits Erfahrungen mit MQTT gesammelt habt, weil ihr z. B. Sensoren oder Steckdosen darüber mit Home Assistant verbunden habt, ist die Wahrscheinlichkeit sehr groß, dass ihr das Mosquitto-Addon installiert und die MQTT-Integration für die Verbindung mit diesem konfiguriert habt. Problem: In Home Assistant kann nur eine MQTT-Integration installiert werden. Lösung: Ihr müsst eine Brücke zum MQTT-Broker von Zendure bauen.
   Hierbei gibt es zwei Möglichkeiten des weiteren Vorgehens. Entweder ist lasst euch durch Home Assistant alle verfügbaren Sensoren automatisch anlegen oder ihr fügt diese manuell hinzu. Direkt vorweg: Ich habe meine manuell hinzugefügt, so hatte ich die Möglichkeit diese direkt noch anzupassen.
   1. **Check vorweg**
      + Um zu überprüfen, ob seitens des Zendure-Brokers überhaupt Daten eures SolarFlows ausgespielt werden, bietet sich das Programm [MQTT-Explorer](http://mqtt-explorer.com/) an, welches es für die gängisten Betriebssysteme gibt.
      + Erstellt dort eine neue Connection mit euren Zugangsdaten (`appKey`, `appSecret`) wie im Screenshot.

        ![grafik](https://github.com/z-master42/solarflow/assets/66380371/f4555568-a65c-43a1-8c8a-e6fdbb46bb96)
      + Unter Advanced (Button) müsst ihr dann noch euren `appKey` als Topic abonnieren, also als neue Subscription hinzufügen --> `appKey/#`.

        ![grafik](https://github.com/z-master42/solarflow/assets/66380371/59e3ea65-2927-4e6f-9bd9-815e761ed3d9)
      + Es sollten dann ziemlich zeitig Werte reinkommen. Wenn ihr alles aufklappt sieht es ungefähr so aus:

        ![grafik](https://github.com/z-master42/solarflow/assets/66380371/cb2d9c67-40ce-4bcb-bd18-054219adf43d)
        Unter der Broker-Adresse erscheint ein Eintrag der wie euer `appKey` lautet (alles in 🔴). Darunter gibt es drei weitere Einträge; `switch`, `sensor`, und einen der euren SolarFlo bezeichnet. Wir nennen ihn daher ab jetzt `deviceID` (alles in 🔵).
        + `switch` enthält als Einträge die Bauanleitungen für die bisher verfügbaren Schalter.
        + `sensor` enthält als Einträge die Bauanleitungen für die bisher verfügbaren Sensoren.
        + `deviceID` enthält als Eintrag die Status der Sensoren, jedoch immer nur diejenigen, deren Wert sich geändert hat.
          
        Aktuell verfügbar sind folgende Sensoren und Schalter:

          | Field | Description | device_class |
          | -------- | ------- | ------- |
          | electricLevel | Device battery percentage | sensor |
          | remainOutTime | Remaining discharge time | sensor |
          | remainInputTime | Remaining charging time | sensor |
          | socSet | Charge Capacity Limitation | sensor |
          | inputLimit | input limit | sensor |
          | outputLimit | output limit | sensor |
          | solarInputPower | solar input power | sensor |
          | packInputPower | pack input power | sensor |
          | outputPackPower | output pack power | sensor |
          | outputHomePower | output home power | sensor |
          | packNum | pack num | sensor |
          | packState | pack state(0:standby 1:input 2:output) | sensor |
          | buzzerSwitch | buzzer switch | switch |
          | masterSwitch | master switch | switch |


   2. **Gleicher Start**
      + Der Anfang ist bei beiden Möglichkeiten gleich.
      + Zunächst muss die Brücke zum Zendure-Broker aufgebaut werden. Hiefür müsst ihr auf das `share`-Verzeichnis eures Home Assistant zugreifen. Über das File Editor-Addon ist dies z. B. nicht möglich. Über dieses habt ihr nämlich nur Zugriff auf das `config`-Verzeichnis. Ich bin daher den Weg über das Samba-Addon gegangen. Möglich ist auch der Weg über SSH, hierfür gibt es auch Addons, und dann das direkte Anlegen über z. B. den Editor _nano_.
      + Erstellt im Verzeichnis eine Datei und nennt diese `zendure.conf`.
      + Fügt folgenden Inhalt ein:
        ```
        connection external-bridge
        address mqtt.zen-iot.com:1883
        remote_username <appKey>
        remote_password <appSecret>
        remote_clientid <appKey>
        topic <appKey>/# both 0
        ```
      + Alles zwischen <> ersetzt ihr natürlich wieder durch eure eigenen Daten.
      + In der Konfiguration des Mosquitto-Addons überprüft ihr jetzt noch ob unter _Customize_ `active` auf `true` gesetzt ist.
      + Abschließend ist das Addon neu zu starten. Wollt ihr, dass Home Assistant euch die Sensorentitäten automatisch anlegt überspringt diesen Schritt zunächst.
      + Im Log sollte dann ein Eintrag ähnlich `Connecting bridge external-bridge (mqtt.zen-iot.com:1883)` auftauchen. Ggf. müsst ihr das Log mehrmals aktualisieren (Geduld). Sollte hingehen irgendwas mit Timeout oder so kommen, einfach das Addon noch mal neu starten.
   3. **Automatische Einfügung**
      
      Damit Home Assistant die Sensoren und Schalter automatisch erstellt, muss es wissen wie diese aufgebaut sind. Zudem gibt es seitens Home Assistant Vorgaben, wie die Topics aussehen müssen damit dies funktioniert.
      Ergänzt hiezu in euer zendure.conf einfach noch eine Zeile mit:
      ```
      topic # in 0 homeassistant/sensor/<appKey>/ <appKey>/sensor/device/
      ```
      Nun könnt ihr das Addon neustarten.
   5. **Manuelle Einfügung**

      Das manuelle, also händische, Anlegen von Emtitäten erfolgt in Home Assistant über die configuration.yaml. Diese liegt im config-Verzeichnis. Auf dieses könnt ihr ebenso via Samba zugreifen. Genau so gut ist aber der Weg über das File Editor Addon. In diesem wird der eingefügte Code zur besseren Lesbarkeit farblich markiert und sollten Formatierungs- oder Syntaxfehler vorliegen wird dies direkt angezeigt. An für sich könnt ihr alles in die configuration.yaml schreiben. Mit der Zeit wird dies aber etwas unübersichtlich, da alles untereinander in einer quasi Textdatei steht. Schöner ist es hier, entsprechende Konfigurationen in neue Dateien auszulagern.
      + Öffnet eure configuration.yaml.
      + Ergänzt unter dem nachstehenden Block, welcher sich ziemlich am Anfang der Datei befindet eine neue Zeile mit mqtt: !include mqtt.yaml.
        ```yaml
        group: !include groups.yaml
        automation: !include automations.yaml
        script: !include scripts.yaml
        scene: !include scenes.yaml
        ```
      + Erstellt eine neue Datei mqtt.yaml und fügt nachstehenden Inhalt ein.
        ```yaml
          sensor:
        - name: "SolarFlow Hub State"
          unique_id: "deviceIDhubState"
          state_topic: "appKey/deviceID/state"
          value_template: "{{ value_json.hubState | int }}"

        - name: "SolarFlow Solar Input Power"
          unique_id: "deviceIDsolarInputPower"
          state_topic: "appKey/deviceID/state"
          unit_of_measurement: "W"
          device_class: "power"
          value_template: "{{ value_json.solarInputPower | int(0) }}"
          state_class: "measurement"
      
        - name: "SolarFlow Pack Input Power"
          unique_id: "deviceIDpackInputPower"
          state_topic: "appKey/deviceID/state"
          unit_of_measurement: "W"
          device_class: "power"
          value_template: "{{ value_json.packInputPower | int(0) }}"
          state_class: "measurement"
      
        - name: "SolarFlow Output Pack Power"
          unique_id: "deviceIDoutputPackPower"
          state_topic: "appKey/deviceID/state"
          unit_of_measurement: "W"
          device_class: "power"
          value_template: "{{ value_json.outputPackPower | int(0) }}"
          state_class: "measurement"
      
        - name: "SolarFlow Output Home Power"
          unique_id: "deviceIDoutputHomePower"
          state_topic: "appKey/deviceID/state"
          unit_of_measurement: "W"
          device_class: "power"
          value_template: "{{ value_json.outputHomePower | int(0) }}"
          state_class: "measurement"
      
        - name: "SolarFlow Output Limit"
          unique_id: "deviceIDoutputLimit"
          state_topic: "appKey/deviceID/state"
          value_template: "{{ value_json.outputLimit | int }}"
          unit_of_measurement: "W"
    
        - name: "SolarFlow Input Limit"
          unique_id: "deviceIDinputLimit"
          state_topic: "appKey/deviceID/state"
          value_template: "{{ value_json.inputLimit | int }}"
          unit_of_measurement: "W"
    
        - name: "SolarFlow Remain Out Time"
          unique_id: "deviceIDremainOutTime"
          state_topic: "appKey/deviceID/state"
          value_template: "{{ value_json.remainOutTime | int }}"
          device_class: "duration"
          unit_of_measurement: "min"
    
        - name: "SolarFlow Remain Input Time"
          unique_id: "deviceIDremainInputTime"
          state_topic: "appKey/deviceID/state"
          value_template: "{{ value_json.remainInputTime | int }}"
          device_class: "duration"
          unit_of_measurement: "min"
    
        - name: "SolarFlow Pack State"
          unique_id: "deviceIDpackState"
          state_topic: "appKey/deviceID/state"
          value_template: "{{ value_json.packState | int }}"
    
        - name: "SolarFlow Pack Num"
          unique_id: "deviceIDpackNum"
          state_topic: "appKey/deviceID/state"
          value_template: "{{ value_json.packNum | int }}"
    
        - name: "SolarFlow Electric Level"
          unique_id: "deviceIDelectricLevel"
          state_topic: "appKey/deviceID/state"
          unit_of_measurement: "%"
          device_class: "battery"
          value_template: "{{ value_json.electricLevel | int }}"
    
        - name: "SolarFlow SOC Set"
          unique_id: "deviceIDsocSet"
          state_topic: "appKey/deviceID/state"
          unit_of_measurement: "%"
          value_template: "{{ value_json.socSet | int / 10 }}"
        ```
        


