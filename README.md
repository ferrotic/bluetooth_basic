> ### Das gesammte File wurde von KI erstellt. Es kann also Abweichungen und/oder fehlerhafte Inhalte geben. Du wurdest gewarnt


# Bluetooth – Gesamtüberblick: Architektur, Protokolle und Verbindungsabläufe

Bluetooth ist nicht „ein Protokoll“, sondern ein **ganzer Protokoll-Stack**. Wenn man Bluetooth sauber in Schichten betrachtet, werden auch Wireshark, `btmon`, nRF52840, ESP32, InternalBlue und spätere Security-Tests deutlich verständlicher.

Die wichtigste Grundidee ist:

```text
Anwendung
   ↓
Profile / Services
   ↓
Transport- und Anwendungsprotokolle
   ↓
L2CAP
   ↓
Link-Steuerung
   ↓
Funk / PHY
   ↓
2.4-GHz-Funksignal
```

Bluetooth existiert heute im Wesentlichen in zwei Welten:

```text
Bluetooth
│
├── Bluetooth Classic = BR/EDR
│   └── Kopfhörer, Lautsprecher, Auto, ältere Eingabegeräte usw.
│
└── Bluetooth Low Energy = BLE / LE
    └── Sensoren, IoT, Beacons, AirTags, Wearables usw.
```

Beide nutzen das 2,4-GHz-ISM-Band, funktionieren intern aber deutlich unterschiedlich.

---

# 1. Grundlegender Ablauf einer Bluetooth-Verbindung

Auf hoher Ebene läuft eine Bluetooth-Kommunikation typischerweise so ab:

1. Geräte finden sich
2. Eine Funkverbindung wird aufgebaut
3. Fähigkeiten werden ausgehandelt
4. Optional: Pairing
5. Optional: Schlüssel werden gespeichert
6. Verbindung wird verschlüsselt
7. Dienste oder Profile werden verwendet
8. Nutzdaten werden übertragen
9. Verbindung wird beendet

Wichtig:

**Connection ≠ Pairing ≠ Bonding ≠ Encryption**

| Begriff | Bedeutung |
|---|---|
| Connection | Zwei Geräte haben aktuell eine Funkverbindung |
| Pairing | Geräte erzeugen bzw. vereinbaren kryptografische Schlüssel |
| Bonding | Diese Schlüssel werden dauerhaft gespeichert |
| Authentication | Nachweis, dass die Gegenstelle den richtigen Schlüssel besitzt |
| Encryption | Funkverkehr wird verschlüsselt |

Ein Gerät kann also verbunden sein, ohne bereits gebondet zu sein.

---

# 2. Bluetooth-Architektur

Ein Bluetooth-System wird logisch in **Host** und **Controller** aufgeteilt.

```text
┌─────────────────────────────────────┐
│              APPLICATION            │
│                                     │
│ Apps / Audio / IoT / OS             │
├─────────────────────────────────────┤
│                 HOST                │
│                                     │
│ GAP                                 │
│ GATT / ATT      SDP / RFCOMM        │
│ SMP             AVDTP / AVCTP       │
│ L2CAP                               │
├─────────────────────────────────────┤
│                 HCI                 │
│       Host Controller Interface     │
├─────────────────────────────────────┤
│              CONTROLLER             │
│                                     │
│ LE Link Layer      BR/EDR Baseband  │
│                    LMP              │
│                                     │
│                 PHY                 │
└─────────────────────────────────────┘
                  │
                  ▼
               Antenne
```

Bei einem USB-Bluetooth-Dongle unter Kali sieht das ungefähr so aus:

```text
Kali Linux
│
│ BlueZ
│ L2CAP / ATT / GATT / SDP ...
│
├──────── HCI über USB ─────────┐
│                               │
                                ▼
                     Bluetooth USB Dongle
                     ├── Controller
                     ├── Link Layer
                     ├── Baseband
                     └── Radio
                                │
                                ▼
                              Funk
```

`btmon` sitzt an der **HCI-Grenze**.

---

# 3. HCI – Host Controller Interface

**HCI** verbindet Host und Bluetooth-Controller.

