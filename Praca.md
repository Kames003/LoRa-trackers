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


OTAA pri štate kazdeho join requestu sa vygeneruju dva zakladne seshion kluče - network session key - network session key - NwkSkey a application session key AppsKey 

1. vychodiskove parametre v zariadeni ED - End Device napr, čo je v našom trackery 

DevEUI - 64 bit unikatne ID zariadenia 
JoinEUI - AppEUI - 64 bitove ID aplikacie 
AppKey  - 128 bitovy root kluč - musi ostat v sieti a v zariadeni v tajnosti 

Join request 

z End device na network server 

tracker vygeneruje 16 bitovy devNonce - je to nahodne čislo zamedzi replay utokom 
+ do join requestu ramca zapíše : JoinEUI, DevEUI, DevNonce + počíta CRC - MIC 

toto pošle do gateway -- network server 

ak je join request validny 

do join accept ramca zabali JoinNonce, DevAddr, DLSettings --> Aktorý je následne ES-ENCRYPTovaný kľúčom AppKey:

ED ho prijme, decryptuje (rovnaký AppKey) a prečíta JoinNonce, DevAddr atď.

účel klučov 

NwkSKey : 

overovanie a ochranu mac vrsty 
signuje / vypočítava všetky uplink aj downlink správy 
šifruje a dešifruje len sieťove príkazy (FOpts) alebo payload, ak FPort=0


Application session key 

výhradne na end to end šifrovanie aplikačných dat 
zašifruje frmpayload pri FPort greater than 1 na uplinku aj downlinku 
• Sieť (NS) iba preposiela zašifrované dáta aplikačnému serveru, ten ich dešifruje pomocou AppSKey

mosquitto_sub -h mqtt.hedurio.com -u hedurio -P qn3nyMYUPHTiigLrV94JRiAUXauE9F -t 'application/#' -v -p 8883

prikaz otvorí MQTT‐subscriber na heduriackom brokri a vypíše ti do terminálu všetko, čo sa na témy matching „application/#“ objaví. Inými slovami:

mosquitto_sub – spúšťa MQTT klienta v móde „subscribe“
-h mqtt.hedurio.com – hostname brokeru
-p 8883 – port (8883 je štandardne TLS/SSL)
-u hedurio – užívateľské meno
-P qn3nyMYUPHTiigLrV94JRiAUXauE9F – heslo
-t 'application/#' – wildcard téma, odchyť všetko pod „application/…“
-v – verbose, tzn. vypíšeš nielen payload, ale aj názov témy

Výsledok: v reálnom čase uvidíš v termináli každú správu (téma + payload), ktorú ti brána pošle na heduriacky MQTT broker. Príkaz nič neukladá ani nepresmerováva ďalej – slúži len na odpočúvanie (debug/monitoring). Pre produkciu/parsovanie si potom vytvoríš vlastného subscriber-clienta (napr. v Node.js, Pythone...) a správy po odchytení uložíš či odošleš do DS.

# Ako vyzerá mqqt arch ? 

broker 

– centrálny server, ktorý prijíma od publisherov správy a doručuje ich subscriberom
– stará sa o smerovanie („routing“) podľa topic‐reťazca, QoS, udržiavanie stavov session, TTL, LWT atď.

Publisher (vydavateľ)
– akýkoľvek „klient“ (senzor, aplikácia…), ktorý dokáže správy (payload) odoslať na broker
– pri odoslaní definuje:
• topic (napr. sensors/temperature/room1)
• QoS (0, 1 alebo 2 – garancia doručenia)
• retained flag (či broker má túto správu ponechať ako „poslednú platnú“)

