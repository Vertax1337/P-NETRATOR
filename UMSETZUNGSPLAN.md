# MSP AVD Accelerator – Umsetzungsplan

> **Status:** Arbeitsdokument / Single Source of Truth  
> **Stand:** 2026-08-07  
> **Repository:** `Vertax1337/P-NETRATOR`  
> **Microsoft-Upstream:** `Azure/avdaccelerator`  
> **Importierter Upstream-Stand:** `63c55c857f77783c01812c935957cd8c9c67a5a7`  
> **Import-Commit in P-NETRATOR:** `aff8e2823011970d1f96fac078dfd5e0e28de2da`  
> **Ursprünglicher P-NETRATOR-Stand gesichert unter:** `archive/p-netrator-original`

---

## 1. Zweck dieses Dokuments

Dieses Dokument ist die verbindliche Aufgaben- und Entscheidungsübersicht für die Anpassung des Microsoft **Azure Virtual Desktop Accelerator** an eine kostenoptimierte, wiederverwendbare MSP-Lösung für kleine und mittlere Unternehmen.

Der Hintergrund für dieses Dokument ist ausdrücklich, dass Entscheidungen aus längeren Chat-/Planungssitzungen nicht verloren gehen oder später unbemerkt neu interpretiert werden sollen.

Daher gelten folgende Regeln:

- Dieses Dokument ist die **Single Source of Truth** für die Umsetzung.
- Bereits getroffene Architekturentscheidungen werden unter **„Verbindliche Leitplanken“** dokumentiert.
- Noch nicht entschiedene Punkte werden ausdrücklich mit **OFFEN** gekennzeichnet.
- Neue Aufgaben werden in die Aufgabenplanung aufgenommen, bevor sie umgesetzt werden.
- Erledigte Aufgaben werden erst nach technischer Validierung auf `[x]` gesetzt.
- Änderungen an der Microsoft-Basis werden ausschließlich in der **Minimalen Patch-Liste** dokumentiert.
- Jede Erweiterung der Patch-Liste muss begründet werden, damit der Abstand zum Microsoft-Upstream möglichst klein bleibt.
- Bei relevanten Architekturänderungen wird zusätzlich der Abschnitt **Entscheidungsprotokoll** aktualisiert.

---

# 2. Zielbild

Das Ziel ist **keine vollständige Eigenimplementierung von Azure Virtual Desktop**.

Der Microsoft AVD Accelerator wird als getestete und gepflegte **AVD Workload Deployment Engine** verwendet. Darum herum entsteht eine dünne MSP-Schicht für:

- kostenoptimierte KMU-Defaults,
- Netzwerk/Foundation,
- optionale AD-DS-Infrastruktur,
- Migration bestehender AD-Gesamtstrukturen,
- optionale APP-/SQL-Server,
- Runtime-Abhängigkeiten zwischen AVD, APP und SQL,
- Azure DevOps Orchestrierung,
- Kundenparameter,
- Preflight-Prüfungen,
- Post-Deployment-Validierung,
- Kostenkontrolle.

Grundprinzip:

```text
MSP Foundation
      ↓
Identity-Auswahl
      ├── Entra ID Only → kein DC erforderlich
      └── AD DS benötigt → Identity-/DC-Dependency bereitstellen
      ↓
Microsoft AVD Accelerator
      ↓
optionale Workloads
      ↓
Konfiguration / Automation
      ↓
Validierung
```

Der Microsoft-Code soll dabei **so wenig wie möglich verändert** werden.

---

# 3. Zielgruppe

Primäre Zielgruppe:

- KMU mit typischerweise **unter 30 Mitarbeitern**,
- meistens **1 AVD Session Host**,
- maximal zunächst 2–3 Session Hosts,
- teilweise reine Entra-ID-/Cloud-Szenarien ohne klassischen Domain Controller,
- häufig vorhandene klassische Branchensoftware,
- teilweise SQL-/APP-Abhängigkeiten,
- Kosten haben hohe Priorität,
- keine unnötige Enterprise-Komplexität.

Die Lösung muss skalierbar bleiben, darf aber nicht standardmäßig Enterprise-Komponenten ausrollen, die für diese Kundengröße wirtschaftlich nicht sinnvoll sind.

---

# 4. Verbindliche Leitplanken

## 4.1 Microsoft Accelerator als Basis

**BESCHLOSSEN**

- Der Microsoft AVD Accelerator wird als Basis verwendet.
- Keine vollständige Neuimplementierung der AVD-Kernlogik.
- Microsoft-Module unter `workload/` und `avm/` bleiben soweit möglich unverändert.
- Eigene MSP-Erweiterungen werden möglichst außerhalb der Microsoft-Struktur abgelegt.
- Upstream-Updates werden kontrolliert übernommen und niemals ungeprüft automatisch in Produktion ausgerollt.

Ziel:

```text
Microsoft Accelerator = AVD Core
MSP Layer             = Foundation + Identity + Workloads + Orchestrierung
```

---

## 4.2 Kostenorientierung

**BESCHLOSSEN**

Die Standardkonfiguration muss für kleine Kunden wirtschaftlich bleiben.

Nicht standardmäßig bereitstellen:

- Azure Firewall,
- DDoS Network Protection Standard,
- AVD Private Link,
- erzwungene Availability Zones,
- erzwungene zwei Session Hosts,
- Domain Controller ohne technische/fachliche Notwendigkeit,
- erzwungene zwei Domain Controller,
- unnötige dauerhaft laufende Management-VMs,
- unnötige Hub-Spoke-Topologien.

Enterprise-/HA-Komponenten dürfen später optional aktivierbar sein.

---

## 4.3 Session Hosts

**BESCHLOSSEN**

Standard:

```text
Session Host Count = 1
```

Die Implementierung soll zunächst 1–3 Session Hosts unterstützen.

VM-Größe bleibt parametrierbar und wird nicht für alle Kunden hart festgelegt.

---

## 4.4 Identity und Domain Controller

**BESCHLOSSEN**

Ein klassischer Domain Controller ist **kein genereller Bestandteil des KMU-Standards**.

Die Identity-Architektur richtet sich nach den technischen Anforderungen des Kunden und insbesondere nach der eingesetzten Branchensoftware.

### Fall A – Entra ID Only

Wenn alle benötigten Anwendungen und Dienste ohne klassisches AD DS betrieben werden können, ist der bevorzugte kleinste Zielzustand:

```text
Entra ID
AVD01
kein Domain Controller
```

In diesem Fall soll **kein DC ausschließlich aus Gewohnheit oder aufgrund des bisherigen On-Prem-Designs bereitgestellt werden**.

### Fall B – Klassisches AD DS erforderlich

Ein Domain Controller wird als zusätzliche Identity-Dependency bereitgestellt, wenn beispielsweise eine Branchensoftware:

- einen klassischen AD-Domain-Join voraussetzt,
- LDAP/Kerberos/NTLM oder andere AD-DS-Funktionen benötigt,
- oder eine andere technische Anforderung gegen Entra-only spricht.

Wenn für einen solchen Kunden eine neue AD-Gesamtstruktur in Azure notwendig ist, ist für das kostenoptimierte KMU-Profil standardmäßig **ein DC zulässig**:

```text
ADDS / CostOptimized:
DC01
```

Zwei DCs bleiben eine optionale resiliente Variante:

```text
ADDS / Resilient:
DC01
DC02
```

Begründung:

Bei kleinen Kunden bestehen häufig auch bei AVD, APP und SQL Single-Instance-Workloads. Es wäre wirtschaftlich inkonsistent, ausschließlich für AD zwingend Hochverfügbarkeit vorzuschreiben. Gleichzeitig soll AD DS aber überhaupt nur bereitgestellt werden, wenn es technisch benötigt wird.

Wenn ein einzelner DC eingesetzt wird, ist dieser ein bewusst akzeptierter SPOF und benötigt deshalb mindestens:

- statische private IP,
- Azure Backup,
- Monitoring,
- dokumentierten Restore-Prozess,
- keine unnötigen Zusatzrollen,
- keinen öffentlichen IP-Zugriff,
- 24/7-Betrieb.

---

## 4.5 On-Premises nach Migration

**BESCHLOSSEN**

Der heute vorhandene physische Server ist nur **IST-Zustand bzw. Migrationsquelle**.

Zielzustand einer Neuimplementierung:

```text
Azure ist eigenständig betriebsfähig.
```

Der physische Server wird nach erfolgreicher Migration abgeschaltet.

Eine eventuell vorhandene NAS darf weiterhin beispielsweise für Backup-Zwecke verwendet werden, ist aber **keine Betriebsabhängigkeit der Azure-Umgebung**.

Ein dauerhaftes Modell „AVD in Azure + einziger produktiver DC On-Premises“ ist daher **kein Standard-Zielbild**.

Bei einem Entra-only-Zielbild existiert nach der Migration entsprechend gar kein klassischer Domain Controller mehr.

---

## 4.6 Netzwerk

**BESCHLOSSEN**

Standard für kleine Kunden ist eine einfache Single-VNet-Architektur.

Beispiel:

```text
VNet Kunde
├── snet-avd
│   └── AVD01
│
├── snet-infra
│   ├── DC01 optional
│   ├── APP01 optional
│   └── SQL01 optional
│
├── snet-private-endpoints optional
│
└── GatewaySubnet optional
```

Grundsätze:

- kein Azure Firewall Default,
- kein Hub-Spoke Default,
- Hub-Spoke/Existing Hub nur bei tatsächlichem Kundenbedarf,
- keine Public IP auf Session Hosts,
- kein standardmäßig geöffnetes RDP/3389 für AVD,
- expliziter Outbound-Zugriff,
- NAT Gateway als bevorzugter kosteneffizienter Outbound-Pfad für das Standardprofil,
- ein NAT Gateway kann innerhalb desselben VNets mehrere relevante Subnetze bedienen.

---

## 4.7 Foundation vor dem Accelerator

**BESCHLOSSEN**

Das Netzwerk soll **nicht** standardmäßig durch den AVD Accelerator erzeugt werden.

Die MSP Foundation stellt zuerst das VNet und die benötigten Subnetze bereit.

Der Accelerator wird anschließend im Existing-VNet-/Brownfield-Modus verwendet:

```bicep
createAvdVnet = false
```

und erhält mindestens:

```text
existingVnetAvdSubnetResourceId
existingVnetPrivateEndpointSubnetResourceId (wenn benötigt)
DNS-Konfiguration
```

Die Deployment-Reihenfolge ist abhängig vom gewählten Identity-Modell:

```text
Entra-only:
Foundation
→ AVD Accelerator

AD DS:
Foundation
→ Identity / DC
→ Identity Health Check
→ AVD Accelerator
```

---

## 4.8 Identity-Modell

**BESCHLOSSEN als Zielmodell, Detailparameter noch umzusetzen**

Die Identity-Auswahl ist ein eigener Architekturparameter. Unterstützte Zielvarianten sollen mindestens umfassen:

```text
EntraID
ADDS
EntraDS
EntraIDKerberos
```

**EntraID ist ein vollständiges Zielmodell und benötigt keinen DC.** AD DS ist eine optionale Dependency und wird nur verwendet, wenn die Kundenanforderungen dies erforderlich machen.

Für AD DS wird zusätzlich unterschieden zwischen:

```text
NewForest
MigrateExistingForest
ExistingAzureForest
```

Bedeutung:

- `NewForest`: neue AD-Gesamtstruktur direkt in Azure,
- `MigrateExistingForest`: bestehende On-Prem-Gesamtstruktur wird temporär erweitert/migriert; On-Prem wird danach abgeschaltet,
- `ExistingAzureForest`: bereits vorhandenes AD DS in Azure wird verwendet.

**OFFEN:** Standardverfahren zur Synchronisierung einer neuen/migrierten AD-Gesamtstruktur nach Entra ID (z. B. Entra Cloud Sync vs. Entra Connect Sync) festlegen.

---

## 4.9 APP- und SQL-Server

**BESCHLOSSEN**

APP und SQL werden **nicht in den AVD Accelerator eingebaut**.

Sie werden als separate optionale Bicep-Workloads implementiert.

Sie sind keine harte Deployment-Abhängigkeit des AVD-Control-Planes, können aber eine fachliche Abhängigkeit der Kundenanwendung darstellen.

Auch APP-/SQL-Workloads dürfen nicht automatisch implizieren, dass AD DS benötigt wird. Die jeweilige Branchensoftware entscheidet über die Identity-Anforderung.

Runtime-Abhängigkeit bei entsprechenden Kunden:

```text
Start:
SQL → APP

Stop:
APP vollständig stoppen → SQL stoppen
```

Bereits korrekte Power States dürfen nicht unnötig erneut angesteuert werden.

