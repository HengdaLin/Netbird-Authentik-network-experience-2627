# Technische Documentatie: NetBird Configuratie voor VLAN-toegang en Automatisatie

## 1. Doel en Overzicht
Dit document beschrijft de configuratie van **NetBird** (`https://netbird.test.lab`, Management `v0.83.1` / Dashboard `v2.97.0`) om studenten op afstand veilig en gecontroleerd toegang te geven tot specifieke VLAN-subnets in de Proxmox/VIVES-omgeving.

Door gebruik te maken van **NetBird Networks**, **Groups**, **Access Control Policies** en een **Routing Peer** hoeft de NetBird-client niet op elke individuele studenten-VM te worden geïnstalleerd. Het automatiseringsportaal hoeft een student (of diens eenmalige Setup Key) enkel aan de juiste VLAN-groep toe te voegen.

---

## 2. API Access Token aanmaken
Om vanuit het automatiseringsportaal automatisch unieke Setup Keys aan te maken en studenten aan VLAN-groepen te koppelen, is een Access Token aangemaakt.

1. Navigeer naar **Users** en selecteer het admin-account (`adminHengda@test.lab`).
2. Open het tabblad **Access Tokens** (naast *Peers*).
3. Klik op **Create Access Token** en geef een duidelijke naam op (bijv. `portal-automation`).
4. Bewaar de gegenereerde sleutel veilig voor gebruik in de automatiseringsworkflow.

---

## 3. VLAN-groepen aanmaken
Onder **Groups** zijn drie aparte groepen aangemaakt die elk overeenkomen met een specifiek VLAN-subnet:

| Groepsnaam | Gekoppeld Subnet | Doel |
| :--- | :--- | :--- |
| `VLAN_10` | `10.50.10.0/24` | Toegang tot netwerk/servers in VLAN 10 |
| `VLAN_20` | `10.50.20.0/24` | Toegang tot netwerk/servers in VLAN 20 |
| `VLAN_21` | `10.50.21.0/24` | Toegang tot netwerk/servers in VLAN 21 |

---

## 4. Networks, Resources en Access Control Policies
Onder **Network Routing → Networks** zijn drie netwerken geconfigureerd. Voor elk netwerk is een resource (het CIDR-subnet) en een bijbehorende Access Control Policy aangemaakt.

### 4.1 Overzicht van geconfigureerde Networks & Resources
| Netwerknaam | Resource-naam | Adres (CIDR) | Omschrijving |
| :--- | :--- | :--- | :--- |
| `VLAN_10` | `VLAN_10` | `10.50.10.0/24` | `VLAN 10 network Permission` |
| `VLAN_20` | `VLAN_20` | `10.50.20.0/24` | `Permission to 10.50.20.0/24` |
| `VLAN_21` | `VLAN_21` | `10.50.21.0/24` | `Permission to 10.50.21.0/24` |

### 4.2 Access Control Policies per netwerk
Voor elke resource is via het tabblad **Access Control → Add Policy** een regel aangemaakt met de volgende instellingen:
* **Protocol:** `ALL`
* **Ports:** `ALL`
* **Source:** De corresponderende studentengroep (`VLAN_10`, `VLAN_20` of `VLAN_21`)
* **Destination:** De bijbehorende resource (`VLAN_10`, `VLAN_20` of `VLAN_21`)
* **Enable Policy:** Aan (`Enabled`)
* **Posture Checks:** Geen (overgeslagen)

---

## 5. Routing Peer configureren
Om het verkeer vanuit NetBird-clients door te sturen naar de interne `10.50.x.0/24`-subnets, is aan elk van de drie netwerken een **Routing Peer** gekoppeld.

### Instellingen Routing Peer
* **Selectiemethode:** `Routing Peers` *(niet Peer Group)*
* **Geselecteerde Peer:** `vm1` (`100.100.51.23`)
* **Advanced Settings:**
  * **Enable Routing Peer:** Aan (`Enabled`)
  * **Masquerade:** Aan (`Enabled`) – *zorgt voor NAT zodat er geen extra statische routes op de lokale routers of doel-VM's nodig zijn.*
  * **Metric:** `9999`

> **Aandachtspunt voor testen en productie:**
> 1. **Statuscontrole:** In het overzicht onder **Network Routing → Networks** moet het bolletje naast `1 Peer(s)` **groen** zijn. Een oranje status geeft aan dat de Routing Peer (`vm1`) op dat moment offline is of geen actieve NetBird-verbinding heeft.
> 2. **Productie-advies:** In de definitieve opstelling wordt aangeraden `vm1` te vervangen door een vaste, altijd actieve Linux gateway-VM (of de NetBird-server zelf) die toegang heeft tot alle benodigde VLAN's.