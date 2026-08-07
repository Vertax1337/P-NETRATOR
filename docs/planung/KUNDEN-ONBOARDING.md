# MSP AVD Accelerator – Kunden-Onboarding

> **Rolle dieses Dokuments:** Operativer Standardprozess für das Onboarding eines neuen Kunden und die Ableitung eines reproduzierbaren Azure-Deployments  
> **Projektstatus, Gates und Prioritäten:** ausschließlich in [`../../UMSETZUNGSPLAN.md`](../../UMSETZUNGSPLAN.md)  
> **Architekturdefaults:** [`ARCHITEKTUR-ENTSCHEIDUNGEN.md`](ARCHITEKTUR-ENTSCHEIDUNGEN.md)  
> **Azure-DevOps-Zielprozess:** [`AZURE-DEVOPS.md`](AZURE-DEVOPS.md)  
> **Validierung und Release:** [`VALIDIERUNG-UND-RELEASE.md`](VALIDIERUNG-UND-RELEASE.md)  
> **Stand:** 2026-08-07

---

# 1. Zweck

Dieses Dokument beschreibt, **wie aus einem neuen Kundenprojekt ein standardisiertes Azure-Deployment entsteht**.

Es beantwortet damit nicht primär die Frage, wie einzelne Bicep-Module technisch implementiert werden, sondern:

```text
Kundenanforderungen
        ↓
Architekturprofil
        ↓
Kosten-/Freigabeentscheidung
        ↓
Kundenparameter
        ↓
Azure-DevOps-Pipeline
        ↓
Deployment
        ↓
Validierung
        ↓
Betriebsübergabe
```

Ziel ist, dass ein Standard-KMU nicht individuell „neu gebaut“ wird, sondern möglichst aus freigegebenen MSP-Bausteinen und wenigen kundenbezogenen Parametern entsteht.

---

# 2. Grundprinzip

Ein neues Kunden-Onboarding soll langfristig nach folgendem Ablauf erfolgen:

```text
01 Kundenaufnahme / Assessment
        ↓
02 Anforderungen klassifizieren
        ↓
03 MSP-Zielprofil ableiten
        ↓
04 Architektur- und Kostenfreigabe
        ↓
05 Azure-/Tenant-Onboarding
        ↓
06 Kundenkonfiguration in Azure DevOps
        ↓
07 Preflight
        ↓
08 Bicep Build / Validation / What-If
        ↓
09 Freigabe / Approval
        ↓
10 Foundation Deployment
        ↓
11 Identity Deployment conditional
        ↓
12 AVD Accelerator Deployment
        ↓
13 APP / SQL conditional
        ↓
14 Runtime-Konfiguration / Automation
        ↓
15 Post-Deployment-Validierung
        ↓
16 Kosten-/Security-Abgleich
        ↓
17 Produktivfreigabe
        ↓
18 Dokumentation / Betriebsübergabe
```

---

# 3. Phase 01 – Kundenaufnahme / Assessment

Vor einem technischen Deployment werden die Anforderungen vollständig erfasst.

Mindestens folgende Bereiche gehören zum Assessment.

## 3.1 Benutzer / Nutzung

Erfassen:

- Gesamtzahl Mitarbeiter,
- Anzahl vorgesehener AVD-Benutzer,
- erwartete gleichzeitige Sessions,
- typische Arbeitszeiten,
- Abend-/Wochenendnutzung,
- Homeoffice-/Remote-Anteil,
- besondere Benutzergruppen oder Administratoren.

Ziel ist die spätere Ableitung von:

- Session-Host-Anzahl,
- Größenstrategie,
- Power-Management-Profil,
- Kostenmodell.

---

## 3.2 Anwendungen / Branchensoftware

Für jede produktive Anwendung mindestens prüfen:

- Hersteller / Produkt / Version,
- Installationsmodell,
- Mehrbenutzerfähigkeit,
- Unterstützte Windows-Version,
- Datenbankabhängigkeit,
- APP-/SQL-Server erforderlich?,
- klassischer Domain Join erforderlich?,
- LDAP / Kerberos / NTLM erforderlich?,
- Entra ID / moderne Authentifizierung unterstützt?,
- Lizenzserver / Hardwaredongle / Spezialtreiber?,
- Netzwerkports / Herstelleranforderungen,
- Supportfreigabe für AVD/RDS.