---

## 4.10 Power Management

**BESCHLOSSEN**

Für Session Hosts möglichst native AVD-Funktionen verwenden:

- AVD Scaling Plan,
- Start VM on Connect.

Eigene Azure-Automation darf bei solchen Host Pools **nicht parallel denselben Session Host skalieren/deallokieren**, wenn AVD Autoscale dafür verantwortlich ist.

Eigene Automation ist vorgesehen für optionale Dependency-VMs wie:

```text
APP01
SQL01
```

---

## 4.11 Azure DevOps

**BESCHLOSSEN**

Langfristige Zielplattform:

```text
Azure DevOps Repos
Azure Pipelines
Azure DevOps Environments / Approvals
Workload Identity Federation
```

GitHub ist aktuell die Arbeitsbasis für den Accelerator-Import, soll aber nicht die dauerhafte CI/CD-Zielplattform sein.

Keine dauerhaften Client Secrets für Azure-Deployment verwenden.

Ziel-Pipeline:

```text
PRECHECK
   ↓
FOUNDATION
   ↓
IDENTITY (conditional)
   ↓
AVD ACCELERATOR
   ↓
WORKLOAD
   ↓
CONFIGURATION
   ↓
VALIDATION
```

Bei Entra-only wird eine AD-/DC-Identity-Stage übersprungen.

Bei ADDS muss die Identity-Stage erfolgreich validiert sein, bevor der Accelerator gestartet wird.

---

# 5. Geplante Repository-Struktur

Die Microsoft-Struktur soll möglichst erhalten bleiben.

Eigene Komponenten sollen klar separiert werden.

Zielbild:

```text
P-NETRATOR/
│
├── avm/                         # Microsoft / möglichst unverändert
├── workload/                    # Microsoft AVD Accelerator / möglichst unverändert
│
├── msp/
│   ├── foundation/
│   │   ├── main.bicep
│   │   └── modules/
│   │
│   ├── identity/
│   │   ├── new-forest/
│   │   ├── migrate-existing-forest/
│   │   └── existing-azure-forest/
│   │
│   ├── workloads/
│   │   ├── app-server/
│   │   └── sql-server/
│   │
│   ├── profiles/
│   │   ├── entra-small.bicepparam
│   │   ├── adds-small.bicepparam
│   │   └── ...
│   │
│   └── customers/
│       └── example/
│
├── pipelines/
│   ├── azure-pipelines.yml
│   └── templates/
│       ├── preflight.yml
│       ├── foundation.yml
│       ├── identity.yml
│       ├── avd.yml
│       ├── workload.yml
│       └── validate.yml
│
└── UMSETZUNGSPLAN.md
```

**WICHTIG:** Kundendateien dürfen keine Kennwörter oder andere Secrets enthalten.

---

# 6. Minimale Patch-Liste des Microsoft Accelerators

Ziel dieser Liste ist, den Microsoft-Code nur dort zu ändern, wo dies für Kosten, Wartbarkeit, Reproduzierbarkeit oder Azure-DevOps-Betrieb notwendig ist.

Neue Accelerator-Patches müssen vor Umsetzung hier ergänzt werden.

## P0 – Management-VM von Session Host entkoppeln

**Status:** `[ ]` OFFEN / umzusetzen  
**Priorität:** P0

Aktueller Microsoft-Stand:

```bicep
mgmtVmSize: avdSessionHostsSize
```

Dadurch erhält die temporäre Storage-/Domain-Join-Management-VM dieselbe Größe wie der AVD Session Host.

Ziel:

```bicep
param managementVmSize string
```

Eigener Default für das KMU-Profil.

Aktueller Kandidat:

```text
Standard_B2als_v2
2 vCPU
4 GB RAM
```

Vor Festlegung:

- regionale Verfügbarkeit prüfen,
- Accelerator-Skripte auf der VM testen,
- Domain Join testen,
- PowerShell/NuGet/DSC-Lauf testen,
- Deploymentdauer beobachten.

**Akzeptanzkriterium:** Management-VM-Größe kann unabhängig vom Session Host gesetzt werden und der vollständige Storage-/FSLogix-Workflow läuft erfolgreich.

---

## P0 – Management-VM OS-Disk minimieren

**Status:** `[ ]` OFFEN / umzusetzen  
**Priorität:** P0

Ziel:

- Windows Server Small-Disk-Image verwenden,
- `Standard_LRS`,
- keine Data Disk,
- OS-Disk nicht größer als technisch notwendig,
- wenn das Marketplace-Image bereits die Mindestgröße vorgibt, diese unverändert übernehmen statt künstlich eine größere Disk festzulegen.

Zu prüfen:

- tatsächliche Mindestgröße des verwendeten Marketplace-Images,
- Verhalten des verwendeten AVM-Moduls bei `diskSizeGB`,
- freier Platz nach Installation/DSC-Konfiguration.

**Akzeptanzkriterium:** Nach Deallocation entstehen für die Management-VM nur minimale persistente Storage-Kosten.

---

## P0 – Management-VM nach erfolgreichem Deployment automatisch deallokieren

**Status:** `[ ]` OFFEN / umzusetzen  
**Priorität:** P0

Die VM soll **nicht gelöscht** werden.

Gewünschter Normalzustand:

```text
Management VM = Deallocated
```

Vorteile:

- Domain Membership bleibt bestehen, sofern AD DS verwendet wird,
- Wiederverwendung für spätere FSLogix-/Storage-Wartung möglich,
- NIC und VM-Konfiguration bleiben bestehen,
- keine Compute-Kosten im Normalbetrieb,
- lediglich minimale OS-Disk-Kosten.

Bevorzugte Umsetzung:

```text
Azure DevOps Post-Deployment Stage
→ Storage/FSLogix-Konfiguration erfolgreich?
→ Management VM deallocate
```

Bei zukünftigen Wartungsaufgaben:

```text
Pipeline
→ VM starten
→ Aufgabe ausführen
→ validieren
→ VM deallokieren
```

**Akzeptanzkriterium:** Nach jedem erfolgreichen Deployment/Wartungslauf endet die Management-VM zuverlässig im Zustand `Deallocated`.

---

## P0 – Management-VM Image-Referenz validieren

**Status:** `[ ]` Prüfen  
**Priorität:** P0

Aktueller Accelerator verwendet für die Management-VM ein Windows Server 2022 Small-Disk-Image.

Aufgabe:

