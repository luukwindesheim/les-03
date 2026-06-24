# Week 3 — Security: Ontwerpkeuzes en onderbouwing

## Casus

Gemeente Coevorden heeft besloten al haar publieke diensten te migreren van on-premise naar Azure.
De uitdaging is om dit veilig te doen met een centrale firewall die:

- Al het uitgaande verkeer inspecteert en filtert
- Alleen bekende en goedgekeurde bestemmingen toestaat
- Firewallregels automatisch kan uitrollen via een script
- Volledig aansluit op de bestaande infrastructuur van het SSC (week 1)

---

## Verschil tussen NSG (week 1) en Azure Firewall (week 3)

In week 1 zijn Network Security Groups ingericht op subnetniveau. Dit is basisbeveiliging.
Week 3 voegt een centrale Next-Generation Firewall toe die verder gaat:

| | NSG (week 1) | Azure Firewall (week 3) |
|---|---|---|
| Niveau | Per subnet / NIC | Centraal voor het hele netwerk |
| Laag | Layer 3/4 (IP + poort) | Layer 7 (ook domeinnamen) |
| Voorbeeld regel | Blokkeer poort 22 | Blokkeer `*.facebook.com` |
| Logging | Beperkt | Volledig (wie, wat, wanneer) |
| Threat intelligence | Nee | Ja (Basic: beperkt) |

Ze vervangen elkaar niet — ze vullen elkaar aan. NSGs zijn de eerste verdedigingslinie per
subnet, de Firewall is de centrale poortwachter voor al het netwerkverkeer.

---

## Architectuur

```
Internet
    │
    ▼
┌─────────────────────────────────┐
│  Azure Firewall Basic           │  ← AzureFirewallSubnet          10.0.6.0/26
│  Centrale netwerkbeveiliging    │    AzureFirewallManagementSubnet 10.0.7.0/26
└─────────────────┬───────────────┘
                  │ Filtert al het in- en uitgaande verkeer
                  ▼
┌─────────────────────────────────┐
│  Bestaand netwerk week 1        │
│  snet-web / snet-app / snet-data│
└─────────────────────────────────┘
```

De twee firewallsubnets zijn een Azure-platformvereiste:
- `AzureFirewallSubnet` — voor het dataverkeer (naam exact vereist door Azure)
- `AzureFirewallManagementSubnet` — voor beheersverkeer (vereist bij Basic SKU)

Het `deploy.sh` script controleert of deze subnets al bestaan en maakt ze alleen aan als dat
nog niet het geval is — zo wordt het bestaande VNet van week 1 niet verstoord.

---

## Automatisch importeren van firewallregels

Het sleutelconcept van week 3 is dat de regels **gescheiden zijn van de Bicep-code**.

```
week3/firewall-rules.json   ← bevat alle regels (aanpasbaar zonder Bicep te kennen)
        │
        └── week1/scripts/deploy.sh leest dit bestand
                │
                └── az deployment group create --parameters @firewall-rules.json
                            │
                            └── Bicep deployt de regels naar Azure Firewall
```

**Workflow voor het toevoegen van een regel:**
1. Open `week3/firewall-rules.json`
2. Voeg de regel toe aan de juiste collectie
3. Voer `bash scripts/deploy.sh` opnieuw uit vanuit de week1 map
4. De regel staat binnen ~10 minuten actief in Azure

Dit is precies wat "automatisch importeren" betekent: geen handmatig klikken in de portal,
maar een JSON-bestand aanpassen en het script aftrappen.

---

## Regelcollecties

### Netwerk-regels (Layer 3/4 — IP en poort)

| Collectie | Regel | Doel |
|---|---|---|
| allow-infrastructure | Allow-DNS | DNS-opzoekingen naar Azure DNS (168.63.129.16) |
| allow-infrastructure | Allow-NTP | Tijdssynchronisatie voor servers (poort 123) |
| allow-coevorden-services | Allow-Web-To-Internet-HTTPS | Weblaag mag HTTPS naar buiten |
| allow-coevorden-services | Allow-App-To-Web | App-laag communiceert met weblaag |
| deny-all | Deny-Internet-Outbound | Blokkeer al het overige verkeer |

### Applicatie-regels (Layer 7 — domeinnamen)

| Collectie | Regel | Toegestane domeinen |
|---|---|---|
| allow-windows-update | Allow-WindowsUpdate | *.windowsupdate.com, *.update.microsoft.com |
| allow-azure-services | Allow-Azure-Management | *.azure.com, *.microsoftonline.com |
| allow-government | Allow-Dutch-Government | *.overheid.nl, *.rijksoverheid.nl, *.digid.nl |

**Waarom deze regels voor Gemeente Coevorden?**

Gemeente Coevorden migreert publieke diensten naar Azure. De servers hebben toegang nodig tot:
- Windows Update (beveiligingsupdates voor alle servers)
- Azure-diensten (voor cloud management en monitoring)
- Nederlandse overheidsdiensten (DigiD, Rijksoverheid) voor de publieke dienstverlening

Al het andere internetverkeer wordt geblokkeerd — dit is het principe van **least privilege**:
alleen wat expliciet nodig is mag door.

---

## Prioriteiten van regels

Azure Firewall verwerkt regels op volgorde van prioriteit (laag getal = hogere prioriteit):