Subscriber (odoberateľ)
– klient, ktorý sa prihlási k jednému alebo viacerým topic-filterom (napr. sensors/# alebo actuators/+/set)
– broker mu následne doručuje každú publikovanú správu, ktorá sa zhoduje s ľubovoľným jeho filterom
– pri prihlásení si tiež môže určiť požadovaný QoS úroveň


Našim „publisherom“ sú IoT trackery na end‐device (LoRa senzory).
Trackery zašlú uplink cez LoRaWAN bránu do Hedurio network servera, ktorý vystavuje MQTT broker.
Trackery publikujú napr. na topic
application/123/device/456/uplink
s QoS=1, payloadom JSONu: {… GPS, timestamp …}.
My (backend/subscriber) sa prihlásime k topic-filtru
application/123/+/uplink
alebo jednoducho
application/#
a broker nám začne posielať všetky uplinky zo všetkých device‐id pod application/123.
Backend dokáže spracovať doručené správy (parsovať JSON, ukladať do db, posielať notifikácie…).

Dôležité vlastnosti MQTT:

– Hierarchický názov topicu rozdelený lomítkami
– Wildcardy: „+“ (jedna úroveň), „#“ (viac úrovní)
– QoS 0,1,2 pre riadenie spoľahlivosti
– Retained správy pre uchovávanie poslednej hodnoty
– Clean vs. persistent sessions (uloženie nevyzdvihnutých správ pri odpojení)
– Šifrovanie/TLS, autentifikácia (užívateľ/heslo, certifikáty)

– Senzor = publisher → broker
– Broker = Hedurio MQTT server → distribúcia podľa topic
– Backend/appka/aktory = subscriber (alebo aj publisher downlinku) → spracovanie či odoslanie príkazov

GeoFences je z PostGIS = virtuálna geografická zóna 

Príklad: Hudobný festival
┌─────────────────────┐
│   FESTIVAL AREA     │  ← Geofence (polygon)
│                     │
│  🎵 Stage          │
│  🍺 Bar            │
│  🚻 WC             │
│                     │
│  👤 Tracker1 INSIDE │  ✅ OK
└─────────────────────┘
         👤 Tracker2 OUTSIDE  ⚠️ ALERT!

Alert keď niekto:

Opustí povolenú zónu (dieťa sa stratilo)
Vstúpi do zakázanej zóny (VIP area bez prístupu)
Prejde do nebezpečnej zóny (backstage, strojovňa)

Napr. 

Festival ma 3sceny ( Stage A,B,C )
- Dashboard zobrazí: "Stage A: 245 ľudí, Stage B: 189 ľudí..."
Heatmapy a štatistiky 
time based geofencing 


vytvorenie geofence okolo podia 
- alert ked niekto opusti zónu 
detekcia kto je vo vnútri geofence 
detekcia kto prave opustil geofence 

 Continuous Aggregates

❌ Ak máte < 10,000 záznamov  
❌ Ak dashboard refreshujú len 1-2 ľudia občas  
❌ Ak nepotrebujete agregácie (stačí len "kde je tracker práve teraz")
 
**Pre váš projekt:**
- **Zatiaľ nepotrebujete** - máte `latest_positions` view
- Pridáte neskôr ak budete robiť: "Štatistiky pohybu za posledných 30 dní"

---

## 3️⃣ Trackovanie dlhšej trasy - čo tým myslím?

Áno! **PostGIS LINESTRING** = spojenie bodov do čiary (trasa pohybu)

### **Čo je to?**
```
GPS pozície (POINT):
  👤 → 👤 → 👤 → 👤 → 👤

Trasa (LINESTRING):
  👤━━━👤━━━👤━━━👤━━━👤
  ╰────────────────────╯
      Jedna línia!
```


Príklad hodne cool 

```cpp

-- Získaj trasu trackera za posledných 6 hodín
SELECT 
    device_id,
    ST_AsGeoJSON(
        ST_MakeLine(location::geometry ORDER BY timestamp)
    )::json AS trail_geojson,
    MIN(timestamp) AS start_time,
    MAX(timestamp) AS end_time,
    COUNT(*) AS point_count
FROM positions
WHERE device_id = 'TestingTracker2'
  AND timestamp > NOW() - INTERVAL '6 hours'
GROUP BY device_id;

```

Output

{
  "type": "LineString",
  "coordinates": [
    [18.1614, 49.8216],
    [18.1618, 49.8215],
    [18.1622, 49.8214],
    [18.1625, 49.8212]
  ]
}

frontend 
// Vykreslenie trasy na mape
L.geoJSON(trail_geojson, {
    style: { color: '#FF0000', weight: 3 }
}).addTo(map);
```

Výsledok:
```
Mapa:
┌─────────────────────────┐
│                         │
│      🗺️                 │
│       ╱                 │
│      ╱  Trasa trackera │
│     ╱   (červená čiara)│
│    👤 ← aktuálna pozícia│
│                         │
└─────────────────────────┘


Analýza pohybu

Celková vzdialenosť prejdená za deň:

WITH trail AS (
    SELECT ST_MakeLine(location::geometry ORDER BY timestamp) AS line
    FROM positions
    WHERE device_id = 'TestingTracker2'
      AND timestamp::date = CURRENT_DATE
)
SELECT 
    ST_Length(line::geography) / 1000.0 AS distance_km
FROM trail;

priemerná rýchlost 

WITH movement AS (
    SELECT 
        timestamp,
        location,
        LAG(location::geometry) OVER (ORDER BY timestamp) AS prev_location,
        LAG(timestamp) OVER (ORDER BY timestamp) AS prev_time
    FROM positions
    WHERE device_id = 'TestingTracker2'
      AND timestamp > NOW() - INTERVAL '1 hour'
)
SELECT 
    AVG(
        ST_Distance(location::geometry, prev_location::geography) /  -- metre
        EXTRACT(EPOCH FROM (timestamp - prev_time)) * 3.6  -- km/h konverzia
    ) AS avg_speed_kmh,
    MAX(
        ST_Distance(location::geometry, prev_location::geography) /
        EXTRACT(EPOCH FROM (timestamp - prev_time)) * 3.6
    ) AS max_speed_kmh
FROM movement
WHERE prev_location IS NOT NULL;
```

**Output:**
```
avg_speed_kmh: 4.2 km/h  (prechodzka)
max_speed_kmh: 8.5 km/h  (rýchla chôdza)

kde stál tracker dlhšie než 5 minut 


WITH stationary_points AS (
    SELECT 
        timestamp,
        location,
        LAG(location::geometry) OVER (ORDER BY timestamp) AS prev_location,
        timestamp - LAG(timestamp) OVER (ORDER BY timestamp) AS time_diff
    FROM positions
    WHERE device_id = 'TestingTracker2'
      AND timestamp::date = CURRENT_DATE
)
SELECT 
    timestamp,
    ST_AsText(location::geometry) AS position,
    time_diff
FROM stationary_points
WHERE prev_location IS NOT NULL
  AND ST_Distance(location::geometry, prev_location::geography) < 50  -- < 50m pohyb
  AND time_diff > INTERVAL '5 minutes'
ORDER BY timestamp;
```

**Output:**
```
timestamp            | position                  | time_diff
---------------------|---------------------------|----------
2025-10-22 09:30:00  | POINT(18.1614 49.8216)   | 00:12:00
2025-10-22 14:15:00  | POINT(18.1625 49.8212)   | 00:08:30
```

→ "Tracker stál pri súradniciach X,Y na 12 minút" (napr. obed, pauza)

---

#### **C) Timeline s trasou (váš use-case!)**
```
Timeline slider:
[=====|=============]
      ↑
  08:00-12:00

Mapa zobrazuje:
- Aktuálne pozície (všetky trackery)
- Trasa vybraného trackera (08:00-12:00) jako červená čiara


Heat Maps 
Upozorniť ked sa dve osoby priblížia 
movements patterns ( analýza vzorcov pohybu ) 
movement patterns - Benefit: "90% ľudí ide z hlavného vchodu cez Bar k Stage A" → optimalizácia infraštruktúry!
speed zones - detekcia abnormalnej rýchlosti - Alert keď niekto beží (možný incident/panika)
Dwell time analysis - čas strávený na mieste - kolko ludia stravia času na mieste - pri food trucku ludia stoja primerne 8 minut -- pridaj ďalši truck
lost person detection - ziskaj poslednu znamu poziciu a priemernu rýchlost/smer - vypočítaj pravdepodobnú poziciu - následne zobraz search area na mape a napr alert nejbliýší stufftracker 


Čo sa týka TimescaleDB - minutové štatistiky ( pre lives dashboards ), auto refresh policy každých 30 sekund o výsledku dashborad querry bude super fast 

compression 
- automatická kompresia starých dát 
dáta staršie ako 7 dni --> komprimované čo nam da usportu miesta radovo o 90 percent 
nasledne dáta staršie ako napr 30 dni vymazať 
DOWNSAMPLING = po 30 dnoch, zmaž detailne data, ponechat len hodinove agregácie 
timesclaedb automaticky vytvára chunks 

# Podstatné príkazy pre Docker compose 

# ✅ SPUSTIŤ kontajnery (používa existujúce images)
docker compose up -d

# ✅ ZASTAVIŤ kontajnery (nepouštia, data zostávajú)
docker compose stop

# ✅ ZNOVA SPUSTIŤ zastavené kontajnery
docker compose start

# ✅ REŠTARTOVAŤ bežiace kontajnery
docker compose restart

# ✅ ZASTAVIŤ A ODSTRÁNIŤ kontajnery (data v volumes ZOSTÁVAJÚ!)
docker compose down

# ⚠️ ZASTAVIŤ, ODSTRÁNIŤ kontajnery A VOLUMES (VYMAŽE DATA!)
docker compose down -v

Čo pouzívať : 

Prvé spustenie : docker compose up -d 
zastaviť dočasne : docker compose stop 
znova spustit : docker compose start 
reštart po zmene kodu : docker compose restart 
zastavit a odstranit kontajnery : docker compose down 
vyčistit všetko vrátane dát : docker compose down -v 


# Sledovať logy databázy
docker-compose logs -f timescaledb

# Sledovať logy všetkých služieb
docker-compose logs -f

# Status kontajnerov
docker-compose ps

# Zobraziť resources (CPU, RAM)
docker stats


# Vyčistiť staré/nepoužívané images
docker image prune -a

# Vyčistiť všetko nepoužívané (images, volumes, networks)
docker system prune -a --volumes

# Zobraziť veľkosť volumes
docker system df


docker exec -it tracker-db psql -U tracker_user -d tracker_db = priamo sa pripojiť na ds 

SELECT * FROM devices;

REFRESH MATERIALIZED VIEW
SELECT * FROM latest_positions;

\q 

Testoavanie databázy : 

REFRESH MATERIALIZED VIEW latest_positions;

2. Aktuálne pozície:

SELECT * FROM latest_positions;

3. Vzdialenosť od centra Bílovce:

SELECT 
    lp.device_id,
    d.device_name,
    lp.latitude,
    lp.longitude,
    ST_Distance(
        lp.location,
        ST_SetSRID(ST_MakePoint(18.0155, 49.7564), 4326)::geography
    ) / 1000.0 AS distance_km
FROM latest_positions lp
JOIN devices d ON lp.device_id = d.device_id;

Trasa trackera (LINESTRING)

SELECT 
    device_id,
    ST_AsText(ST_MakeLine(location::geometry ORDER BY timestamp)) AS trail_linestring,
    COUNT(*) AS point_count,
    MIN(timestamp) AS start_time,
    MAX(timestamp) AS end_time
FROM positions
WHERE device_id = '2cf7f1c05300063d'
GROUP BY device_id;

Vzdialenosť medzi po sebe idúcimi pozíciami:

WITH position_changes AS (
    SELECT 
        id,
        timestamp,
        location,
        LAG(location) OVER (ORDER BY timestamp) AS prev_location
    FROM positions
    WHERE device_id = '2cf7f1c05300063d'
)
SELECT 
    id,
    timestamp,
    ST_Distance(location, prev_location) AS distance_meters,
    EXTRACT(EPOCH FROM (timestamp - LAG(timestamp) OVER (ORDER BY timestamp))) AS time_diff_seconds
FROM position_changes
WHERE prev_location IS NOT NULL
ORDER BY timestamp;

 Database štatistiky:

 SELECT 
    'Devices' as table_name, COUNT(*) as count FROM devices
UNION ALL
SELECT 'Positions', COUNT(*) FROM positions
UNION ALL
SELECT 'Positions (24h)', COUNT(*) FROM positions WHERE timestamp > NOW() - INTERVAL '24 hours';

TimescaleDB chunks:

SELECT 
    chunk_name,
    range_start,
    range_end,
    pg_size_pretty(total_bytes) as size,
    compression_status
FROM timescaledb_information.chunks
WHERE hypertable_name = 'positions'
ORDER BY range_start DESC;