```text
HOST
 ↕
HCI
 ↕
CONTROLLER
```

Der Host kann zum Beispiel senden:

```text
HCI Command:
"Starte Scan"
```

Der Controller antwortet:

```text
HCI Event:
"Gerät gefunden"
```

Typische HCI-Datentypen sind:

```text
HCI Command
HCI Event
ACL Data
SCO Data
ISO Data
```

Mit `btmon` lassen sich beispielsweise sehen:

```text
Pairing Event
Encryption Change
Connection Complete
Remote Features
ACL Data
Disconnection Complete
```

`btmon` sieht aber nur die Kommunikation zwischen **deinem Host und deinem Bluetooth-Controller**.

Ein echter Funksniffer sitzt hingegen hier:

```text
Phone ───── Funk ───── Gerät
             ▲
             │
          Sniffer
```

---

# 4. Bluetooth Classic vs BLE

| | Bluetooth Classic | BLE |
|---|---|---|
| Bezeichnung | BR/EDR | LE |
| Kanäle | 79 | 40 |
| Kanalabstand | 1 MHz | 2 MHz |
| Hauptmodulation | GFSK | GFSK |
| weitere Modulation | π/4-DQPSK, 8DPSK | – |
| Basisrate | 1 Mbit/s | 1 Mbit/s |
| schnellere Variante | EDR 2/3 Mbit/s | LE 2M |
| Frequency Hopping | ja | ja |

---

# Bluetooth Classic / BR/EDR

# 5. BR und EDR

Bluetooth Classic heißt:

**BR/EDR = Basic Rate / Enhanced Data Rate**

BR:

```text
GFSK
1 Mbit/s
```

EDR:

```text
π/4-DQPSK → 2 Mbit/s
8-DPSK    → 3 Mbit/s
```

Classic verwendet **79 Kanäle** und springt ständig zwischen diesen.

```text
Zeit →

2402 MHz    X
2403 MHz                       X
2404 MHz          X
2405 MHz
2406 MHz                 X
...
2480 MHz
```

Das Verfahren heißt:

**FHSS – Frequency Hopping Spread Spectrum**

Bluetooth Classic verwendet Zeitschlitze von:

```text
625 µs
```

Dadurch ergibt sich nominal ein Hop-Rhythmus von:

```text
1600 Slots pro Sekunde
```

Das macht passives Bluetooth-Classic-Sniffing schwierig.

---

# 6. Inquiry – Geräte finden

Bluetooth Classic verwendet für Discovery hauptsächlich:

```text
Inquiry
```

Ein Gerät fragt sinngemäß:

> Welche Bluetooth-Geräte sind hier?

Andere Geräte antworten.

Dabei können Informationen auftauchen wie:

```text
BD_ADDR
Class of Device
Clock information
Name
Extended Inquiry Response
```

Die **BD_ADDR** ist die klassische 48-Bit-Bluetooth-Adresse.

Beispiel:

```text
AA:BB:CC:DD:EE:FF
```

---

# 7. Paging – Verbindung aufbauen

Wenn Gerät A Gerät B kennt, wird eine Verbindung über das:

**Page Procedure**

aufgebaut.

```text
Laptop
   │
   │ Inquiry
   ▼
Lautsprecher gefunden

Laptop
   │
   │ Page
   ▼
Lautsprecher
   │
   ▼
Bluetooth-Link
```

---

# 8. ACL

Normale Daten werden bei Bluetooth Classic meist über:

**ACL – Asynchronous Connection-Less**

transportiert.

Darüber laufen später beispielsweise:

```text
L2CAP
SDP
RFCOMM
AVDTP
AVCTP
A2DP
```

---

# 9. Baseband

Das Baseband kümmert sich unter anderem um:

```text
Zeitslots
Frequency Hopping
Paketformate
Fehlerkorrektur
Acknowledgements
ACL
SCO/eSCO
```

---

# 10. LMP – Link Manager Protocol

**LMP – Link Manager Protocol**

läuft zwischen den beiden Bluetooth-Controllern.

```text
Controller A
    │
    │ LMP
    ▼
Controller B
```

