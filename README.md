# XIAO IR Mate Bridge — Firmware v2.0

Bidirektionale IR ↔ OSC / Art-Net / WebSocket / MQTT Bridge  
für das **XIAO IR Mate** (Seeed Studio)

---

## Hardware

| Komponente | Wert |
|---|---|
| **Board** | Seeed Studio XIAO ESP32-**C3** |
| **Wiki** | <https://wiki.seeedstudio.com/XIAO_IR_Mate_Smart_IR_Remote/> |

### Pin-Belegung (fest, laut Seeed Wiki)

| GPIO | Board-Pin | Funktion |
|------|-----------|---------|
| GPIO3 | D1 | IR Sender (3× High-Power IR-LEDs) |
| GPIO4 | D2 | IR Empfänger (aktiv LOW / invertiert) |
| GPIO5 | D3 | Touch Sensor (INPUT_PULLDOWN) |
| GPIO6 | D4 | Vibrations-Motor (aktiv HIGH) |
| GPIO7 | D5 | WS2812 RGB-LED (1 Pixel) |
| GPIO9 | D9 | Reset-Taste (INPUT_PULLUP, aktiv LOW) |

> **Wichtig:** Das Board verwendet den ESP32-**C3** — kein USB OTG,  
> daher **kein USB-HID** möglich. Das ist eine Hardware-Einschränkung des C3.

---

## Features

| Feature | Beschreibung |
|---|---|
| 📡 IR Lernen | 10 Slots für rohe IR-Signale (keine Protokoll-Dekodierung nötig) |
| 📡 IR Senden | Per Weboberfläche, Touch, OSC, Art-Net, WebSocket, MQTT |
| 🌐 Web-UI | Dashboard, Slot-Manager, Protokollkonfig, Live-Log |
| 🔴 OSC | UDP :8000 empfangen · UDP :9000 senden |
| 🎛️ Art-Net | UDP :6454 · Universe 0 · ch1=Slot, ch2=Trigger |
| 🔌 WebSocket | ws://[ip]/ws bidirektional |
| 📨 MQTT | Broker konfigurierbar, User/Pass, Topics ir/received & ir/send |
| 🦾 LIRC | irsend-kompatibler TCP-Server (SEND_ONCE) |
| 🧩 Macros | Trigger send sequences when stored slots are received |
| 💡 LED | WS2812 Farbanzeige: WiFi-Status, Lernmodus, Senden, Protokollaktivität |
| 📳 Vibration | Feedback: kurz=Aktion, lang=Lernmodus/Reset |

---

## Flash / Installation

### Voraussetzungen

```bash
pip install platformio
# oder VSCode + PlatformIO IDE Extension
```

### Flashen

```bash
cd ir-bridge
# Build-only
./scripts/build.sh
# Build + upload (nutze deinen Port, z.B. COM12 oder /dev/ttyUSB0)
./scripts/flash.sh --port COM12
```

oder unter PowerShell:

```powershell
cd ir-bridge
.\scripts\build.ps1
.\scripts\flash.ps1 -Port COM12
```

Beim ersten Flash ggf. **Boot-Button gedrückt halten** → Reset drücken.

---

## Erster Start

1. **Kein WLAN konfiguriert** → Gerät startet als AP  
   SSID: `XIAO-IR-Bridge` · Passwort: `12345678`
2. Browser: **<http://192.168.4.1>**
3. *WiFi & System* → SSID + Passwort eintragen → Speichern & Neustart
4. Im eigenen Netz unter der angezeigten IP aufrufen

---

## IR-Signal Lernen (Web-UI)

1. *IR Signale* → Slot auswählen → **📡 Lernen**
2. Fernbedienung auf das Gerät richten → Taste drücken
3. Signal wird als Rohdaten gespeichert (kein Protokoll-Decode)
4. Namen vergeben → **Speichern**

**Touch-Taste am Gerät**: Kurzer Touch → Sendet Slot 0

**Reset-Taste**:

- Kurz (&lt;5s) → Neustart
- Lang (&gt;5s) → Factory Reset (alle Signale + Einstellungen gelöscht)

---

## Protokoll-Referenz

### OSC (UDP)