Die Identity-Entscheidung wird **nicht aus Gewohnheit**, sondern aus diesen technischen Anforderungen abgeleitet.

---

## 3.3 Identity

Grundfrage:

```text
Reicht Entra ID vollständig aus?
```

Wenn ja:

```text
Entra ID Only
→ kein Domain Controller
```

Wenn nein:

```text
klassisches AD DS erforderlich
```

Dann zusätzlich erfassen:

- neue Gesamtstruktur oder vorhandenes AD?,
- vorhandener Forest/Domain Name,
- Benutzer-/Gruppenbestand,
- Abhängigkeiten zu On-Premises,
- benötigte GPOs,
- DNS-Anforderungen,
- Migration vs. Existing Azure Forest.

Mögliche Zielvarianten:

```text
EntraID
ADDS / NewForest
ADDS / MigrateExistingForest
ADDS / ExistingAzureForest
EntraDS / Sonderfall
```

---

## 3.4 Daten / Files / Profile

Erfassen:

- bestehende Fileserver-Daten,
- SharePoint-/OneDrive-Nutzung,
- Azure-Files-Bedarf,
- Benutzerprofilgröße,
- Outlook-/OST-Nutzung,
- OneDrive Known Folder Move,
- Profilpersistenz erforderlich?,
- mehrere Session Hosts vorgesehen?,
- Datenwachstum,
- Backup-/Retention-Anforderungen.

Daraus werden später insbesondere O002–O004 angewendet:

- FSLogix Default,
- Azure Files Tier,
- Service Endpoint vs. Private Endpoint.

---

## 3.5 Netzwerk / Migration

Erfassen:

- bestehende Standorte,
- Internetanbindung,
- bestehende VPNs,
- vorübergehende S2S-Verbindung für Migration nötig?,
- IP-Adressbereiche / Überschneidungen,
- lokale Systeme, die während Migration weiter erreichbar sein müssen,
- DNS-/Namensabhängigkeiten,
- ggf. verbleibende NAS für Backup/Archiv.

Zielzustand bleibt grundsätzlich:

> Azure ist nach abgeschlossener Migration ohne den bisherigen physischen Server betriebsfähig.

---

## 3.6 Verfügbarkeit / Betriebsanforderungen

Kundenanforderung klassifizieren:

```text
CostOptimized
oder
Resilient / erhöhte Verfügbarkeit
```

CostOptimized bedeutet typischerweise:

- 1 Session Host,
- kein DC, wenn Entra-only möglich,
- bei notwendigem ADDS: 1 DC zulässig,
- keine Azure Firewall als Default,
- keine erzwungenen Availability Zones,
- keine unnötigen Premium-Komponenten.

Resilient wird nur bei tatsächlicher fachlicher Anforderung gewählt.

---

## 3.7 Security / Compliance / Monitoring

Erfassen:

- besondere Compliance-Vorgaben,
- Public Endpoint mit VNet Restriction zulässig?,
- Private Endpoint zwingend?,
- Logging-/Retention-Vorgaben,
- Defender-Anforderungen,
- Backup-/Restore-Ziele,
- Administrations-/Notfallzugriff,
- dokumentationspflichtige Betriebsprozesse.

---

# 4. Phase 02 – Kundenanforderungen klassifizieren

Nach dem Assessment wird das Kundenprojekt auf standardisierte technische Entscheidungen reduziert.

Beispiel:

```text
CustomerCode:          must
Region:                germanywestcentral
Environment:           prod
AvailabilityProfile:   CostOptimized
IdentityMode:          EntraID
SessionHostCount:      1
DeployAppServer:       false
DeploySqlServer:       false
FSLogix:               <gemäß O002>
PowerManagement:       <gemäß O008>
Monitoring:            <gemäß O005>
```

oder:

```text
CustomerCode:          kunde-b
AvailabilityProfile:   CostOptimized
IdentityMode:          ADDS
ADDSMode:               NewForest
DomainControllers:      1
SessionHostCount:       1
DeployAppServer:        true
DeploySqlServer:        true
DependencyAutomation:   true
```

---

# 5. Phase 03 – MSP-Zielprofil ableiten

Geplante Standardprofile mindestens:

```text
entra-small
adds-small
```

Optionale Erweiterungen später:

```text
app-sql
resilient
existing-azure-ad
custom
```