Darüber werden beispielsweise ausgehandelt:

```text
Features
Authentication
Encryption
Power Control
Rollen
Packet Types
Pairing-Funktionen
```

LMP ist für Tools wie **InternalBlue** und **BrakTooth** besonders interessant.

---

# 11. Classic Pairing

Früher gab es Legacy-Pairing, beispielsweise mit:

```text
0000
1234
```

Später kamen:

```text
Secure Simple Pairing (SSP)
Secure Connections
```

---

# 12. Secure Simple Pairing

SSP kennt mehrere Association Models:

| Methode | Beispiel |
|---|---|
| Just Works | Lautsprecher ohne Display |
| Numeric Comparison | beide Geräte zeigen dieselbe Zahl |
| Passkey Entry | PIN wird eingegeben |
| Out-of-Band | externe Authentisierung |

Wichtig:

**Just Works bietet normalerweise keinen MITM-Schutz.**

Die Verbindung kann trotzdem verschlüsselt sein.

---

# 13. Kryptografie bei Bluetooth Classic

Grob gibt es mehrere Sicherheitsgenerationen:

| Generation | Key Exchange | Authentication | Encryption |
|---|---|---|---|
| Legacy | SAFER+ | SAFER+ | E0 |
| Secure Simple Pairing | ECDH P-192 | SAFER+ | E0 |
| Secure Connections | ECDH P-256 | HMAC-SHA-256 | AES-CCM |

---

# 14. L2CAP

**L2CAP – Logical Link Control and Adaptation Protocol**

ist eine der wichtigsten Schichten.

```text
Application
    │
    ▼
RFCOMM / ATT / SDP / ...
    │
    ▼
L2CAP
    │
    ▼
Bluetooth Link
```

L2CAP übernimmt:

```text
Multiplexing
Segmentierung
Reassembly
logische Channels
```

Beispiel:

```text
                  ┌── SDP
                  │
Bluetooth ─ L2CAP ├── RFCOMM
                  │
                  ├── ATT
                  │
                  └── andere Protokolle
```

---

# 15. PSM

**PSM – Protocol/Service Multiplexer**

identifiziert Protokolle innerhalb von L2CAP.

L2CAP kann dadurch unterscheiden:

```text
SDP
RFCOMM
andere Protokolle
```

---

# 16. SDP – Service Discovery Protocol

**SDP** dient dazu, die Fähigkeiten eines Classic-Geräts herauszufinden.

Ein Lautsprecher könnte beispielsweise anbieten:

```text
Audio Sink
AVRCP
Hands-Free
Serial Port
```

Ein Client fragt:

> Welche Services hast du?

Das Gerät antwortet mit Service Records.

---

# 17. Bluetooth Profiles

Ein Profile beschreibt, wie mehrere Protokolle für einen bestimmten Zweck verwendet werden.

Beispiel:

```text
A2DP
```

steht für:

**Advanced Audio Distribution Profile**

---

# 18. Wichtige Classic-Protokolle und Profiles

| Name | Bedeutung | Verwendung |
|---|---|---|
| L2CAP | Logical Link Control and Adaptation Protocol | Basis-Transport |
| SDP | Service Discovery Protocol | Dienste entdecken |
| RFCOMM | serielle Port-Emulation | serieller Datenkanal |
| OBEX | Object Exchange | Dateien/Kontakte |
| AVDTP | Audio/Video Distribution Transport Protocol | Audio-Transport |
| AVCTP | Audio/Video Control Transport Protocol | Steuerbefehle |
| BNEP | Bluetooth Network Encapsulation Protocol | Netzwerk |
| HIDP | Human Interface Device Protocol | Tastatur/Maus |
| A2DP | Advanced Audio Distribution Profile | Musik |
| AVRCP | Audio/Video Remote Control Profile | Play/Pause/Lautstärke |
| HFP | Hands-Free Profile | Telefonie |
| HSP | Headset Profile | Headsets |
| SPP | Serial Port Profile | virtuelle serielle Verbindung |
| PAN | Personal Area Networking Profile | Netzwerk |

---

# 19. Beispiel: Musikstream