- aktuelle Marketplace-Referenz in der Zielregion prüfen,
- sicherstellen, dass Publisher/Offer/SKU weiterhin gültig und deploybar sind,
- nur bei Bedarf auf eine aktuelle Referenz anpassen.

**Akzeptanzkriterium:** Image kann in der Zielregion automatisiert ohne veraltete Marketplace-Abhängigkeit bereitgestellt werden.

---

## P0 – Externe Accelerator-Skripte versionsfest machen

**Status:** `[ ]` OFFEN / umzusetzen  
**Priorität:** P0

Problem:

Teile des Accelerators laden Skripte bzw. DSC-Artefakte direkt von Microsoft-GitHub-URLs bzw. `main`.

Ein von uns getesteter Commit darf später nicht unbemerkt anderen Script-Code ausführen, nur weil sich Microsoft `main` verändert hat.

Ziel:

- Script-/Artifact-Basis-URI parametrierbar machen oder
- auf einen konkreten getesteten Upstream-Commit / Release pinnen oder
- Artefakte im Azure-DevOps-Deployment kontrolliert bereitstellen.

**Akzeptanzkriterium:** Ein Release unseres MSP-Deployments führt reproduzierbar exakt die getesteten Script-Versionen aus.

---

## P1 – Deterministisches Naming / `utcNow()` auf Idempotenz prüfen

**Status:** `[ ]` Prüfen  
**Priorität:** P1

Der Accelerator verwendet `utcNow()`/`time` in mehreren Deploymentnamen und teilweise bei der Ermittlung eindeutiger Namen.

Aufgabe:

- ermitteln, welche Ressourcen bei einem Re-Deployment tatsächlich neue Namen erhalten würden,
- What-If mit identischen Parametern mehrfach ausführen,
- nur dort patchen, wo Idempotenz oder Wiederholbarkeit betroffen ist,
- stabile Seeds aus Subscription/Customer/Environment bevorzugen.

**Akzeptanzkriterium:** Erneutes Deployment mit identischen Parametern erzeugt keine unbeabsichtigten zusätzlichen persistenten Ressourcen.

---

## Patch-Regel

Folgende Bereiche sollen **nicht** ohne konkreten Fehler oder zwingenden technischen Grund verändert werden:

- Host Pool Kernlogik,
- Application Group Kernlogik,
- Workspace Kernlogik,
- AVD Agent Registration,
- Scaling Plan Kernlogik,
- Entra-/AD Join Logik,
- Microsoft AVM-Module.

Ziel ist ein **kleiner, nachvollziehbarer Patch-Delta zum Upstream**.

---

# 7. Umsetzungsphasen

## Phase 0 – Repository und Governance

### Aufgaben

- [x] Microsoft AVD Accelerator in `P-NETRATOR` importieren.
- [x] ursprünglichen P-NETRATOR-Stand auf `archive/p-netrator-original` sichern.
- [x] `UMSETZUNGSPLAN.md` als Single Source of Truth anlegen.
- [ ] Repository-Sichtbarkeit prüfen; vor Speicherung realer Kundenparameter ggf. auf privat umstellen.
- [ ] Microsoft-Upstream-Strategie dokumentieren (`origin`/`upstream`, Merge-/Review-Prozess).
- [ ] Lizenzhinweise des Microsoft MIT-lizenzierten Codes unverändert erhalten.
- [ ] Verzeichnis `msp/` anlegen.
- [ ] klare Trennung zwischen Microsoft-Code und MSP-Code dokumentieren.

### Exit-Kriterium

Repository ist vorbereitet, Upstream-Basis ist reproduzierbar und eigene Erweiterungen können ohne Vermischung mit Microsoft-Code begonnen werden.

---

## Phase 1 – Accelerator-Baseline unverändert validieren

### Ziel

Bevor eigene Änderungen vorgenommen werden, muss bekannt sein, dass die importierte Microsoft-Basis grundsätzlich deploybar ist.

### Aufgaben

- [ ] Bicep Build der Baseline ausführen.
- [ ] Referenzen/Module auf Vollständigkeit prüfen.
- [ ] Marketplace Images für Zielregion Germany West Central prüfen.
- [ ] benötigte Resource Provider dokumentieren.
- [ ] Accelerator-Preconditions in einen technischen Preflight-Katalog übernehmen.
- [ ] Minimalparameter für Testdeployment definieren.
- [ ] What-If gegen Testsubscription durchführen.
- [ ] unverändertes Testdeployment dokumentieren oder begründen, falls bewusst übersprungen.

### Exit-Kriterium

Wir kennen den funktionierenden Ausgangszustand und können spätere Fehler eindeutig eigenen Änderungen zuordnen.

---

## Phase 2 – MSP Foundation

### Ziel

Netzwerk und gemeinsame Azure-Basis werden vor dem Accelerator bereitgestellt. Die Foundation darf keinen Domain Controller voraussetzen.

### Aufgaben

- [ ] `msp/foundation/main.bicep` erstellen.
- [ ] Resource Groups definieren.
- [ ] VNet-Modul erstellen.
- [ ] `snet-avd` erstellen.
- [ ] `snet-infra` erstellen.
- [ ] optionales `snet-private-endpoints` erstellen.
- [ ] optionales `GatewaySubnet` vorsehen.
- [ ] NSGs erstellen.
- [ ] kein standardmäßiges Inbound RDP für Session Hosts.
- [ ] keine Public IP für Session Hosts.
- [ ] NAT Gateway als Standard-Outbound implementieren.
- [ ] NAT Gateway mehreren relevanten Subnetzen desselben VNets zuordnen.
- [ ] optionale Custom-DNS-Server als Output/Parameter vorbereiten.
- [ ] Outputs für Accelerator erzeugen (`avdSubnetId`, `privateEndpointSubnetId`, etc.).
- [ ] What-If und idempotentes Re-Deployment testen.

### Exit-Kriterium

Ein vollständig nutzbares VNet kann unabhängig von Identity-Modell und Accelerator bereitgestellt und mehrfach idempotent deployed werden.

---

## Phase 3 – Identity / optionales AD DS

### Ziel

Entra-only ist ein vollständiger Deploymentpfad. Klassisches AD DS wird nur als optionale Dependency bereitgestellt, wenn der Kunde bzw. seine Branchensoftware es benötigt.

### 3.1 Entra-only

