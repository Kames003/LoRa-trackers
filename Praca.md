Config cez SenseCraft ---> stlačit a drzat button pre 3 sec ---> nasledne v advance settings ---> lora nastavut Decice EUI a AppEUI na to čo je vo wisgate v evente pod ktorým su pridané tieto trackers 
Okamžite odoslanie lokacie je stlačenim akčneho tlačidla 

# Čo je to OTAA 
bezpečnostný mechanizmus pre pripojenie do LoRaWAN siete 

Ako to funguje ? 

Join request : tracker pošle žiadosť o pripojenie do siete obsahujúcu DevUI - unikátne idčko zariadenia a AppEUI

Join accept : network server overi ziadosť pomocou AppKey - predsdielaný kluč - odpovedá potvrdenim 

Generování session klíčů: Po úspěšném přijetí se dynamicky vygenerují:
* NwkSKey (Network Session Key) - pro šifrování MAC příkazů
* AppSKey (Application Session Key) - pro šifrování aplikačních dat

Prečo zrovna OTAA ? 

Bezpečnost: Session klíče se mění při každém připojení - Keď sa tracker vypne a znova sa pripojí dostane nové session klíče 

Prvé pripojenie : 

1. Tracker zapnut → Join Request
2. Network Server → Join Accept
3. Vygenerují se: NwkSKey_v1, AppSKey_v1
4. Tracker komunikuje s těmito klíči

Tracker se vypne a znovu zapne:

1. Tracker zapnut → Nový Join Request
2. Network Server → Nový Join Accept
3. Vygenerují se: NwkSKey_v2, AppSKey_v2 (JINÉ než v1!)
4. Tracker komunikuje s NOVÝMI klíči


MQTT (Message Queuing Telemetry Transport)

je to pub/sub pracuje publishe and subscribe mechanism 

publisher = posiela správy do toho tematu do nijakeho komunikačného kanal

mqqt broker je srdcom mqqt protokolu = zodpovedny za priatie týchto správ ich prefiltrovanie zistenie who is interested in them a nasledne ich publishnut k subsribe clientom 

v mqqt su tu dve important veci a to sú Topic and message and the payload 

topics explains the structure of the datas 
su reprezentované stringom a oddelene / ===Shell/A1/Temp 
kazdy / reprezentuje level 
takze budeme mať Shell a v nom Area1 mame temperature value 32 

==> takto teda vyzerá vstup do mqtt brokera 

takze mqtt bude mat info o všetkych publisheroch / clientoch a o ich publikovaných informáciach 


Subscriber = napr client ktorý je subscribnutý k tejto informácií, broker will send the publisher information do toho špecifického klienta ktorý mal o nu zaujem  

moze byt viacero odberatelov ktorý ktorý pozaduju udaje publishnute z clienta do mqtt brokera 

totožne tak by to fungovalo keby máme napr. ďalšieho publishera vytvori nový entry 

MQTT Topic struktura

# Linux filesystem
/home/user/documents/project/file.txt
  ↓     ↓      ↓        ↓       ↓
root  user  folder  subfolder  file

# MQTT topics
lorawan/trackers/event123/tracker001/location
   ↓       ↓        ↓         ↓         ↓
root   category  event    device    data-type

Pozor lorawan/trackers/event123/tracker001/location je len len virtuálny kanal - topic, neexistuje je to len adresa kam sa posielajú správy samotné mqtt broker si teda len pamatá kto sa zaujima o aky topic 

---

Celkový dátový tok nášho projektu 

1. Tracker (T-1000) 
   ↓ [LoRaWAN - OTAA]
2. LoRaWAN Gateway
   ↓ [MQTT publish]
3. MQTT Broker (Mosquitto)
   ↓ [MQTT subscribe]
4. Backend
   - Base64 dekódování
   - Parsing payloadu
   - Uložení do DB
   ↓ [WebSocket push]
5. Webová aplikace
   - Zobrazení na mapě
   - Real-time aktualizace
   - Časová osa


aplikované pre náš projekt : 


publisher je logicky LoRaWAN brána 


Topic: "event/summer-festival/tracker/T1000A-001/uplink"
Payload: 
{
  "payload_raw": "AQIDBAUGBwg=",
  "device_id": "T1000A-001",
  "timestamp": "2025-10-08T14:23:45Z",
  "gateway_id": "gateway-01"
}


MQTT BROKER (Mosquitto):

Přijme zprávu od Gateway
Uloží ji do topic struktury
Ví, kdo je subscribnutý na tento topic
Pošle zprávu všem subscriberům

SUBSCRIBER - backed 
može byt subscriber2 iny klient ktorý prakticky moze takisto odoberať data napr monitoring / logging systemu samotného 

Dátovy tok v našom projekte by sa dal opísať i nijako takto : 


┌─────────────┐
│   Tracker   │ (T-1000A, T-1000B)
│  (LoRaWAN)  │
└──────┬──────┘
       │ LoRaWAN payload
       ↓
┌─────────────────┐
│ LoRaWAN Gateway │ (PUBLISHER)
│      (4G)       │
└──────┬──────────┘
       │ MQTT Publish
       │ Topic: event/{id}/tracker/{device}/uplink
       ↓
