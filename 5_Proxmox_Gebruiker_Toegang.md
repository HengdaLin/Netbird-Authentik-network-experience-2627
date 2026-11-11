# Technische Documentatie: Proxmox VE Toegang voor Collega

**Doel:** Een collega veilig toegang geven tot de Proxmox-server (via Tailscale), met een eigen account en beperkte rechten, zonder het `root`-wachtwoord te delen.

## 1. Overzicht

Toegang bestaat uit twee onderdelen:

| Onderdeel | Doel | Waar |
| --- | --- | --- |
| Netwerktoegang | Collega kan de server bereiken | Tailscale (document 4) |
| Proxmox-account | Collega kan inloggen en werken met eigen rechten | Proxmox (dit document) |

## 2. Voorwaarden

* Tailscale is geïnstalleerd en actief op de server (zie `4_Install_Tailscale.md`).
* Het Tailscale IP van de server is bekend: `tailscale ip -4`.
* Je collega heeft Tailscale geïnstalleerd en een eigen Tailscale-account.

## 3. Netwerktoegang: machine delen via Tailscale

Delen geeft de collega alleen toegang tot deze ene machine, niet tot de rest van de tailnet.

1. Open `https://login.tailscale.com/admin/machines`.
2. Klik bij `proxmox` op de drie puntjes en kies **Share**.
3. Genereer een uitnodigingslink en stuur die naar de collega.
4. De collega accepteert de uitnodiging met zijn eigen Tailscale-account.
5. Controleer dat de collega kan pingen: `ping <tailscale-ip>`.

## 4. Proxmox-gebruiker aanmaken

Het `root`-wachtwoord wordt **nooit** gedeeld. Er wordt een eigen gebruiker aangemaakt in de realm **Proxmox VE authentication server** (`pve`).

### 4.1 Via de webinterface

1. **Datacenter, Permissions, Users, Add**
2. Vul in:
   * User name: `collega`
   * Realm: **Proxmox VE authentication server**
   * Password: tijdelijk wachtwoord
3. Klik op **Add**.

### 4.2 Via de shell (als `root`)

```bash
pveum user add collega@pve
pveum passwd collega@pve
```

Vervang `collega` door de echte gebruikersnaam. Het commando `pveum passwd` vraagt interactief om het wachtwoord.

## 5. Rechten toekennen

Rechten bestaan uit een **gebruiker**, een **pad** en een **rol**.

### 5.1 Beschikbare rollen

| Rol | Wat de gebruiker mag |
| --- | --- |
| `PVEVMUser` | VM's gebruiken (starten, stoppen, console) |
| `PVEVMAdmin` | VM's en containers beheren |
| `Administrator` | Alles, inclusief de host zelf (alleen indien echt nodig) |

### 5.2 Standaardrechten (VM's beheren en aanmaken)

```bash
pveum acl modify /vms --users collega@pve --roles PVEVMAdmin
pveum acl modify /storage --users collega@pve --roles PVEDatastoreUser
pveum acl modify /sdn/zones/localnetwork --users collega@pve --roles PVESDNUser
```

| Pad | Rol | Waarvoor |
| --- | --- | --- |
| `/vms` | `PVEVMAdmin` | VM's en containers beheren |
| `/storage` | `PVEDatastoreUser` | Schijven en ISO's gebruiken bij het aanmaken van VM's |
| `/sdn/zones/localnetwork` | `PVESDNUser` | Netwerkkaarten (`vmbr0`) aan een VM koppelen |

> ⚠️ Zonder de rechten op `/storage` en `/sdn/zones/localnetwork` kan de collega wel bestaande VM's beheren, maar krijgt hij de melding *Permission check failed* bij het aanmaken van een nieuwe VM.

### 5.3 Rechten beperken tot één VM

Voor een collega die maar bij één VM mag (bijvoorbeeld VM 100):

```bash
pveum acl modify /vms/100 --users collega@pve --roles PVEVMAdmin
```

### 5.4 Via de webinterface

**Datacenter, Permissions, Add, User Permission**: kies het pad, de gebruiker en de rol.

## 6. Inloggen door de collega

1. Open `https://<tailscale-ip>:8006` in de browser (met Tailscale actief).
2. Accepteer de waarschuwing over het zelfondertekende certificaat (normaal bij Proxmox).
3. Gebruikersnaam: `collega`
4. Realm: **Proxmox VE authentication server**
5. Wijzig direct het wachtwoord: rechtsboven op de gebruikersnaam klikken, dan **Password**.

## 7. Twee-factor-authenticatie (aanbevolen)

De collega kan 2FA zelf instellen via **Datacenter, Permissions, Two Factor** (of via zijn eigen gebruikersinstellingen). Ondersteund zijn onder meer TOTP-apps en WebAuthn.

## 8. SSH naar de host (optioneel)

SSH naar de Proxmox-host is **alleen** nodig als de collega de shell van de host zelf moet gebruiken. Voor VM-beheer volstaat de webinterface. Een `pve`-gebruiker kan niet via SSH inloggen; daarvoor is een apart Linux-account op de host nodig:

```bash
adduser collega
```

Geef alleen een Linux-account als dat echt nodig is, en nooit `sudo`-rechten zonder reden.

## 9. Controleren

| Doel | Commando |
| --- | --- |
| Alle gebruikers tonen | `pveum user list` |
| Alle rechten tonen | `pveum acl list` |
| Effectieve rechten van één gebruiker | `pveum user permissions collega@pve` |
| Tailscale-verbinding | `tailscale status` |

## 10. Toegang intrekken

```bash
pveum acl delete /vms --users collega@pve --roles PVEVMAdmin
pveum acl delete /storage --users collega@pve --roles PVEDatastoreUser
pveum acl delete /sdn/zones/localnetwork --users collega@pve --roles PVESDNUser
pveum user delete collega@pve
```

Trek daarna ook de Tailscale-share in: admin console, **Machines**, bij `proxmox` de share van de collega verwijderen.

## 11. Troubleshooting

| Symptoom | Oorzaak | Oplossing |
| --- | --- | --- |
| Pagina laadt niet | Tailscale niet actief bij de collega | Tailscale starten en inloggen |
| `Login failed` | Verkeerde realm | Realm op **Proxmox VE authentication server** zetten |
| `Permission check failed` bij VM aanmaken | Ontbrekende rechten op storage of netwerk | Sectie 5.2 uitvoeren |
| Collega ziet geen VM's | Geen rechten op `/vms` | `pveum acl modify /vms ...` uitvoeren |
| Server niet bereikbaar | Server offline of WiFi weg | Controleer `tailscale status` en WiFi (document 2) |
| Collega kan geen internet in een VM | `vmbr0` heeft geen poorten | Beperking van de WiFi-opzet (zie document 2) |

## 12. Beveiligingsaanbevelingen

* Deel het `root`-wachtwoord nooit.
* Geef de minimaal benodigde rol.
* Laat de collega het tijdelijke wachtwoord direct wijzigen.
* Schakel 2FA in.
* Deel de Tailscale-uitnodigingslink alleen met de beoogde persoon.
* Trek de toegang in zodra die niet meer nodig is.

## 13. Samenvatting van de wijzigingen

| Onderdeel | Wijziging |
| --- | --- |
| Tailscale admin console | Machine `proxmox` gedeeld met de collega |
| Proxmox-gebruiker | `collega@pve` aangemaakt |
| ACL `/vms` | Rol `PVEVMAdmin` |
| ACL `/storage` | Rol `PVEDatastoreUser` |
| ACL `/sdn/zones/localnetwork` | Rol `PVESDNUser` |