## 5.1 Beispiel Entra-only

```text
entra-small
├── Entra ID
├── kein DC
├── 1 Session Host
├── Single VNet
├── NAT Gateway
├── kein Azure Firewall Default
├── Start VM on Connect
├── Power Management gemäß O008
├── FSLogix gemäß O002
└── Monitoring gemäß O005
```

## 5.2 Beispiel ADDS / LOB

```text
adds-small
├── DC01
├── AVD01
├── Single VNet
├── NAT Gateway
├── Azure Files / FSLogix gemäß O002–O004
├── APP01 optional
├── SQL01 optional
└── Dependency Automation optional
```

---

# 6. Phase 04 – Architektur- und Kostenfreigabe

Bevor Azure-Ressourcen bereitgestellt werden, wird eine Deployment Summary erzeugt.

Beispiel:

```text
Customer:             Mustermann GmbH
Region:               Germany West Central
Profile:              adds-small

Identity:
  AD DS               yes
  DC Count            1

AVD:
  Session Hosts       1
  Start VM on Connect yes
  Power Management    <gewählter Standard>

Workloads:
  APP01               yes
  SQL01               yes

Network:
  Single VNet
  NAT Gateway         yes
  Azure Firewall      no
  Private Endpoint    <gemäß Profil>

Profiles:
  FSLogix             <gemäß Profil>

Estimated Monthly Cost:
  Foundation          ...
  AVD                 ...
  Identity            ...
  APP/SQL             ...
  Storage             ...
  Monitoring          ...
```

Vor Deployment müssen Abweichungen vom Standardprofil nachvollziehbar dokumentiert sein.

---

# 7. Phase 05 – Azure-/Tenant-Onboarding

Für das Deployment werden die erforderlichen Kundenressourcen und Berechtigungen vorbereitet.

Zielmodell:

```text
Customer Azure Subscription
        ↑
Azure DevOps Service Connection
        ↑
Workload Identity Federation
```

Keine langlebigen Client Secrets als Standard.

Zu klären bzw. bereitzustellen:

- Tenant,
- Subscription,
- Zielregion,
- MSP-Berechtigungen,
- Service Connection,
- Test-/Prod-Trennung,
- ggf. bestehende Ressourcen/RBAC-Grenzen.

Beispielnamen:

```text
sc-must-prod
sc-must-test
```

Die endgültige MSP-Isolationsstrategie wird beim Azure-DevOps-Aufbau festgelegt.

---

# 8. Phase 06 – Kundenkonfiguration in Azure DevOps

Es wird **kein Accelerator-Fork pro Kunde** angelegt.

Stattdessen:

```text
zentrale MSP-Codebasis
+
kundenbezogene Parameter
```

Geplante Struktur beispielsweise:

```text
msp/customers/
└── mustermann/
    ├── customer.bicepparam
    └── workloads.bicepparam
```

Mögliche Parameter:

```bicep
param customerCode = 'must'
param environment = 'prod'
param location = 'germanywestcentral'
param identityMode = 'EntraID'
param sessionHostCount = 1
param deployAppServer = false
param deploySqlServer = false
```

Nicht im Repository:

- Passwörter,
- Domain-Join-Secrets,
- Storage Keys,
- PATs,
- Client Secrets,
- langlebige SAS.

---

# 9. Phase 07 – Preflight

Vor der eigentlichen Bereitstellung wird geprüft, ob das Deployment technisch möglich ist.

Mindestens:

- Subscription erreichbar,
- notwendige Resource Provider vorhanden/registrierbar,
- Zielregion verfügbar,
- gewünschte VM-SKUs verfügbar,
- Family-/vCPU-Quota ausreichend,
- Marketplace Images vorhanden,
- Management-VM Small-Disk-Image vorhanden,
- RBAC ausreichend,
- Netzwerkparameter valide,
- IP-Adressbereiche konfliktfrei,
- Identity-Modell vollständig,
- benötigte Microsoft-/AVD-Endpunkte erreichbar,
- AVD Enterprise Application IDs ermittelbar.

Bei ADDS zusätzlich:

- Domain-/Forest-Parameter,
- DNS-Plan,
- Join-Modell,
- ggf. Migrationsverbindung.

Fehler im Preflight stoppen das Deployment vor kostenpflichtiger Teilbereitstellung soweit möglich.

---

# 10. Phase 08 – Build / Validation / What-If