```text
Smartphone → Bluetooth-Lautsprecher
```

Der Stack sieht ungefähr so aus:

```text
Musik-App
   │
   ▼
A2DP
   │
   ▼
AVDTP
   │
   ▼
L2CAP
   │
   ▼
ACL
   │
   ▼
Baseband / RF
   │
   ▼
Lautsprecher
```

Zur Steuerung wird häufig zusätzlich:

```text
AVRCP
```

verwendet.

Beispiele:

```text
Play
Pause
Next
Previous
Volume
```

---

# 20. Telefonie

Bei Headsets wird häufig:

```text
HFP
```

verwendet.

Für Sprachdaten kommen traditionell:

```text
SCO
eSCO
```

zum Einsatz.

Merksatz:

```text
Musik:
A2DP → ACL

Telefonie:
HFP → SCO/eSCO
```

---

# Bluetooth Low Energy / BLE

# 21. BLE-Stack

```text
Application
    │
    ▼
GAP / GATT
    │
    ├── ATT
    └── SMP
          │
          ▼
        L2CAP
          │
          ▼
         HCI
          │
          ▼
      Link Layer
          │
          ▼
          PHY
```

---

# 22. BLE Physical Layer

BLE besitzt:

```text
40 Kanäle
2 MHz Kanalabstand
```

Drei Kanäle sind primäre Advertising Channels:

```text
Channel 37 → 2402 MHz
Channel 38 → 2426 MHz
Channel 39 → 2480 MHz
```

Die übrigen 37 werden als Datenkanäle verwendet.

```text
2402                                         2480 MHz
 │                                              │
 37  0 1 2 ... 10   38  11 12 ... 36          39
 ▲                  ▲                           ▲
 Advertising        Advertising                 Advertising
```

---

# 23. Advertising

Vor einer Verbindung sendet ein BLE-Gerät:

```text
Advertising
```

sinngemäß:

> Hallo, ich bin hier.

Beispiel:

```text
Smart Bulb
   │
   ├──── Channel 37
   ├──── Channel 38
   └──── Channel 39
```

Ein Scanner hört auf diesen Kanälen.

---

# 24. Advertising-Daten

Advertising kann enthalten:

```text
Device Name
Flags
Service UUIDs
Manufacturer Specific Data
Service Data
Tx Power
```

---

# 25. Legacy und Extended Advertising

BLE kennt:

```text
Legacy Advertising
Extended Advertising
Periodic Advertising
```

Extended Advertising kann zusätzliche Daten auf Secondary Channels verschieben.

```text
Channel 37
ADV_EXT_IND
     │
     ▼
Channel 17
AUX_ADV_IND
```

---

# 26. BLE-Verbindungsaufbau

```text
Peripheral                          Central

    │                                 │
    │──── Advertising ───────────────►│
    │                                 │
    │◄──── Connection Request ────────│
    │                                 │
    │========== Connection ===========│
```

Typisches Beispiel:

```text
Smartphone = Central
Smart Bulb = Peripheral
```

---

# 27. Connection Parameters

Beim Verbindungsaufbau werden unter anderem festgelegt:

```text
Access Address
CRC initialization
Connection interval
Peripheral latency
Supervision timeout
Channel map
Channel selection information
```

---

# 28. BLE Connection Events

BLE-Geräte kommunizieren nicht permanent.

Sie treffen sich zu sogenannten:

```text
Connection Events
```

Beispiel:

```text
Zeit ───────────────────────────────────►

Event        Event        Event        Event
  │            │            │            │
  ▼            ▼            ▼            ▼
 CH4          CH19         CH7          CH31
```

Dazwischen kann das Gerät schlafen.

---

# 29. Adaptive Frequency Hopping

BLE verwendet innerhalb einer Verbindung die 37 Datenkanäle.

Gestörte Kanäle können aus der Channel Map entfernt werden.

```text
Channel Map:

0 ✓
1 ✓
2 ✗
3 ✓
4 ✓
...
17 ✗
...
36 ✓
```

---

# 30. BLE Link Layer

Die **Link Layer** ist verantwortlich für:

```text
Advertising
Scanning
Connection establishment
Connection events
Channel hopping
Acknowledgements
Encryption
Connection parameter updates
PHY updates
```

---

# 31. LL Control Procedures

Typische Link-Layer-Control-Pakete sind:

```text
LL_FEATURE_REQ
LL_FEATURE_RSP

LL_ENC_REQ
LL_ENC_RSP

LL_START_ENC_REQ
LL_START_ENC_RSP

LL_TERMINATE_IND
```

Damit können beispielsweise:

```text
Features ausgehandelt
Encryption aktiviert
PHY geändert
Channel Maps aktualisiert
Verbindungen beendet
```

werden.

---

# 32. L2CAP bei BLE

Auch BLE verwendet L2CAP:

```text
Application
      │
      ▼
GATT / SMP
      │
      ▼
L2CAP
      │
      ▼
Link Layer
```

---

# 33. ATT – Attribute Protocol

**ATT – Attribute Protocol**

stellt Informationen als Attribute dar.

Ein Attribute besitzt beispielsweise:

```text
Handle
UUID
Value
Permissions
```

Beispiel:

```text
Handle:      0x0025
UUID:        2A19
Value:       64
Permissions: Read
```

---

# 34. ATT-Operationen

Typische Operationen:

```text
Read
Write
Write Without Response
Notify
Indicate
```

Unterschied zwischen Notification und Indication:

```text
Notification

Server ─────► Client
keine Bestätigung
```

```text
Indication

Server ─────► Client
       ◄──── Confirmation
```

Security-relevante Fragen:

```text
Ist Read erlaubt?
Ist Write erlaubt?
Ist Encryption erforderlich?
Ist Authentication erforderlich?
Ist Authorization erforderlich?
```

---

# 35. GATT – Generic Attribute Profile

GATT strukturiert ATT-Attribute.

```text
GATT Server
│
├── Service
│   │
│   ├── Characteristic
│   │      │
│   │      ├── Value
│   │      └── Descriptor
│   │
│   └── Characteristic
│
└── Service
```

Merksatz:

```text
ATT:
Wie werden Attribute übertragen?

GATT:
Wie werden Attribute logisch strukturiert?
```

---

# 36. Beispiel Smart Bulb

```text
Service
UUID: 12345678-...

├── Characteristic: Power
│   ├── Read
│   └── Write
│
├── Characteristic: Brightness
│   ├── Read
│   └── Write
│
└── Characteristic: Color
    └── Write
```

Ein Einschaltbefehl könnte beispielsweise sein:

```text
ATT Write Request
Handle 0x002A
Value 01
```

Security-Fragen:

```text
Kann jeder schreiben?
Ist Pairing nötig?
Ist Verschlüsselung nötig?
Wird der Client authentisiert?
```

---

# 37. UUIDs

Standardisierte UUIDs:

```text
180F = Battery Service
2A19 = Battery Level
```

Proprietäre Services verwenden meist 128-Bit-UUIDs:

```text
12345678-1234-5678-1234-56789abcdef0
```

---

# 38. GAP – Generic Access Profile

GAP beschreibt grundlegende Abläufe wie:

```text
Discovery
Advertising
Scanning
Connecting
Security
Privacy
```

Merksatz:

```text
GAP:
Wie finden und verbinden sich Geräte?

GATT:
Welche Dienste und Daten stellt das Gerät bereit?
```

---

# 39. SMP – Security Manager Protocol

BLE-Pairing wird hauptsächlich über:

**SMP – Security Manager Protocol**

abgewickelt.

```text
SMP
 │
 ▼
L2CAP
 │
 ▼
Link Layer
```

SMP kümmert sich um:

```text
Pairing
Authentication
Key generation
Key distribution
Bonding
```

---

# 40. BLE Pairing

Ein vereinfachter Ablauf:

```text
Gerät A                         Gerät B
   │                              │
   │──── Pairing Request ────────►│
   │◄─── Pairing Response ────────│
   │                              │
   │    IO Capabilities           │
   │    Security Requirements     │
   │                              │
   │──── Public Key ─────────────►│
   │◄─── Public Key ──────────────│
   │                              │
   │   Authentication             │
   │                              │
   │   Key Generation             │
   │                              │
   │====== Encryption ============│
```