- [ ] Pipeline-/Bicep-Pfad ohne AD-Abhängigkeit definieren.
- [ ] sicherstellen, dass keine DC-Ressourcen erzeugt werden.
- [ ] Entra Join mit Accelerator testen.
- [ ] optionales Intune Enrollment testen.

### 3.2 New Forest – nur bei AD-DS-Anforderung

- [ ] `msp/identity/new-forest/` erstellen.
- [ ] DC01 VM bereitstellen.
- [ ] kleine, geeignete DC-VM-Größe festlegen.
- [ ] statische private IP verwenden.
- [ ] AD DS installieren.
- [ ] neue Forest/Domain erstellen.
- [ ] DNS konfigurieren.
- [ ] VNet-DNS auf DC01 umstellen.
- [ ] Health Check für AD/DNS/Replikationszustand erstellen.
- [ ] Azure Backup für DC01 vorsehen.
- [ ] Monitoring definieren.
- [ ] optional `domainControllerCount = 2` für Resilient-Profil vorbereiten.

### 3.3 Migration Existing Forest – nur bei AD-DS-Anforderung

- [ ] Migrationsablauf dokumentieren.
- [ ] temporäre S2S-Verbindung als Migrationsoption vorsehen.
- [ ] Azure-DC als zusätzlichen DC integrieren.
- [ ] Replikation prüfen.
- [ ] FSMO-Transfer-Prozess dokumentieren.
- [ ] DNS-Umstellung dokumentieren.
- [ ] Demotion des alten On-Prem-DC dokumentieren.
- [ ] Abschaltung des physischen Servers als Abschlusskriterium aufnehmen.

### 3.4 Entra-Synchronisation – nur für AD-DS-Szenarien relevant

- [ ] **OFFEN:** Cloud Sync vs. Connect Sync für unsere Standardszenarien bewerten.
- [ ] Installations-/Betriebsort festlegen.
- [ ] Automatisierungsgrad festlegen.

### Exit-Kriterium

Sowohl ein vollständiges Entra-only-Deployment ohne DC als auch AD-abhängige Accelerator-Deployments ohne dauerhaft benötigten On-Prem-Server sind möglich.

---

## Phase 4 – Minimale Accelerator-Patches

### Aufgaben

- [ ] Management-VM-Größe entkoppeln.
- [ ] `Standard_B2als_v2` als Kandidat technisch validieren.
- [ ] Management-VM OS-Disk minimieren.
- [ ] automatische Deallocation implementieren.
- [ ] Image-Referenz validieren.
- [ ] externe Script-/DSC-Versionen pinnen bzw. parametrieren.
- [ ] `utcNow()`-/Naming-Idempotenz prüfen.
- [ ] Patch-Diff gegen Microsoft-Upstream dokumentieren.
- [ ] Bicep Build durchführen.
- [ ] What-If durchführen.
- [ ] Testdeployment durchführen.

### Exit-Kriterium

Der Accelerator bleibt funktional, die Management-VM verursacht im Normalbetrieb keine Compute-Kosten und unser Patch-Delta bleibt klein.

---

## Phase 5 – AVD KMU-Profile

### Zielprofile

Mindestens:

```text
entra-small
adds-small
```

Dabei ist `entra-small` ein eigenständiges Profil ohne Domain Controller. `adds-small` ist nur für Kunden gedacht, deren technische Anforderungen klassisches AD DS notwendig machen.

Später optional:

```text
resilient
existing-azure-ad
app-sql
```

### Aufgaben

- [ ] Session Host Count Default = 1.
- [ ] Session Host VM-Größe parametrierbar.
- [ ] Pooled Host Pool als KMU-Default prüfen/festlegen.
- [ ] Start VM on Connect aktivieren.
- [ ] Scaling Plan aktivieren.
- [ ] Availability Zones standardmäßig deaktiviert.
- [ ] DDoS Network Protection standardmäßig deaktiviert.
- [ ] AVD Private Link standardmäßig deaktiviert.
- [ ] Defender nicht ungeprüft aktivieren; zusätzliche Kosten transparent behandeln.
- [ ] Monitoring-Baseline festlegen.
- [ ] Kundenparameter von Code trennen.

### Exit-Kriterium

Ein kleiner Kunde kann mit wenigen Parametern reproduzierbar ausgerollt werden, ohne Enterprise-Kostenkomponenten oder einen nicht benötigten DC mitzubezahlen.

---

## Phase 6 – FSLogix / Azure Files

### Aufgaben

- [ ] FSLogix als optionalen Bestandteil der Profile definieren.
- [ ] Azure Files Standard vs. Premium für KMU bewerten.
- [ ] LRS als kostenoptimierten Default prüfen.
- [ ] Service Endpoint vs. Private Endpoint bewerten.
- [ ] bei Service Endpoint sicherstellen, dass `Microsoft.Storage` Service Endpoint gesetzt ist.
- [ ] bei Private Endpoint DNS-Zonen und DNS-Weiterleitung automatisieren.
- [ ] AD DS Authentication testen.
- [ ] Entra Kerberos testen.
- [ ] NTFS-/RBAC-Berechtigungen reproduzierbar setzen.
- [ ] Profile Persistenz nach Session-Host-Rebuild testen.
- [ ] Management-VM-Start/Deallocate-Zyklus für Storage-Wartung testen.

### **OFFEN**

Festlegen, welches FSLogix-/Storage-Modell Default für `entra-small` und `adds-small` wird.

### Exit-Kriterium

Profile funktionieren nach Rebuild/Replace eines Session Hosts weiterhin korrekt und die Storage-Kosten bleiben nachvollziehbar.

---

## Phase 7 – APP-/SQL-Workloads

### Aufgaben

- [ ] `msp/workloads/app-server/` erstellen.
- [ ] `msp/workloads/sql-server/` erstellen.
- [ ] VM-Größen vollständig parametrierbar halten.
- [ ] kein Public IP Default.
- [ ] Domain Join nur optional und abhängig vom Identity-/Applikationsmodell integrieren.
- [ ] Backup-Anforderungen definieren.
- [ ] APP-/SQL-spezifische Disks parametrierbar machen.
- [ ] Branchensoftwareinstallation nicht hart in die Infrastruktur-Baseline einbauen.

### Exit-Kriterium

APP und SQL können als optionale, unabhängig wartbare Module hinzugefügt werden und erzwingen nicht automatisch den Einsatz von AD DS.

---

## Phase 8 – Dependency Automation

### Ziel

Für Kunden mit APP-/SQL-Abhängigkeit:

```text
AVD benötigt APP
APP benötigt SQL
```

### Startlogik

```text
SQL starten
→ Status/Health prüfen
→ optional kurze Wartezeit
→ APP starten
```

### Stopplogik

```text
APP stoppen
→ vollständigen Stop bestätigen
→ SQL stoppen
```

### Aufgaben

- [ ] Runtime-Trigger festlegen.
- [ ] bestehende Azure-Automation-Logik als Referenz verwenden.
- [ ] bereits korrekte VM-Power-States überspringen.
- [ ] Fehler-/Timeout-Verhalten definieren.
- [ ] Logging definieren.
- [ ] sicherstellen, dass Dependency-Automation den AVD Session Host nicht parallel zum nativen AVD Autoscale steuert.
- [ ] Start VM on Connect + Dependency-Start testen.
- [ ] Deallocation + Dependency-Shutdown testen.

### Exit-Kriterium

Abhängige Server werden zuverlässig in richtiger Reihenfolge gestartet/gestoppt, ohne Konflikt mit AVD Autoscale.

---

## Phase 9 – Azure DevOps Migration und CI/CD

### Ziel

Dauerhafte Betriebsplattform ist Azure DevOps.

### Aufgaben

- [ ] Ziel-Azure-DevOps-Projekt festlegen.
- [ ] Repository nach Azure Repos migrieren.
- [ ] Microsoft-Upstream-Update-Strategie nach Migration erhalten.
- [ ] Azure Resource Manager Service Connection mit Workload Identity Federation erstellen.
- [ ] keine Client Secrets verwenden.
- [ ] `pipelines/azure-pipelines.yml` erstellen.
- [ ] Preflight Stage erstellen.
- [ ] Foundation Stage erstellen.
- [ ] Identity Stage als bedingte Stage erstellen.
- [ ] AVD Stage erstellen.
- [ ] Workload Stage erstellen.
- [ ] Configuration Stage erstellen.
- [ ] Validation Stage erstellen.
- [ ] Production Environment mit Approval Gate einrichten.
- [ ] PR Validation einrichten.
- [ ] Bicep Lint ausführen.
- [ ] `bicep build` ausführen.
- [ ] ARM/Bicep Validation ausführen.
- [ ] What-If ausführen.
- [ ] optional PSRule for Azure bewerten.
- [ ] Deployment standardmäßig nicht bei jedem Merge automatisch gegen Kundensubscriptions ausführen.

### Exit-Kriterium

Deployment kann kontrolliert und ohne langlebige Credentials aus Azure DevOps gestartet werden.

---

## Phase 10 – Post-Deployment Validierung

### Automatisierte Prüfungen

- [ ] alle erwarteten Ressourcen vorhanden.
- [ ] nur die für das gewählte Identity-Modell erwarteten Identity-Ressourcen vorhanden.
- [ ] Session Host im Host Pool registriert.
- [ ] Session Host `Available`.
- [ ] Domain Join korrekt (wenn ADDS).
- [ ] kein unnötiger Domain Controller vorhanden (wenn Entra-only).
- [ ] Entra Join korrekt (wenn Entra).
- [ ] AVD Agent/Bootloader gesund.
- [ ] Benutzer kann sich anmelden.
- [ ] FSLogix-Profil wird erstellt/gemountet.
- [ ] Profil bleibt nach Session-Host-Rebuild erhalten.
- [ ] Start VM on Connect funktioniert.
- [ ] Scaling Plan funktioniert.
- [ ] Management-VM nach Deployment `Deallocated`, sofern sie im gewählten Szenario bereitgestellt wurde.
- [ ] keine Public IPs vorhanden, sofern nicht ausdrücklich vorgesehen.
- [ ] keine unerwarteten Enterprise-Kostenkomponenten aktiv.

---

# 8. Testmatrix

Mindestens folgende Szenarien müssen vor Release 1.0 getestet werden:

| ID | Szenario | Status |
|---|---|---|
| T01 | Entra ID, 1 Session Host, **kein DC**, ohne FSLogix | `[ ]` |
| T02 | Entra ID, 1 Session Host, **kein klassischer DC**, FSLogix/Entra Kerberos | `[ ]` |
| T03 | Neue AD-Gesamtstruktur, 1 DC, 1 Session Host, FSLogix | `[ ]` |
| T04 | Existing/Migrated AD DS, 1 Session Host, FSLogix | `[ ]` |
| T05 | AD DS, 2 Session Hosts | `[ ]` |
| T06 | AD DS, AVD + APP + SQL + Dependency Automation | `[ ]` |
| T07 | erneutes identisches Deployment / Idempotenz | `[ ]` |
| T08 | Session Host ersetzen / Profil bleibt erhalten | `[ ]` |
| T09 | Management-VM Start → Wartung → Deallocate | `[ ]` |

Für jedes Szenario:

```text
Bicep Build
→ Validation
→ What-If
→ Deployment
→ Login Test
→ Funktionsprüfung
→ Re-Deployment/What-If
→ Kostenprüfung
```

---

# 9. Kosten-Guardrails

Diese Regeln sind Teil der Architektur und nicht nur Empfehlungen.

## Standardmäßig vermeiden

- Azure Firewall,
- Domain Controller ohne technische Notwendigkeit,
- zwei DCs ohne Verfügbarkeitsanforderung,
- zwei Session Hosts ohne Bedarf,
- 24/7 laufende Management-VM,
- unnötige Premium-Disks,
- unnötige Private Endpoints,
- unnötige Defender-Pläne,
- unnötige DDoS-Pläne,
- Hub-Spoke ohne tatsächlichen Mehrwert.

## Standardmäßig bevorzugen

- Entra ID Only ohne DC, **wenn die Kundenanwendungen und technischen Anforderungen dies zulassen**,
- 1 Session Host,
- bei tatsächlich erforderlichem AD DS: 1 DC im kostenoptimierten New-Forest-Szenario,
- ein VNet,
- ein NAT Gateway für relevante Subnetze desselben VNets,
- Start VM on Connect,
- AVD Scaling Plan,
- kleine deallokierte Management-VM, sofern diese benötigt wird,
- Standard-LRS für die Management-VM,
- parametrische VM-Größen statt überdimensionierter Defaults.

## Aufgabe vor Release

