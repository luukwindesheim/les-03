# Week 4 — Storage: Ontwerpkeuzes en onderbouwing

## Casus

Het SSC bouwt een applicatie voor Gemeente Coevorden die:
- Een website host voor publieke dienstverlening
- Vergunningsaanvragen en inwonersgegevens opslaat in een database
- PDF-documenten opslaat (vergunningen, brieven, gescande documenten)
- SMS-codes verstuurt ter authenticatie bij inloggen

---

## Drie opslagtypes — waarom elk een andere oplossing

### 1. Blob Storage — ongestructureerde bestanden

**Waarvoor:** PDF's, gescande brieven, foto's van meldingen, exports.

**Waarom Blob en geen database:**
Een database slaat je geen bestanden in op — dat is traag en duur. Blob Storage is specifiek
ontworpen voor grote bestanden. De database slaat alleen een verwijzing op:
`pdfs/vergunning-12345.pdf`. De applicatie haalt het bestand zelf op via Blob Storage.

**Containers:**
- `pdfs` — vergunningsdocumenten en officiële brieven
- `documenten` — overige bestanden

### 2. SQL Database — gestructureerde data

**Waarvoor:** Inwonersregistraties, aanvragen, statussen, datums, gebruikersaccounts.

**Waarom SQL:**
Dit is data waar je vragen over wilt stellen. *Hoeveel aanvragen zijn er deze maand?
Welke inwoner woont op dit adres? Wat is de status van aanvraag 12345?*
Voor dit soort vragen heb je een relationele database nodig met tabellen en relaties.

**SKU: Basic (5 DTUs)**
Voor de labomgeving is Basic ruim voldoende. In productie zou je Standard of Premium
kiezen afhankelijk van het aantal gelijktijdige gebruikers.

### 3. Queue Storage — berichtenverkeer

**Waarvoor:** SMS-verificatieberichten bij authenticatie.

**Hoe het werkt:**
```
Webserver                    Queue (sms-authenticatie)     SMS-service
    │                               │                           │
    │  "stuur SMS 06-xxx code 4829" │                           │
    └──────────────────────────────►│                           │
                                    │   pakt bericht op         │
                                    │◄──────────────────────────┤
                                    │                           │ verstuurt SMS
```

**Waarom Queue en niet direct:**
De webserver hoeft niet te wachten tot de SMS verstuurd is. Als de SMS-service
even offline is gaan berichten niet verloren — ze wachten in de wachtrij.

**Opmerking over SMS:**
In productie zou Azure Communication Services de Queue uitlezen en de SMS versturen.
Die resource staat niet in de toegestane lijst van deze omgeving. De Queue zelf
demonstreert het principe volledig.

---

## Beveiliging van de opslag

**Storage Account:**
- Alleen HTTPS toegestaan (`supportsHttpsTrafficOnly: true`)
- Minimaal TLS 1.2
- Geen publieke toegang tot Blob containers (`publicAccess: None`)

**SQL Server:**
- Alleen verbindingen vanuit het app-subnet (10.0.3.0/24) toegestaan
- Azure-services mogen verbinding maken (voor beheer via portal)
- Minimaal TLS 1.2

---

## Kostenbeheer en right-sizing (knock-out criterium)

### Lopende kosten (schatting, regio West Europe)

| Resource | SKU / Config | Geschatte kosten/maand |
|---|---|---|
| Storage Account | Standard_LRS | ~€ 0,02 per GB opgeslagen data |
| SQL Server | — | Gratis (kosten zitten in database) |
| SQL Database | Basic, 5 DTU | ~€ 4,50 |
| **Totaal week 4** | | **~€ 5-10/maand** |
| **Totaal week 1 + 3 + 4** | | **~€ 285-330/maand** |

### Right-sizing keuzes

**Storage Account: Standard_LRS**
LRS (Locally Redundant Storage) slaat drie kopieën op binnen één datacenter.
Voor een labomgeving is dit voldoende. In productie voor een gemeente zou je ZRS
(Zone Redundant Storage) kiezen voor betere beschikbaarheid — maar dat kost meer.

**SQL Database: Basic**
5 DTUs is ruim genoeg voor de labomgeving. DTU staat voor Database Transaction Unit —
een maat voor de rekenkracht van de database. In productie zou je op basis van het
verwachte aantal gelijktijdige gebruikers kiezen voor Standard S1 of hoger.

---

## Hoe te deployen

Week 4 is een uitbreiding op week 1 en 3 — hetzelfde script voor alles:

```bash
cd /mnt/c/Users/Student/Documents/CTI/Opdrachten/week1
bash scripts/deploy.sh
```

---

## Bestandsstructuur

```
week4/
├── modules/
│   ├── storage.bicep       ← Storage Account met Blob containers + Queue
│   └── sqlserver.bicep     ← SQL Server + Database
└── UITLEG.md               ← dit document
```

---

## Videodemonstratie — wat te laten zien

### Criterium: Toon de verschillende typen data en leg uit waarvoor

Ga naar `portal.azure.com` → `S1500487`.

**Storage Account (Blob):**
1. Klik op het storage account (naam begint met `ssc...`)
2. Ga naar **Containers** — toon `pdfs` en `documenten`
3. Zeg: *"Hier slaat de applicatie PDF-documenten op. Een inwoner uploadt een bijlage
   bij een vergunningsaanvraag — die PDF komt in de `pdfs` container terecht.
   De database slaat alleen het pad op, niet het bestand zelf."*

**Storage Account (Queue):**
1. Ga naar **Queues** — toon `sms-authenticatie`
2. Zeg: *"Dit is de wachtrij voor SMS-berichten. Als een inwoner inlogt zet de
   applicatie hier een bericht in. Een aparte service leest die wachtrij en
   verstuurt de SMS. De applicatie hoeft daar niet op te wachten."*

**SQL Database:**
1. Klik op de SQL Server (`ssc-zwolle-sql-...`)
2. Ga naar **Databases** — toon `ssc-zwolle-db`
3. Zeg: *"Hier staan de gestructureerde gegevens: inwonersregistraties, aanvragen,
   statussen. Dit is data waar je vragen over wilt stellen — vandaar een SQL database
   en geen blob storage."*

### Criterium: Demonstreer de werking van de scripts

Toon in de terminal:
```bash
bash scripts/deploy.sh
```

Vertel: *"Met één script worden alle opslagresources aangemaakt. De Storage Account,
de containers, de Queue en de SQL database worden automatisch ingericht.
Zo kan het SSC voor elke nieuwe gemeente in één commando de volledige omgeving uitrollen."*