---

# 41. LE Legacy Pairing vs Secure Connections

Es gibt:

```text
LE Legacy Pairing
```

und:

```text
LE Secure Connections
```

Modernes LE Secure Connections verwendet unter anderem:

```text
ECDH
NIST P-256
AES-CCM
```

---

# 42. BLE-Schlüssel

Wichtige Schlüssel:

```text
LTK
IRK
CSRK
```

## LTK – Long Term Key

Für Link-Verschlüsselung.

```text
LTK
  ↓
Link Encryption
```

## IRK – Identity Resolving Key

Für BLE Privacy und das Auflösen privater Adressen.

## CSRK – Connection Signature Resolving Key

Für signierte BLE-Daten.

---

# 43. Bluetooth Privacy

BLE kann statt einer dauerhaften Adresse verschiedene Random Addresses verwenden:

```text
Static Random Address
Resolvable Private Address
Non-Resolvable Private Address
```

Eine **Resolvable Private Address (RPA)** kann mithilfe des:

```text
IRK
```

einer bekannten Geräteidentität zugeordnet werden.

---

# 44. Pairing vs Bonding

Beim ersten Kontakt:

```text
Phone
  │
  ├── Connection
  ├── Pairing
  ├── Keys erzeugen
  └── Keys speichern
         ↑
       Bonding
```

Beim nächsten Kontakt:

```text
Phone
  │
  ├── Connection
  ├── gespeicherten Key verwenden
  └── Encryption starten
```

---

# 45. Kompletter BLE-Ablauf

```text
Smart Bulb                                  Smartphone
    │                                           │
    │──── Advertising ─────────────────────────►│
    │                                           │
    │◄──────── Connection Request ──────────────│
    │                                           │
    │========== Link-Layer Connection ==========│
    │                                           │
    │          Feature Exchange                 │
    │                                           │
    │────────────── L2CAP ──────────────────────│
    │                                           │
    │◄──────── SMP Pairing Request ─────────────│
    │───────── SMP Pairing Response ───────────►│
    │                                           │
    │           Key Generation                  │
    │                                           │
    │=========== AES-CCM Encryption ============│
    │                                           │
    │◄──────── ATT/GATT Discovery ──────────────│
    │                                           │
    │──────── Services ────────────────────────►│
    │                                           │
    │◄──────── Characteristic Read ─────────────│
    │──────── Characteristic Value ────────────►│
    │                                           │
    │◄──────── Characteristic Write ────────────│
    │                                           │
    │──────── Notification ────────────────────►│
```

---

# 46. Kompletter Bluetooth-Classic-Ablauf

```text
Laptop                                    Speaker
  │                                          │
  │──────── Inquiry ────────────────────────►│
  │◄────── Inquiry Response ─────────────────│
  │                                          │
  │──────── Page ───────────────────────────►│
  │                                          │
  │========== BR/EDR Link ===================│
  │                                          │
  │──────── LMP Feature Exchange ───────────►│
  │◄─────── LMP Response ────────────────────│
  │                                          │
  │──────── Secure Simple Pairing ──────────►│
  │                                          │
  │========== Authentication ================│
  │========== Encryption ====================│
  │                                          │
  │──────── L2CAP ──────────────────────────►│
  │                                          │
  │──────── SDP Service Discovery ──────────►│
  │◄────── A2DP / AVRCP Services ────────────│
  │                                          │
  │──────── AVDTP Setup ────────────────────►│
  │                                          │
  │========== A2DP Audio over ACL ===========│
  │                                          │
  │──────── AVCTP / AVRCP Commands ─────────►│
```

---

# 47. Protokollschichten im Vergleich

| Schicht | Bluetooth Classic | BLE |
|---|---|---|
| Anwendung | Audio, Apps | IoT, Sensoren, Apps |
| Profiles | A2DP, HFP, HID, SPP | GATT-basierte Profile |
| Service/Control | SDP, RFCOMM, AVDTP, AVCTP | GATT, ATT, SMP |
| Multiplexing | L2CAP | L2CAP |
| Host/Controller | HCI | HCI |
| Link Management | LMP | LL Control |
| Link | Baseband | Link Layer |
| Funk | BR/EDR PHY | LE PHY |
| Frequenzbereich | 2.4 GHz | 2.4 GHz |