┌──────────────────┐
│  MQTT Broker     │ (Mosquitto)
│   (centrální)    │
└──────┬───────────┘
       │ MQTT Subscribe
       ├─────────┬────────┐
       ↓         ↓        ↓
┌─────────┐ ┌────────┐ ┌─────────┐
│ Backend │ │ Logger │ │ Monitor │
│  (tvůj) │ │        │ │         │
└─────────┘ └────────┘ └─────────┘

Wildcards v MQTT 

wildcars su ako * v Linuxe 

/home/*/documents  == je tam tam hviezdička berie akehokovlek jedneho uzivatela 

subscribe('lorawan/trackers/event123/+/location')

// Dostaneš zprávy z:
✓ lorawan/trackers/event123/tracker001/location
✓ lorawan/trackers/event123/tracker002/location
✓ lorawan/trackers/event123/tracker999/location

// NEDOSTANEŠ zprávy z:
✗ lorawan/trackers/event123/location              ← chybí úroveň
✗ lorawan/trackers/event123/tracker001/battery    ← jiná koncovka
✗ lorawan/trackers/event456/tracker001/location   ← jiný event

SINGLE WILDCARDS == + 

// Chci všechny trackery z konkrétního eventu
subscribe('lorawan/trackers/event123/+/location')

// Chci konkrétní tracker ze všech eventů
subscribe('lorawan/trackers/+/tracker001/location')

// Chci všechny trackery ze všech eventů
subscribe('lorawan/trackers/+/+/location')

= Nahrazuje JEDNU nebo VÍCE úrovní (zbytek stromu)

# Analogie v Linuxu:
/home/alice/**        # Všechno pod /home/alice (rekurzivně)
  ✓ /home/alice/documents
  ✓ /home/alice/photos
  ✓ /home/alice/photos/vacation/2025
  ✓ /home/alice/deep/nested/structure/file.txt

subscribe('lorawan/trackers/event123/#')

// Dostaneš VŠECHNY zprávy pod event123:
✓ lorawan/trackers/event123/tracker001/location
✓ lorawan/trackers/event123/tracker001/battery
✓ lorawan/trackers/event123/tracker001/status/online
✓ lorawan/trackers/event123/tracker002/location
✓ lorawan/trackers/event123/config/update
✓ lorawan/trackers/event123/stats/active-devices

--- 

GPS komplikácie 
pri frekvencií 1.5 GHz tu blokuje i sklo stena strom 
sklo okna blokuje GPS signál minimálne o 50 - 80 percent 
budova odráža signal 
satelity musia mať priamy výhľad 
gps anténa je smerovaná dole 
cold start trv=a dlho 
Signál je výborny RSSI: -42 dBm, SNR: 13.5 dB
GPS scan time bol príliš krátky - 60s nestačilo tak som ho rozšíril v configu appky na 120 

Battery: 50%
Firmware: 2.5
Hardware: 1.6
Work Mode: 1 (Periodic Mode) ✅
Positioning Strategy: 0 (GPS Only) ✅
Heartbeat Interval: 720 min
Periodic Interval: 5 min ✅
Event Interval: 60 min

"errMessage": "FAILED TO OBTAIN THE UTC TIMESTAMP"

Tracker odosiela 2 typy správ cez fPort 5 
správa bez gps - heartbeat 

{
  "Battery": 49%,
  "Firmware": "2.5",
  "Work Mode": 1,
  ...
}
  
GPS správa (s pozíciou)

{
  "Latitude": 49.821644,
  "Longitude": 18.161606,
  "Battery": 52%,
  "timestamp": ...
}

1. Tracker (T-1000-B)
   ↓ [LoRaWAN OTAA, 868 MHz]
   
2. WisGate Edge Pro (LoRaWAN Gateway)
   ↓ [4G/LTE → Internet]
   ↓ [MQTT publish na port 8883/TLS]
   
3. MQTT Broker (mqtt.hedurio.com)
   ↓ [Topic: application/SENSECAP/#]
   
4. Backend (Python Flask + Flask-SocketIO)
   ├─ MQTT Client (subscribe)
   │  ↓ Príjme správu
   │  ↓ Dekóduje Base64 → JSON
   │  ↓ Parse GPS súradnice (lat, lon)
   │  ↓ Uloží do PostgreSQL
   │  ↓
   ├─ WebSocket Server (Flask-SocketIO)
   │  ↓ Broadcast 'tracker-update' event
   │  ↓
   └─ REST API
      ├─ GET /api/trackers
      ├─ GET /api/tracker/:id
      └─ GET /api/tracker/:id/history
   
5. Frontend (React + Leaflet.js)
   ├─ WebSocket Client (Socket.IO-client)
   │  ↓ Počúva 'tracker-update' event
   │  ↓ Aktualizuje state
   │  ↓
   ├─ Mapa (Leaflet.js)
   │  ↓ Vykresli markery
   │  ↓ Real-time update pozícií
   │  ↓
   └─ Timeline Slider
      ↓ HTTP GET /api/tracker/:id/history?from=X&to=Y
      ↓ Zobrazí historické pozície