```
Empfangen auf :8000
IR→OSC:   /ir/received   s  "Signal3"
OSC→IR:   /ir/send       i  3          (Slot-Index)
          /ir/send       s  "Signal3"  (Name)
```

### Art-Net (UDP :6454, Universe 0)

```
IR→ArtNet:  Broadcast · ch1=Slot+1 · ch2=255
ArtNet→IR:  ch1=Slot(1–10) · ch2>0 → sendet
```

### WebSocket (ws://[ip]/ws)

```json
// IR empfangen (Device → Client)
{"type":"ir_rx","slot":3,"name":"Signal3","samples":68}

// Senden (Client → Device)
{"cmd":"send","slot":3}
{"cmd":"send","name":"Signal3"}

// Lernmodus starten (Client → Device)
{"cmd":"learn","slot":3}
```

### MQTT

| Topic | Richtung | Payload |
|---|---|---|
| `ir/received` | Device → Broker | `{"slot":3,"name":"Signal3"}` |
| `ir/send` | Broker → Device | `{"slot":3}` oder `{"name":"Signal3"}` |

### REST API

| Method | Endpoint | Body / Query |
|---|---|---|
| GET | `/api/status` | — |
| GET | `/api/slots` | — |
| POST | `/api/learn` | `{"slot":3}` |
| POST | `/api/learncancel` | — |
| POST | `/api/send` | `{"slot":3}` oder `{"name":"..."}` |
| POST | `/api/clear` | `{"slot":3}` |
| POST | `/api/rename` | `{"slot":3,"name":"play"}` |
| GET | `/api/settings` | — |
| POST | `/api/settings` | JSON-Objekt |
| GET | `/api/log` | — |
| POST | `/api/reboot` | — |
| POST | `/api/factoryreset` | — |
| GET | `/api/macros` | — |
| POST | `/api/macro` | `{"trigger":3,"actions":[{"slot":1,"delay":100}]}` |
| DELETE | `/api/macro?trigger=3` | — |
| POST | `/api/sendraw` | `{"freq":38000,"raw":[9000,4500,...]}` |

---

## LIRC (irsend) Support

Eine einfache LIRC-kompatible TCP-Schnittstelle läuft auf Port **8765**.
Verwende z.B. `irsend` oder einen LIRC-Client, um Befehle wie

```
SEND_ONCE myremote KEY_POWER
```

auszuführen (Slot-Namen werden intern über `remote.key` oder `key` gemappt).

### Backup / Restore von IR-Signalen

In der Web-Oberfläche gibt es jetzt einen Bereich zum **Exportieren** aller gespeicherten Signale als JSON-Datei und zum **Importieren** derselben Datei (z.B. zur Sicherung oder Übertragung auf ein anderes Gerät).

Du kannst das auch per API machen:

```bash
curl http://<IP>/api/slots/export -o ir-signals.json
curl -X POST http://<IP>/api/slots/import -H 'Content-Type: application/json' --data-binary @ir-signals.json
```

### Automatisches Importieren von LIRC-Remotes

Es gibt Skripte im Ordner `scripts/`, mit denen du eine LIRC `.conf`-Datei automatisch per API an das Gerät senden kannst:

- `scripts/upload-lirc-conf.ps1` (PowerShell)
- `scripts/upload-lirc-conf.sh` (bash)

Beispiel:

```powershell
.\scripts\upload-lirc-conf.ps1 -Host 192.168.1.42 -File .\remotes\sony\sony.conf
```

```bash
./scripts/upload-lirc-conf.sh --host 192.168.1.42 --file remotes/sony/sony.conf
```

Du kannst auch direkt aus dem LIRC-Git-Repository laden:

```bash
./scripts/upload-lirc-conf.sh --host 192.168.1.42 --git git://git.code.sf.net/p/lirc/git --path remotes/sony/sony.conf
```

## LED-Farben

| Farbe | Bedeutung |
|---|---|
| Weiß blinkt | WLAN-Suche |
| Grün | WLAN verbunden |
| Rot blinkt | AP-Modus |
| Orange pulsiert | Lernmodus aktiv |
| Blau blitzt | IR wird gesendet |
| Lila blitzt | OSC-Aktivität |

---

## Lizenz

MIT