---

# 48. Wo die Security-Tools sitzen

## Kali + USB Bluetooth Dongle

```text
Application
    │
BlueZ
    │
L2CAP / ATT / GATT / SDP
    │
HCI           ← btmon
    │
USB
    │
Bluetooth Controller
    │
RF
```

Geeignet für:

```text
Discovery
Pairing
Connections
GATT
SDP
HCI
Encryption Events
Profiles
```

---

## nRF52840

```text
                 RF
                  ▲
                  │
Device A ─────────┼──────── Device B
                  │
              nRF52840
                  │
              Wireshark
```

Geeignet für BLE:

```text
Advertising
Link Layer
Connection establishment
Channel activity
L2CAP
SMP
ATT
GATT
```

---

## ESP32 + BrakTooth

Geeignet für Bluetooth Classic:

```text
BR/EDR
Baseband
LMP
```

---

## Raspberry Pi + InternalBlue

Geeignet für:

```text
HCI
LMP
Controller Internals
Firmware
Link-Layer Behaviour
```

---

# 49. Wireshark bei BLE

Ein BLE-Paket kann in Wireshark ungefähr so zerlegt werden:

```text
Bluetooth Low Energy Link Layer
    ↓
L2CAP
    ↓
ATT
    ↓
Read Request
```

Logisch bedeutet das:

```text
RF-Paket
↓
LE Link Layer
↓
L2CAP
↓
ATT
↓
GATT / Anwendung
```

Das ist vergleichbar mit:

```text
Ethernet
↓
IP
↓
TCP
↓
HTTP
```

---

# 50. Wireshark bei Classic

Beispiel Audio:

```text
BR/EDR
 ↓
Baseband
 ↓
ACL
 ↓
L2CAP
 ↓
AVDTP
 ↓
A2DP
 ↓
Audio
```

Beispiel Headset-Steuerung:

```text
BR/EDR
 ↓
ACL
 ↓
L2CAP
 ↓
RFCOMM
 ↓
HFP Control
```

---

# 51. LE Audio

Modernes LE Audio verwendet zusätzliche Mechanismen:

```text
ISO
CIS
BIS
CIG
BIG
ISOAL
LC3
```

Vereinfacht:

```text
LE Audio
   │
   ▼
LC3 Codec
   │
   ▼
ISO / ISOAL
   │
   ▼
LE Link Layer
```

Dabei gilt:

```text
CIS = Connected Isochronous Stream
BIS = Broadcast Isochronous Stream
```

---

# 52. Bluetooth Mesh

Bluetooth Mesh basiert auf BLE, besitzt aber eine zusätzliche Netzwerkarchitektur.

```text
     B
    / \
   A   C
    \ /
     D
```

Zusätzliche Schichten sind unter anderem:

```text
Network Layer
Transport Layer
Access Layer
Models
```

Bluetooth Mesh ist daher nicht dasselbe wie eine normale GATT-Verbindung.

---

# 53. Security-Angriffsflächen

Bluetooth-Sicherheit kann auf mehreren Ebenen untersucht werden.

```text
PHY / RF
→ Sniffing
→ Jamming

Discovery / Advertising
→ Tracking
→ Information Leakage

Pairing / SMP / SSP
→ MITM
→ Downgrade
→ schwache Association

Link Layer / LMP
→ Protokollschwachstellen

GATT / ATT
→ fehlende Authentisierung
→ unsichere Read/Write Characteristics

Profiles
→ Implementierungsfehler

Application
→ Replay
→ fehlende Autorisierung
→ proprietäre Protokolle
```

---

# 54. Das wichtigste mentale Modell

Bei einem unbekannten Gerät zuerst fragen:

```text
Ist es Classic oder BLE?
        │
        ├───────────────┐
        ▼               ▼
     Classic           BLE
        │               │
   Inquiry/SDP      Advertising
        │               │
   BR/EDR Link       LE Link
        │               │
      LMP             SMP
        │               │
     L2CAP           L2CAP
        │               │
 SDP/RFCOMM/...      ATT
 AVDTP/AVCTP          │
        │             GATT
        ▼               ▼
     Profile         Service
        │               │
        ▼               ▼
   Anwendung        Anwendung
```

---

# 55. Wichtige Abkürzungen

| Kürzel | Bedeutung | Merksatz |
|---|---|---|
| BR | Basic Rate | Classic 1 Mbit/s |
| EDR | Enhanced Data Rate | schnelleres Classic |
| BLE / LE | Bluetooth Low Energy | energiesparendes Bluetooth |
| PHY | Physical Layer | Funkübertragung |
| LL | Link Layer | BLE-Link |
| LMP | Link Manager Protocol | Classic-Link-Verwaltung |
| HCI | Host Controller Interface | OS ↔ Bluetooth-Chip |
| ACL | Asynchronous Connection-Less | Classic-Datentransport |
| SCO/eSCO | Synchronous Connection | Sprache |
| L2CAP | Logical Link Control and Adaptation Protocol | Multiplexing / Transport |
| SDP | Service Discovery Protocol | Classic-Dienste finden |
| ATT | Attribute Protocol | BLE-Attribute |
| GATT | Generic Attribute Profile | BLE-Services |
| GAP | Generic Access Profile | Discovery / Connection |
| SMP | Security Manager Protocol | BLE-Pairing |
| RFCOMM | Radio Frequency Communication | virtuelle serielle Verbindung |
| AVDTP | Audio/Video Distribution Transport Protocol | Audio-Transport |
| AVCTP | Audio/Video Control Transport Protocol | Mediensteuerung |
| A2DP | Advanced Audio Distribution Profile | Musik |
| AVRCP | Audio/Video Remote Control Profile | Play/Pause |
| HFP | Hands-Free Profile | Telefonie |
| SSP | Secure Simple Pairing | Classic Pairing |
| LTK | Long Term Key | BLE-Verschlüsselung |
| IRK | Identity Resolving Key | BLE Privacy |
| CSRK | Connection Signature Resolving Key | Signaturen |

---

# 56. Zusammenfassung des gesamten Stacks

```text
                BLUETOOTH

       Classic             BLE
          │                 │
      BR/EDR PHY          LE PHY
          │                 │
      Baseband          Link Layer
          │                 │
         LMP                │
          │                 │
          └────── HCI ──────┘
                  │
                L2CAP
                  │
         ┌────────┴─────────┐
         │                  │
      Classic              BLE
         │                  │
     SDP/RFCOMM         SMP / ATT
     AVDTP/AVCTP            │
         │                 GATT
         │                  │
     A2DP/HFP          Services
         │                  │
         └──── Application ─┘
```

---

# 57. Zuordnung der eigenen Labortools

```text
USB Bluetooth Dongle + btmon
→ HCI / Host-Sicht

nRF52840
→ BLE Over-the-Air / Link Layer

ESP32 + BrakTooth
→ Bluetooth Classic / Baseband / LMP

Raspberry Pi + InternalBlue
→ Controller / HCI / LMP intern

Wireshark
→ Analyse und Dekodierung der Protokollschichten
```

---

# 58. Praktischer nächster Schritt

Nach dem Verständnis dieser Architektur sollte als Nächstes ein kompletter Verbindungsaufbau praktisch analysiert werden.

## BLE

```text
Advertising
→ Connection Request
→ Link Layer
→ L2CAP
→ SMP Pairing
→ Encryption
→ ATT
→ GATT
→ Characteristic Read/Write
```

## Bluetooth Classic

```text
Inquiry
→ Page
→ BR/EDR Link
→ LMP
→ Pairing
→ Encryption
→ L2CAP
→ SDP
→ AVDTP / RFCOMM
→ A2DP / HFP
```

Diese Abläufe lassen sich anschließend mit `btmon`, Wireshark, nRF52840, ESP32 und InternalBlue praktisch nachvollziehen.