Ablauf:

```text
Bicep Restore
→ Bicep Lint
→ Bicep Build
→ ARM/Bicep Validation
→ What-If
```

What-If muss vor dem produktiven Deployment sichtbar sein.

Besonders kritisch:

```text
Delete
Replace
unerwartete neue Premium-/Enterprise-Ressourcen
```

Produktive Deployments starten nicht automatisch nach einem Merge.

---

# 11. Phase 09 – Freigabe

Bevor das Deployment gegen die produktive Kundensubscription ausgeführt wird:

```text
What-If
→ technischer Review
→ Azure DevOps Environment Approval
→ Deploy
```

Je nach organisatorischem Modell kann zusätzlich eine dokumentierte Kunden-/Projektfreigabe vorgesehen werden.

---

# 12. Phase 10 – Foundation Deployment

MSP Foundation stellt mindestens bereit:

```text
Resource Groups
VNet
├── snet-avd
├── snet-infra
├── snet-private-endpoints optional
└── GatewaySubnet optional
NSGs
NAT Gateway
Service Endpoints gemäß Profil
DNS Basis
```

Der Accelerator verwendet anschließend das bestehende Netzwerk:

```bicep
createAvdVnet = false
```

---

# 13. Phase 11 – Identity conditional

## 13.1 Entra-only

```text
Foundation
→ keine DC-Stage
→ AVD
```

## 13.2 ADDS NewForest

```text
Foundation
→ DC01
→ AD DS / DNS
→ Health Check
→ AVD
```

## 13.3 MigrateExistingForest

```text
temporäre Standortanbindung
→ Azure DC als zusätzlicher DC
→ AD/DNS Replikation
→ Workload-Migration
→ FSMO/DNS ggf. verlagern
→ alten DC demoten
→ physischen Server decommissionen
```

---

# 14. Phase 12 – AVD Accelerator

Der Microsoft Accelerator stellt den AVD-Core bereit:

```text
Host Pool
Workspace
Application Group
Session Host(s)
AVD-Konfiguration
Scaling/Power-Integration gemäß gewähltem Profil
FSLogix conditional
```

Verwendet wird nicht ungeprüft Microsoft `main`, sondern der freigegebene MSP-Stand mit dokumentiertem Compatibility-/Hardening-Patchset.

---

# 15. Phase 13 – APP / SQL conditional

Falls erforderlich:

```text
APP01
SQL01
```

werden als separate MSP-Workloads bereitgestellt.

Sie gehören nicht in den Microsoft-AVD-Core.

Domain Join nur bei tatsächlicher technischer Notwendigkeit.

---

# 16. Phase 14 – Runtime-Konfiguration / Automation

Versionierte Runtime-Artefakte werden aus dem MSP-Release erzeugt.

Zielprinzip:

```text
Azure Repos Commit
→ Pipeline
→ Runtime Package
→ Release Manifest + Hashes
→ private Runtime-Quelle
→ Ziel-VMs
```

Die konkrete Zugriffsmethode wird durch O007 festgelegt.

Bei APP-/SQL-Abhängigkeit:

```text
Start:
SQL → Health → APP

Stop:
APP vollständig → SQL
```

---

# 17. Phase 15 – Post-Deployment-Validierung

Ein erfolgreiches ARM-/Bicep-Deployment ist **nicht** ausreichend für die Produktivfreigabe.

Mindestens prüfen:

- erwartete Ressourcen vorhanden,
- keine unerwarteten Ressourcen,
- Session Host im Host Pool registriert,
- Session Host `Available`,
- Agent/Bootloader gesund,
- Join-Zustand korrekt,
- Login funktioniert,
- FSLogix funktioniert, falls aktiviert,
- Cross-User-Profilzugriff wird verweigert,
- Start VM on Connect funktioniert,
- gewähltes Power Management funktioniert,
- Management-VM deallokiert, falls vorhanden,
- Management-VM Firewall aktiv,
- keine sensitiven DSC-/MOF-Artefakte vorhanden,
- keine unerwarteten Public IPs,
- Monitoring liefert erwartete Daten, falls aktiviert.

Details und Testmatrix:

[`VALIDIERUNG-UND-RELEASE.md`](VALIDIERUNG-UND-RELEASE.md)

---

# 18. Phase 16 – Kosten-/Security-Abgleich

