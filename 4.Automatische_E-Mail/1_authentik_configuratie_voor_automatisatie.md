# Technische Documentatie: Authentik Configuratie voor Automatisatie

## 1. Doel en Overzicht
Dit document beschrijft de configuratie van **Authentik** (`https://authentik.test.lab`) ter voorbereiding op het geautomatiseerde docentenportaal voor VM-provisioning (via Terraform) en NetBird-toegang. 

Authentik vervult in deze architectuur twee rollen:
1. **Identity Provider (IdP) & Toegangsbeheer:** Docenten loggen via Single Sign-On (OIDC) in op het aanvraagportaal. Studenten hebben geen toegang tot dit portaal.
2. **Centrale Gebruikers- en Groependatabase:** Het automatiseringsportaal leest via de Authentik REST API alle beschikbare studentengroepen, individuele studenten en hun e-mailadressen uit.

---

## 2. Groepenstructuur configureren
Om studenten zowel per vak/klas als individueel te kunnen selecteren, is een hiërarchische groepenstructuur opgezet onder **Directory → Groups**.

### Bestaande basisgroepen
* `authentik Admins` (Superuser privileges: **Yes**)
* `admins` (Superuser privileges: **No**)
* `teachers` (Superuser privileges: **No**) – Docenten met toegang tot het portaal.
* `students` (Superuser privileges: **No**) – Bovenliggende groep voor alle studenten.

### Aangemaakte subgroepen per vak
Voor de verschillende opleidingsonderdelen zijn subgroepen aangemaakt die overerven van `students`:

| Groepsnaam | Parent Group | Superuser Privileges | Doel |
| :--- | :--- | :--- | :--- |
| `Students_NetwExp` | `students` | Uit (`No`) | Studenten voor het vak Network Experience |
| `Students_EntpNetw` | `students` | Uit (`No`) | Studenten voor het vak Enterprise Networking |

> **Opmerking:** Door `students` als *Parent* in te stellen, erven leden van de vakgroepen automatisch alle rechten en eigenschappen van de algemene studentengroep.

---

## 3. Testgebruikers aanmaken
Onder **Directory → Users** zijn 4 testaccounts aangemaakt (één per relevante groep) om de volledige automatiseringsflow te kunnen testen:
* 1 gebruiker in `teachers`
* 1 gebruiker in `students`
* 1 gebruiker in `Students_NetwExp`
* 1 gebruiker in `Students_EntpNetw`

> **Aandachtspunt e-mailadressen:** Elke gebruiker moet voorzien zijn van een geldig e-mailadres zodat het automatiseringssysteem de NetBird- en VM-gegevens apart kan versturen. In een testomgeving met één gedeelde inbox wordt *plus-adressering* aangeraden (bijv. `gebruiker+netwexp@domein.be`) zodat elke student door API's als uniek wordt herkend.

---

## 4. API Token aanmaken voor het Automatisatieportaal
Om het portaal toe te staan gebruikers en groepen op te halen uit Authentik, is een API Token vereist.

1. Navigeer naar **Directory → Tokens and App passwords** en klik op **Create**.
2. Vul de volgende gegevens in:
   * **Identifier:** `portal-api-token`
   * **Intent:** `API Token`
   * **User:** Een beheeraccount uit de groep `authentik Admins`
3. Sla het token op en bewaar de gegenereerde sleutel veilig voor gebruik in het automatiseringsscript/portaal.

### Verificatie
Het token kan worden getest door een `GET`-verzoek te sturen naar het Authentik API-endpoint:
```bash
curl -H "Authorization: Bearer <JOUW_API_TOKEN>" https://authentik.test.lab/api/v3/core/users/
```

---

## 5. OIDC Applicatie en Provider configureren (Docentenportaal)
Om te garanderen dat alleen docenten op het portaal kunnen inloggen, is een aparte OAuth2/OpenID-applicatie aangemaakt onder **Applications → Applications → Create with Provider**.

### 5.1 Applicatie-instellingen
* **Application Name:** `VM & Netbird Portal`
* **Slug:** `vm-netbird-portal`
* **Policy engine mode:** `ANY`

### 5.2 OAuth2/OpenID Provider-instellingen
* **Provider Name:** `Provider for VM & Netbird Portal`
* **Authorization Flow:** `default-provider-authorization-implicit-consent` *(voorkomt dat docenten bij elke login opnieuw een toestemmingsscherm krijgen)*
* **Client Type:** `Confidential`
* **Client ID & Client Secret:** Automatisch gegenereerd en opgeslagen voor de portaalconfiguratie.
* **Signing Key:** `authentik Self-signed Certificate`
* **Redirect URIs:** Wordt aangevuld zodra de definitieve URL van het docentenportaal vaststaat.

### 5.3 Toegangsbeperking (Group Binding)
Tijdens stap 4 (*Configure Bindings*) is een **Group Binding** toegevoegd om studenten de toegang te ontzeggen:
* **Type:** `Group`
* **Group:** `teachers`
* **Enabled:** Aan (`True`)
* **Negate Result:** Uit (`False`)
* **Order:** `0`
* **Timeout:** `30`
* **Failure Result:** `Don't Pass`

### Verificatie
* De applicatie `VM & Netbird Portal` staat zichtbaar in **Applications → Applications** naast `NetBird`.
* Leden van de groep `teachers` zien de applicatie op hun Authentik-dashboard; leden van `students` niet.