```
Prioriteit 100 — allow-infrastructure      (DNS, NTP — altijd nodig)
Prioriteit 200 — allow-coevorden-services  (applicatieverkeer)
Prioriteit 900 — deny-all                  (alles wat overblijft blokkeren)
```

De `deny-all` op prioriteit 900 zorgt ervoor dat verkeer dat niet expliciet is toegestaan
automatisch geblokkeerd wordt. Dit heet **defense-in-depth**.

---

## Kostenbeheer en right-sizing (knock-out criterium)

### Lopende kosten (schatting, regio West Europe)

| Resource | SKU / Config | Geschatte kosten/maand |
|---|---|---|
| Azure Firewall | Basic | ~€ 90 |
| Public IP (dataverkeer) | Standard, Static | ~€ 3,60 |
| Public IP (beheer) | Standard, Static | ~€ 3,60 |
| Firewall Policy | Basic | Gratis |
| **Totaal week 3** | | **~€ 97/maand** |
| **Totaal week 1 + 3 samen** | | **~€ 280-320/maand** |

### Right-sizing keuze: Basic SKU

De Basic SKU is gekozen omdat:
- Het de enige toegestane SKU is voor Azure Firewall in deze omgeving
- Basic ondersteunt netwerk- en applicatieregels — voldoende voor de casus
- Standard/Premium bieden extra features (IDPS, TLS-inspectie) maar kosten ~€ 900+/maand
- Voor Gemeente Coevorden als gemeente is Basic proportioneel aan de schaal van de dienstverlening

**Aanbeveling na behalen voldoende:**
De Azure Firewall is samen met de Application Gateway de grootste kostenpost.
Verwijder de resource group zo snel mogelijk:
```bash
az group delete --name S1500487 --yes
```

---

## Hoe te deployen

Week 3 is een uitbreiding op week 1 — er is maar één deploy script voor alles:

```bash
cd /mnt/c/Users/Student/Documents/CTI/Opdrachten/week1
bash scripts/deploy.sh
```

Het script regelt automatisch:
1. SSL-certificaat genereren (week 1)
2. Firewallsubnets aanmaken als ze nog niet bestaan (week 3)
3. Validatie van de volledige template
4. Deployment van week 1 + week 3 samen naar resource group `S1500487`

**Firewallregels updaten:**
```bash
# 1. Pas week3/firewall-rules.json aan
# 2. Voer het script opnieuw uit vanuit de week1 map
bash scripts/deploy.sh
```

---

## Bestandsstructuur

```
week1/
├── main.bicep                  ← bevat week 1 én roept week 3 modules aan
├── scripts/
│   └── deploy.sh               ← één script voor de volledige deployment
└── modules/                    ← week 1 modules

week3/
├── firewall-rules.json         ← de te importeren firewallregels (pas dit aan)
├── modules/
│   ├── firewall-policy.bicep   ← Firewall Policy + regelcollecties
│   └── firewall.bicep          ← Azure Firewall Basic + publieke IPs
└── UITLEG.md                   ← dit document
```

---

## Videodemonstratie — wat te laten zien

### Criterium: Demonstreer wat de firewallregels doen

1. Ga naar **portal.azure.com** → resource group `S1500487`
2. Klik op `ssc-zwolle-fw-policy`
3. Ga naar **Rule collections** in het linkermenu
4. Toon de drie netwerk-regelcollecties: lees de namen en leg uit wat elke collectie doet
5. Toon de drie applicatie-regelcollecties: wijs op de FQDNs en leg uit waarom juist deze domeinen
6. Vertel: *"De deny-all op prioriteit 900 blokkeert al het verkeer dat niet expliciet is toegestaan.
   Dit is least privilege — een overheidsprincipe dat ook voor ICT-beveiliging geldt."*

### Criterium: Demonstreer de werking van de scripts

1. Toon `week3/firewall-rules.json` in een editor
2. Voeg live een nieuwe regel toe, bijvoorbeeld:
   ```json
   {
     "name": "Allow-DigiD-Extra",
     "description": "Extra DigiD endpoint voor burgerportal",
     "sourceAddresses": ["10.0.2.0/24"],
     "targetFqdns": ["*.digid.nl"],
     "protocols": [{ "protocolType": "Https", "port": 443 }]
   }
   ```
3. Sla het bestand op
4. Trap het script af vanuit de terminal:
   ```bash
   bash scripts/deploy.sh
   ```
5. Wacht tot de deployment klaar is (~10 minuten)
6. Ga terug naar de portal en toon dat de nieuwe regel zichtbaar is

### Criterium: Leg de configuratie uit en waarom toepasbaar op de casus

Vertel in de video:
- *"Gemeente Coevorden migreert publieke diensten naar Azure. Ik heb gekozen voor Azure Firewall
  Basic omdat dit de enige toegestane SKU is én omdat het voldoet aan de beveiligingseisen."*
- *"De NSGs uit week 1 regelen de segmentatie per subnet. De Firewall voegt daar bovenop centrale
  inspectie toe op domeinnaamsniveau — dat kan een NSG niet."*
- *"De regels zijn bewust opgesplitst: infrastructuur (DNS/NTP), Coevorden-services en een
  expliciete deny-all. Zo is direct zichtbaar waarvoor elke regel dient."*
- *"Door de regels in een apart JSON-bestand te zetten kan een beheerder zonder Bicep-kennis
  regels aanpassen en uitrollen."*