Nach Deployment wird Soll gegen Ist geprüft.

Beispiel:

```text
Expected:
1 NAT Gateway
1 AVD Session Host
0 Azure Firewall
0 Bastion
0 DDoS Standard

Actual:
1 NAT Gateway   ✓
1 AVD           ✓
0 Firewall      ✓
0 Bastion       ✓
0 DDoS          ✓
```

Abweichungen müssen begründet und dokumentiert werden.

Kostenprüfung ist für das KMU-Framework Bestandteil der technischen Qualitätssicherung.

---

# 19. Phase 17 – Produktivfreigabe

Produktivfreigabe erst wenn:

- technische Validierung erfolgreich,
- Security-Gates erfüllt,
- Kosten-/Ressourcenabgleich plausibel,
- Login-/Applikationstest erfolgreich,
- Backup-/Monitoringanforderungen erfüllt,
- bekannte Abweichungen dokumentiert.

---

# 20. Phase 18 – Dokumentation / Betriebsübergabe

Für jeden Kunden soll ein Deployment-/Betriebsdatensatz entstehen.

Mindestens dokumentieren:

```text
Kunde
├── Tenant / Subscription
├── Region
├── MSP Release Version
├── Microsoft Upstream Commit
├── Kundenprofil
├── Netzwerk
├── Identity-Modell
├── Session Hosts
├── APP / SQL
├── FSLogix
├── Power Management
├── Backup
├── Monitoring
├── Runtime Artifact Version
├── Wartungsprozess
└── Recovery-/Notfallinformationen
```

Betriebsfragen, die beantwortet sein müssen:

- Wie wird ein Session Host gewartet?
- Wie wird Power Management temporär ausgenommen?
- Wie wird die Management-VM für Wartung gestartet und wieder deallokiert?
- Wie wird FSLogix geprüft?
- Wie wird ein zusätzlicher Session Host hinzugefügt?
- Wie wird ein neues MSP Runtime Release eingespielt?
- Wie wird ein What-If gegen die bestehende Umgebung ausgeführt?
- Wie erfolgt Restore/Recovery bei kritischen Komponenten?

---

# 21. Standardisierungsziel

Der gewünschte Reifegrad des Projekts ist erreicht, wenn ein Standard-KMU im Wesentlichen so bereitgestellt werden kann:

```text
Assessment
→ Profil auswählen
→ Abweichungen festlegen
→ Kundenparameter erzeugen
→ Preflight
→ What-If
→ Approval
→ Deploy
→ Validate
→ Handover
```

Der Schwerpunkt des MSP liegt dann nicht mehr auf wiederholtem Infrastrukturprogrammieren, sondern auf:

- sauberer Anforderungsaufnahme,
- Auswahl des richtigen Profils,
- kontrollierten Abweichungen,
- fachlicher Migration,
- Validierung und Betrieb.

---

# 22. Abhängigkeit von offenen Architekturentscheidungen

Dieses Dokument beschreibt den Zielprozess bereits vollständig, legt aber offene technische Defaults **nicht eigenständig fest**.

Bis G0 abgeschlossen ist, gelten insbesondere O001–O010 aus:

[`ARCHITEKTUR-ENTSCHEIDUNGEN.md`](ARCHITEKTUR-ENTSCHEIDUNGEN.md)

als maßgeblich.

Beispiele:

- O002 bestimmt den FSLogix-Default,
- O003 bestimmt den Azure-Files-Tier-Default,
- O004 bestimmt Service Endpoint vs. Private Endpoint,
- O005 bestimmt Monitoring,
- O006 bestimmt die Session-Host-Größenstrategie,
- O007 bestimmt die Runtime-Artefakt-Auslieferung,
- O008 bestimmt das AVD Power Management,
- O009 bestimmt das Patch-/Replacement-Modell,
- O010 bestimmt den Storage-/Domain-Join-Helper.

Der Onboarding-Prozess darf diese Entscheidungen später nur **anwenden**, nicht neu erfinden.

---

# 23. Pflege-Regel

Dieses Dokument wird aktualisiert, wenn sich der **operative Kundenprozess** ändert.

Nicht hier pflegen:

- Projektstatus,
- Prioritäten,
- offene Architekturentscheidungsstatus,
- Accelerator-Finding-Details.

Diese bleiben in den jeweiligen Master-/Fachdokumenten.