- [ ] Kostenmodell je Standardprofil erstellen.
- [ ] monatliche Fixkosten von laufzeitabhängigen Kosten trennen.
- [ ] Entra-only- und ADDS-Profile kostenmäßig getrennt ausweisen.
- [ ] Kosten von optionalen Features separat ausweisen.
- [ ] Beispielkalkulation für `<30 Mitarbeiter` erstellen.

---

# 10. Security-Guardrails

- [ ] keine Secrets in Git/Azure DevOps Repos.
- [ ] Key Vault bzw. sichere Pipeline-Secret-Verarbeitung.
- [ ] Workload Identity Federation für Azure DevOps.
- [ ] keine Public IP auf AVD Session Hosts.
- [ ] kein offenes Inbound RDP als Standard.
- [ ] Trusted Launch / Secure Boot / vTPM beibehalten, sofern Ziel-SKU/Image kompatibel.
- [ ] Least-Privilege RBAC.
- [ ] Domain-Join-Konto nur in ADDS-Szenarien und nicht pauschal als Domain Admin verwenden.
- [ ] Backup/Restore für DC dokumentieren, wenn AD DS eingesetzt wird.
- [ ] Kundenparameter dürfen keine Passwörter enthalten.

---

# 11. Preflight-Prüfungen

Die Microsoft-Prerequisites sollen soweit möglich nicht manuell abgearbeitet, sondern von einer Preflight-Stage validiert bzw. automatisiert werden.

### Prüfen/automatisieren

- [ ] `Microsoft.DesktopVirtualization` registriert.
- [ ] `Microsoft.Compute` registriert.
- [ ] `Microsoft.Network` registriert.
- [ ] `Microsoft.Storage` registriert.
- [ ] Quotas für gewünschte VM-SKUs vorhanden.
- [ ] Zielregion unterstützt gewünschte VM-Größen.
- [ ] Marketplace Images vorhanden.
- [ ] AVD Enterprise Application Object ID ermitteln.
- [ ] Netzwerk-/Subnetzrechte vorhanden.
- [ ] erforderliche AVD-/Microsoft-Endpunkte erreichbar.
- [ ] Identity-Modell vor Deployment eindeutig gewählt.
- [ ] AD/DNS Health Check nur bei ADDS.
- [ ] Join-Berechtigungen nur bei ADDS prüfen.
- [ ] AVD-Lizenzierung als Kunden-/Projektvoraussetzung dokumentieren.

---

# 12. Aufgabenübersicht / Priorisierung

| ID | Priorität | Aufgabe | Status | Abhängigkeit |
|---|---:|---|---|---|
| A001 | P0 | Accelerator-Import | `[x]` | – |
| A002 | P0 | Umsetzungsplan als Single Source of Truth | `[x]` | A001 |
| A003 | P0 | unveränderte Baseline build/validieren | `[ ]` | A001 |
| A004 | P0 | MSP Foundation / Existing-VNet-Modell | `[ ]` | A003 |
| A005 | P0 | Management-VM Größe entkoppeln | `[ ]` | A003 |
| A006 | P0 | Management-VM Disk minimieren | `[ ]` | A003 |
| A007 | P0 | Management-VM automatisch deallokieren | `[ ]` | A005/A006 |
| A008 | P0 | Accelerator Script-/DSC-Versionen pinnen | `[ ]` | A003 |
| A009 | P0 | optionales New-Forest-DC-Modul für ADDS | `[ ]` | A004 |
| A010 | P0 | Accelerator mit Entra-only Profile **ohne DC** testen | `[ ]` | A004/A005 |
| A011 | P0 | Accelerator mit ADDS/NewForest testen | `[ ]` | A009/A005 |
| A012 | P1 | FSLogix KMU-Default festlegen | `[ ]` | A010/A011 |
| A013 | P1 | APP-Modul | `[ ]` | A004 |
| A014 | P1 | SQL-Modul | `[ ]` | A004 |
| A015 | P1 | Dependency Automation | `[ ]` | A013/A014 |
| A016 | P1 | Azure DevOps Repository-Migration | `[ ]` | stabile Codebasis |
| A017 | P1 | Azure Pipelines / WIF / Approvals | `[ ]` | A016 |
| A018 | P1 | Post-Deployment-Tests | `[ ]` | A010/A011 |
| A019 | P1 | Kostenmodell Standardprofile | `[ ]` | A010/A011/A012 |
| A020 | P2 | Resilient-Profil mit 2 DCs/Hosts für entsprechende Anforderungen | `[ ]` | Release-Baseline |
| A021 | P2 | Existing Hub/Hub-Spoke optional | `[ ]` | Release-Baseline |

---

# 13. Offene Architekturentscheidungen

Diese Punkte dürfen nicht stillschweigend entschieden werden.

## O001 – Entra Synchronisation

**OFFEN – nur für ADDS NewForest/MigrateExistingForest relevant**

Für NewForest/MigrateExistingForest festlegen:

```text
Entra Cloud Sync
vs.
Entra Connect Sync
```

---

## O002 – FSLogix Default

**OFFEN**

Festlegen, ob FSLogix in den Standardprofilen automatisch aktiviert wird oder bewusst optional bleibt.

---

## O003 – Azure Files Tier

**OFFEN**

Für KMU bewerten:

```text
Standard LRS
vs.
Premium LRS
```

---

## O004 – Azure Files Netzwerkzugriff

**OFFEN**

Default festlegen:

```text
Service Endpoint
vs.
Private Endpoint
```

Kosten und Komplexität sind ausdrücklich zu berücksichtigen.

---

## O005 – Monitoring Baseline

**OFFEN**

Festlegen, welche AVD-/VM-/Guest-Monitoring-Komponenten zwingend in jedes KMU-Profil gehören und welche optional bleiben.

---

## O006 – Standard Session Host SKU

**OFFEN**

Keine harte Festlegung ohne Messwerte. Microsoft-Default und reale Kundenauslastung vergleichen.

---

# 14. Nicht-Ziele für Release 1.0

Für die erste stabile Version nicht priorisieren:

- Azure Firewall als Standard,
- klassisches AD DS als Zwang für jeden Kunden,
- komplexe Enterprise Hub-Spoke Landing Zone,
- vollständige HA für alle Komponenten,
- SQL Always On,
- zwei APP-Server,
- zwingendes AVD Private Link,
- zwingende Availability Zones,
- kundenspezifische Branchensoftware vollständig automatisiert installieren,
- unnötige Abweichungen vom Microsoft Accelerator.

---

# 15. Upstream-Update-Strategie

Ziel:

Microsoft-Änderungen weiter übernehmen können, ohne unsere KMU-Anpassungen zu verlieren.

Geplanter Ablauf:

```text
Microsoft Upstream Update
        ↓
Upstream Änderungen prüfen
        ↓
Merge in separaten Integrations-Branch
        ↓
Patch-Konflikte prüfen
        ↓
Bicep Build
        ↓
What-If
        ↓
Testmatrix
        ↓
Freigabe
        ↓
Übernahme in MSP Main
```

Regel:

**Keine automatische ungeprüfte Synchronisation von Microsoft `main` in produktiven MSP-Code.**

---

# 16. Entscheidungsprotokoll

## 2026-08-07 – Accelerator statt vollständiger Eigenentwicklung

**Entscheidung:** Microsoft AVD Accelerator wird als AVD-Core verwendet. Eigene Logik wird als dünne MSP-Schicht ergänzt.

**Grund:** Weniger Eigenverantwortung für AVD Agent Registration, Host Pools, Application Groups, Workspaces, Scaling, FSLogix-Integration und API-Änderungen.

---

## 2026-08-07 – Keine Azure Firewall als KMU-Default

**Entscheidung:** Azure Firewall ist kein Bestandteil des Standardprofils.

**Grund:** Für Kunden mit typischerweise unter 30 Mitarbeitern wirtschaftlich nicht sinnvoll.

---

## 2026-08-07 – Entra ID Only bevorzugen, AD DS nur bei Bedarf

**Entscheidung:** Ein klassischer Domain Controller ist kein genereller Bestandteil der Zielarchitektur. Wenn Anwendungen und Dienste Entra-only unterstützen, wird kein DC bereitgestellt. AD DS wird als optionale Dependency eingesetzt, wenn insbesondere Branchensoftware klassisches AD voraussetzt.

Wenn AD DS benötigt wird und eine neue Gesamtstruktur erstellt werden muss, kann das kostenoptimierte KMU-Profil mit einem DC betrieben werden. Zwei DCs sind eine optionale resiliente Variante.

**Grund:** Die kleinste technisch ausreichende Architektur soll gewählt werden. Ein nicht benötigter DC verursacht Kosten und Betriebsaufwand ohne Mehrwert.

---

## 2026-08-07 – On-Prem-Server ist nur Migrationsquelle

**Entscheidung:** Nach einer vollständigen Azure-Neuimplementierung wird der alte physische Server abgeschaltet. NAS ist optional, aber keine Azure-Betriebsabhängigkeit.

---

## 2026-08-07 – Netzwerk wird vor dem Accelerator bereitgestellt

**Entscheidung:** MSP Foundation erzeugt VNet/Subnetze. Accelerator läuft standardmäßig mit `createAvdVnet = false` gegen bestehende Subnetze.

**Grund:** Das Netzwerk wird unabhängig vom Identity-Modell benötigt. Bei ADDS kann der DC vor dem AVD-Deployment aufgebaut werden; bei Entra-only wird die Identity-/DC-Stufe übersprungen.

---

## 2026-08-07 – APP/SQL separat vom Accelerator

**Entscheidung:** APP-/SQL-Server werden als separate Bicep-Module implementiert. Ihr Vorhandensein erzwingt nicht automatisch AD DS; die konkrete Branchensoftware bestimmt die Identity-Anforderung.

---

## 2026-08-07 – Management-VM behalten, aber deallokieren

**Entscheidung:** Die Accelerator-Management-VM wird nach erfolgreicher Verwendung **nicht gelöscht**, sondern automatisch deallokiert.

**Grund:** Fast keine laufenden Kosten, eine vorhandene Domain Membership bleibt bei ADDS erhalten und die VM kann für spätere Storage-/FSLogix-Wartung erneut verwendet werden.

---

## 2026-08-07 – Management-VM entkoppeln und stark verkleinern

**Entscheidung:** Management-VM darf nicht dieselbe Größe wie der Session Host übernehmen. Kleine Burst-VM ist das Ziel; `Standard_B2als_v2` ist der erste zu testende Kandidat.

---

## 2026-08-07 – Azure DevOps als Zielplattform

**Entscheidung:** Langfristig Azure Repos + Azure Pipelines + Workload Identity Federation + Environments/Approvals statt GitHub Actions.

---

# 17. Definition of Done für Release 1.0

Release 1.0 ist erst erreicht, wenn mindestens folgende Bedingungen erfüllt sind:

- [ ] Microsoft Accelerator ist mit kleinem dokumentiertem Patch-Delta integriert.
- [ ] Foundation ist reproduzierbar, idempotent und unabhängig von einem DC deploybar.
- [ ] Entra-only Deployment **ohne Domain Controller** funktioniert.
- [ ] optionale ADDS/NewForest-Variante mit einem DC funktioniert.
- [ ] Accelerator nutzt bestehendes MSP-VNet.
- [ ] Management-VM ist klein dimensioniert.
- [ ] Management-VM verwendet minimale sinnvolle Disk.
- [ ] Management-VM wird nach Einsatz automatisch deallokiert, sofern sie im gewählten Szenario bereitgestellt wird.
- [ ] keine Azure Firewall im KMU-Default.
- [ ] Start VM on Connect funktioniert.
- [ ] AVD Scaling Plan funktioniert.
- [ ] FSLogix-Entscheidung ist getroffen und mindestens ein produktionsfähiges Profil getestet.
- [ ] Session Host kann ersetzt werden, ohne dass Benutzerprofil verloren geht (bei FSLogix-Profil).
- [ ] Azure DevOps Pipeline funktioniert mit Workload Identity Federation.
- [ ] Production Approval ist eingerichtet.
- [ ] Preflight und Post-Deployment Validation existieren.
- [ ] Kostenprofile für einen typischen Entra-only- und einen ADDS-Kunden mit `<30 Mitarbeitern` sind dokumentiert.
- [ ] README verweist auf diesen Umsetzungsplan.

---

# 18. Pflege dieses Dokuments

Bei jeder zukünftigen Arbeitssitzung zuerst:

1. `UMSETZUNGSPLAN.md` lesen.
2. aktuellen Aufgabenstatus prüfen.
3. neue Entscheidung oder neue Aufgabe hier eintragen.
4. erst danach Code ändern.
5. nach Abschluss Status und Entscheidungsprotokoll aktualisieren.

Damit bleibt der technische Kontext unabhängig von Chatverläufen dauerhaft im Repository erhalten.
