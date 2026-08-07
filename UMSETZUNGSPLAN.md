# MSP AVD Accelerator – Umsetzungsplan

> **Status:** Arbeitsdokument / Single Source of Truth  
> **Stand:** 2026-08-07  
> **Repository:** `Vertax1337/P-NETRATOR`  
> **Microsoft-Upstream:** `Azure/avdaccelerator`  
> **Importierter Upstream-Stand:** `63c55c857f77783c01812c935957cd8c9c67a5a7`  
> **Import-Commit in P-NETRATOR:** `aff8e2823011970d1f96fac078dfd5e0e28de2da`  
> **Ursprünglicher P-NETRATOR-Stand:** `archive/p-netrator-original`

---

# 1. Zweck und Arbeitsregel

Dieses Dokument ist die verbindliche Aufgaben-, Befund- und Entscheidungsübersicht für die Anpassung des Microsoft **Azure Virtual Desktop Accelerator** an eine kostenoptimierte, wiederverwendbare MSP-Lösung für kleine und mittlere Unternehmen.

Hintergrund ist ausdrücklich, dass technische Erkenntnisse und Entscheidungen aus längeren Arbeitssitzungen nicht verloren gehen oder später unbemerkt anders interpretiert werden sollen.

Daher gelten folgende Regeln:

- Dieses Dokument ist die **Single Source of Truth** für die Umsetzung.
- Bereits getroffene Architekturentscheidungen werden hier dauerhaft dokumentiert.
- Noch nicht entschiedene Punkte werden ausdrücklich mit **OFFEN** gekennzeichnet.
- Neue Findings werden vor Umsetzung einer der folgenden Kategorien zugeordnet:
  - **Accelerator-Patch** – Änderung am importierten Microsoft-Code erforderlich.
  - **MSP-Foundation-Aufgabe** – muss unsere vorgeschaltete Infrastruktur-/Pipeline-Schicht lösen; Microsoft-Code soll dafür nicht verändert werden.
  - **Feature-abhängiger Patch** – nur notwendig, wenn das entsprechende Feature tatsächlich aktiviert wird.
  - **P2 / zurückgestellt** – für Release 1.0 nicht erforderlich, aber dokumentiert.
- Neue Aufgaben werden in die Aufgabenübersicht aufgenommen, bevor sie umgesetzt werden.
- Erledigte Aufgaben werden erst nach technischer Validierung auf `[x]` gesetzt.
- Jede Änderung am Microsoft-Code muss begründet werden, damit der Abstand zum Upstream möglichst klein bleibt.
- Bei relevanten Architekturentscheidungen wird das Entscheidungsprotokoll aktualisiert.
- Vor jeder neuen Arbeitssitzung wird zuerst diese Datei gelesen.

---

# 2. Projektziel

Das Ziel ist **keine vollständige Eigenimplementierung von Azure Virtual Desktop**.

Der Microsoft AVD Accelerator bleibt die **AVD Workload Deployment Engine**. Darum herum entsteht eine dünne MSP-Schicht für:

- kostenoptimierte KMU-Defaults,
- Netzwerk/Foundation,
- Identity-Auswahl,
- optionale AD-DS-Infrastruktur,
- Migration bestehender AD-Gesamtstrukturen,
- optionale APP-/SQL-Server,
- Runtime-Abhängigkeiten zwischen AVD, APP und SQL,
- Azure DevOps Orchestrierung,
- private/versionierte Runtime-Artefakte,
- Kundenparameter,
- Preflight-Prüfungen,
- Post-Deployment-Validierung,
- Kostenkontrolle,
- reproduzierbare und idempotente Deployments.

Grundprinzip:

```text
MSP Foundation
      ↓
Identity-Auswahl
      ├── Entra ID Only → kein DC erforderlich
      └── klassisches AD DS benötigt → Identity-/DC-Dependency bereitstellen
      ↓
Microsoft AVD Accelerator
      ↓
optionale Workloads
      ↓
Konfiguration / Automation
      ↓
Validierung
```

Der Microsoft-Code soll **so wenig wie möglich, aber so viel wie für einen sicheren und reproduzierbaren Produktionsbetrieb erforderlich verändert werden**.

---

# 3. Zielgruppe

Primäre Zielgruppe:

- KMU mit typischerweise **unter 30 Mitarbeitern**,
- meistens **1 AVD Session Host**,
- zunächst maximal 2–3 Session Hosts,
- teilweise reine Entra-ID-/Cloud-Szenarien ohne klassischen Domain Controller,
- häufig klassische Branchensoftware,
- teilweise SQL-/APP-Abhängigkeiten,
- hohe Kostenpriorität,
- keine unnötige Enterprise-Komplexität.

Die Lösung muss skalierbar bleiben, darf aber nicht standardmäßig Enterprise-Komponenten ausrollen, die für diese Kundengröße wirtschaftlich nicht sinnvoll sind.

---

# 4. Verbindliche Architekturleitplanken

## 4.1 Microsoft Accelerator als Basis

**BESCHLOSSEN**

- Microsoft AVD Accelerator bleibt die Basis für den AVD-Core.
- Keine vollständige Neuimplementierung von Host Pool, Workspace, Application Group, Session-Host-Grunddeployment etc.
- Microsoft-Module unter `workload/` und `avm/` bleiben soweit möglich unverändert.
- Eigene MSP-Erweiterungen werden außerhalb der Microsoft-Struktur abgelegt.
- Upstream-Updates werden kontrolliert übernommen und niemals ungeprüft produktiv ausgerollt.

```text
Microsoft Accelerator = AVD Core
MSP Layer             = Foundation + Identity + Workloads + Orchestrierung + Hardening
```

---

## 4.2 Kostenorientierung

**BESCHLOSSEN**

Nicht standardmäßig bereitstellen:

- Azure Firewall,
- DDoS Network Protection Standard,
- AVD Private Link,
- erzwungene Availability Zones,
- erzwungene zwei Session Hosts,
- Domain Controller ohne technische/fachliche Notwendigkeit,
- erzwungene zwei Domain Controller,
- unnötige dauerhaft laufende Management-VMs,
- unnötige Hub-Spoke-Topologien,
- Premium-Komponenten ohne fachlichen Mehrwert.

Enterprise-/HA-Komponenten dürfen später optional aktivierbar sein.

---

## 4.3 Session Hosts

**BESCHLOSSEN**

Standard:

```text
Session Host Count = 1
```

Die Implementierung soll zunächst 1–3 Session Hosts unterstützen.

VM-Größe bleibt vollständig parametrierbar.

---

## 4.4 Identity und Domain Controller

**BESCHLOSSEN**

Ein klassischer Domain Controller ist **kein genereller Bestandteil des KMU-Standards**.

### Fall A – Entra ID Only

Wenn alle benötigten Anwendungen und Dienste ohne klassisches AD DS betrieben werden können:

```text
Entra ID
AVD01
kein Domain Controller
```

Kein DC wird nur aus Gewohnheit oder wegen des historischen On-Prem-Designs bereitgestellt.

### Fall B – Klassisches AD DS erforderlich

AD DS wird nur bereitgestellt, wenn beispielsweise eine Branchensoftware:

- klassischen Domain Join benötigt,
- LDAP/Kerberos/NTLM oder andere AD-DS-Funktionen voraussetzt,
- oder eine andere technische Anforderung gegen Entra-only spricht.

Kostenoptimiertes New-Forest-Profil:

```text
DC01
```

Resilientes Profil:

```text
DC01
DC02
```

Ein einzelner DC ist ein bewusst akzeptierter SPOF und benötigt mindestens:

- statische private IP,
- Azure Backup,
- Monitoring,
- dokumentierten Restore-Prozess,
- keinen öffentlichen IP-Zugriff,
- keine unnötigen Zusatzrollen,
- 24/7-Betrieb.

---

## 4.5 On-Premises nach Migration

**BESCHLOSSEN**

Der bestehende physische Server ist nur **IST-Zustand bzw. Migrationsquelle**.

Nach erfolgreicher Migration ist Azure eigenständig betriebsfähig und der alte physische Server wird abgeschaltet.

Eine NAS darf optional für Backup/Archiv/Sonderfälle verbleiben, ist aber keine Betriebsabhängigkeit der Azure-Umgebung.

Ein dauerhaftes Zielbild „AVD in Azure + einziger produktiver DC On-Premises“ ist **kein Standard-Zielbild**.

---

## 4.6 Netzwerk

**BESCHLOSSEN**

Standard ist eine einfache Single-VNet-Architektur:

```text
VNet Kunde
├── snet-avd
│   └── AVD01 (+ optional AVD02)
├── snet-infra
│   ├── DC01 optional
│   ├── APP01 optional
│   └── SQL01 optional
├── snet-private-endpoints optional
└── GatewaySubnet optional
```

Grundsätze:

- kein Azure Firewall Default,
- kein Hub-Spoke Default,
- keine Public IP auf Session Hosts,
- kein standardmäßig geöffnetes RDP/3389,
- expliziter Outbound-Zugriff,
- NAT Gateway als bevorzugter kosteneffizienter Outbound-Pfad,
- ein NAT Gateway kann mehrere relevante Subnetze desselben VNets bedienen.

---

## 4.7 Foundation vor dem Accelerator

**BESCHLOSSEN**

Das Netzwerk wird standardmäßig **nicht** durch den Accelerator erzeugt.

Die MSP Foundation erstellt zuerst VNet/Subnetze und notwendige Netzwerkeigenschaften.

Der Accelerator wird anschließend mit:

```bicep
createAvdVnet = false
```

gegen die bestehende MSP-Foundation ausgeführt.

Deployment-Reihenfolge:

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

## 4.8 Identity-Zielmodell

**BESCHLOSSEN als Architektur, Detailparameter noch umzusetzen**

Unterstützte Zielvarianten mindestens:

```text
EntraID
ADDS
EntraDS
EntraIDKerberos
```

Für AD DS zusätzlich:

```text
NewForest
MigrateExistingForest
ExistingAzureForest
```

`MigrateExistingForest` ist ein Migrationsworkflow und keine dauerhafte On-Prem-Abhängigkeit.

**OFFEN:** Entra Cloud Sync vs. Entra Connect Sync für ADDS-Standardszenarien festlegen.

---

## 4.9 APP- und SQL-Server

**BESCHLOSSEN**

APP/SQL werden nicht in den Microsoft Accelerator eingebaut, sondern als separate optionale MSP-Bicep-Workloads implementiert.

Ihr Vorhandensein impliziert nicht automatisch AD DS. Die konkrete Branchensoftware bestimmt die Identity-Anforderung.

Runtime-Abhängigkeit, wenn benötigt:

```text
Start:
SQL → Health → APP

Stop:
APP vollständig stoppen → SQL stoppen
```

---

## 4.10 Power Management

**GRUNDSATZ BESCHLOSSEN, konkrete Default-Variante noch zu validieren**

Für AVD möglichst native Funktionen verwenden:

- Start VM on Connect,
- AVD Scaling Plan/Autoscale.

Eigene Automation darf nicht parallel denselben Session Host power-managen, wenn AVD Autoscale dafür verantwortlich ist.

Eigene Automation bleibt für APP-/SQL-Dependencies vorgesehen.

**OFFEN / TEST:** `NativeAutoscale` gegen die vorhandene eigene AutoShutdown-/Deallocate-Logik im 1-Host-KMU-Szenario vergleichen. Kein endgültiger Wechsel, bevor beide Varianten technisch getestet wurden.

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

Keine langlebigen Client Secrets für Azure-Deployment.

Ziel-Pipeline:

```text
PRECHECK
   ↓
FOUNDATION
   ↓
IDENTITY (conditional)
   ↓
RUNTIME ARTIFACT PREPARE
   ↓
AVD ACCELERATOR
   ↓
WORKLOAD
   ↓
CONFIGURATION
   ↓
VALIDATION
   ↓
MGMT VM DEALLOCATE
```

---

# 5. Geplante Repository-Struktur

```text
P-NETRATOR/
├── avm/                         # Microsoft / möglichst unverändert
├── workload/                    # Microsoft AVD Accelerator / kontrolliertes Patchset
├── msp/
│   ├── foundation/
│   ├── identity/
│   │   ├── new-forest/
│   │   ├── migrate-existing-forest/
│   │   └── existing-azure-forest/
│   ├── workloads/
│   │   ├── app-server/
│   │   └── sql-server/
│   ├── profiles/
│   │   ├── entra-small.bicepparam
│   │   ├── adds-small.bicepparam
│   │   └── ...
│   └── customers/
│       └── example/
├── runtime/
│   ├── scripts/
│   ├── dsc/
│   └── release-manifest/
├── pipelines/
│   ├── azure-pipelines.yml
│   └── templates/
└── UMSETZUNGSPLAN.md
```

Kundendateien dürfen keine Kennwörter oder andere Secrets enthalten.

---

# 6. Deep-Dive des importierten Accelerators – Ausgangslage

Am 2026-08-07 wurde der importierte Accelerator nochmals systematisch gegen unser Projektziel geprüft.

Geprüfte Schwerpunkte:

- Bicep-/AVM-Struktur,
- Marketplace Images,
- Preview-/API-Abhängigkeiten,
- Naming/Idempotenz,
- AVD Agent/Bootloader Bootstrap,
- Host-Pool Registration Token,
- Storage-/FSLogix-Konfiguration,
- Management-VM,
- Credential-/Secret-Verarbeitung,
- externe Runtime-Abhängigkeiten,
- Monitoring,
- Scaling Plan,
- Existing-VNet-Verhalten,
- aktuelle offene Microsoft-Issues/PRs.

## 6.1 Upstream-Status

Der importierte Stand `63c55c857f77783c01812c935957cd8c9c67a5a7` ist am 2026-08-07 weiterhin der aktuelle Microsoft-`main`-Stand. Der letzte Commit auf diesem Stand stammt vom 2026-01-21.

Gleichzeitig existieren wichtige noch nicht gemergte Upstream-Fixes. Relevant für uns sind insbesondere:

- `Azure/avdaccelerator#874` – kaputter AVD-Bootloader-Download,
- PR `#873` – Agent-/Bootloader-URLs auf stabile Microsoft fwlinks umstellen,
- `#869` – zu breite FSLogix-NTFS-Rechte bei EntraIDKerberos,
- `#867` / PR `#870` – AVD Insights DCR/Performance Counter Problem auf Win11 24H2/25H2,
- `#866` – AzureFilesDomainJoin/DSCStorageScripts Retry-/File-Lock-Problem,
- PR `#851` – Custom Image/VDOT durch unversionierte Upstream-Abhängigkeit gebrochen,
- PR `#847` – Availability-Zone-/Redeployment-Probleme,
- PR `#850` – neuer Host-Pool-Managementpfad `AutomaticSessionHostConfiguration` (Preview),
- `#872` – GPU-DINE-Policy ohne erforderliche Role Assignment.

**Konsequenz:** Upstream-Pflege darf zukünftig nicht nur `main` vergleichen, sondern muss auch relevante offene Issues und PRs prüfen.

---

# 7. Deep-Dive Findings – konsolidierte Matrix

Die folgende Tabelle ist die verbindliche Übersicht der gefundenen Probleme und Maßnahmen.

| ID | Prio | Kategorie | Befund | Auswirkung auf unser Projekt | Empfohlene Maßnahme |
|---|---:|---|---|---|---|
| F001 | P0 | Accelerator-Patch | Management-VM übernimmt Session-Host-Größe | unnötige Compute-Kosten | eigene `managementVmSize`; kleine Burst-SKU testen |
| F002 | P0 | Accelerator-Patch | Management-VM OS-Disk nicht konsequent kostenminimal | unnötige persistente Kosten | Small-Disk-Image, Standard_LRS, minimale technisch sinnvolle Disk |
| F003 | P0 | Pipeline/MSP | Management-VM bleibt nach Arbeit aktiv | unnötige Compute-Kosten | nach erfolgreicher Konfiguration automatisch **deallokieren**, nicht löschen |
| F004 | P0 | Accelerator-Patch | Management-VM nutzt Windows Server 2022 Legacy-Offer `WindowsServer` | veraltete Marketplace-Referenz | auf aktuelles `windowsserver2022` Offer migrieren und Ziel-SKU in Germany West Central validieren |
| F005 | P0 | Accelerator-Patch | Script-/DSC-Quellen zeigen auf öffentliche `main`-/GitHub-Artefakte | getesteter Commit kann später anderen Code ausführen | Runtime-Artefakte versionieren und aus eigener Release-Quelle bereitstellen |
| F006 | P0 | Accelerator-Patch | persistente Ressourcennamen hängen indirekt von `utcNow()` ab | Re-Deployment kann neue KV-/Storage-Ressourcen erzeugen | deterministisches Naming ohne Zeitstempel |
| F007 | P0 | Accelerator-Patch | gemeinsamer `CreationTimeUTC`-Tag wird bei Re-Deployment mit neuem `time` geändert | unnötiger What-If-/Tag-Drift auf vielen Ressourcen | Creation-Timestamp nicht als bei jedem Deployment neu berechneten Common-Tag führen; Deploymentzeit separat protokollieren |
| F008 | P0 | Accelerator-Patch | AVD Bootloader URL `RWrxrH` ist bereits kaputt | Session-Host-Provisionierung kann heute scheitern | mindestens Upstream PR #873 übernehmen; Bootstrap-URLs zentralisieren |
| F009 | P0 | Accelerator-Patch | Host-Pool-Modul erzeugt bei jedem Deployment neuen Registration Token (`utcNow` + `Update`) | eigentlich harmlose Re-Deployments verändern Session-Host-Bootstrap | Token nur erzeugen/rotieren, wenn neue Hosts registriert oder explizit repariert werden |
| F010 | P0 | Accelerator-Patch | Session-Host-Skript installiert Agent/Bootloader ohne Vorprüfung erneut | unnötige MSI-Läufe/Reboots und schlechte Idempotenz | Zustand/Version/Registration prüfen; Bootstrap idempotent gestalten |
| F011 | P0 | Accelerator-Patch | Session-Host-Skript setzt `NoAutoUpdate = 1` | produktiver Host kann ohne eigenes Replacement-/Patchmodell ungepatcht bleiben | Verhalten parametrieren; Default erst nach festgelegtem Patch-/Replacement-Modell deaktivieren |
| F012 | P0 | Accelerator-Patch AD/FSLogix | Domain-Join-Passwort wird in einen PowerShell-Befehlsstring eingebaut und dieser geloggt | Credential-Leak in Logs/Extension-Kontext möglich | keine Secret-Werte in Kommando-/Logstrings; `Invoke-Expression` mit Secret entfernen |
| F013 | P0 | Accelerator-Patch AD/FSLogix | DSC erlaubt Plaintext Password (`PSDscAllowPlainTextPassword = true`) | Secret kann in MOF/temporären Dateien persistieren | sicheren Ausführungsweg entwerfen; wenn DSC bleibt: Credentials verschlüsseln bzw. unmittelbaren sicheren Cleanup erzwingen |
| F014 | P0 | Accelerator-Patch AD/FSLogix | MOF-/DSC-Cleanup ist auskommentiert | Credential-/Konfigurationsartefakte bleiben auf deallokierter MGMT-VM | MOF, ZIP, temporäre Dateien und sensitive Logs nach erfolgreichem Lauf sicher entfernen |
| F015 | P0 | Accelerator-Patch AD/FSLogix | Storage-Skript deaktiviert Windows Firewall für Domain/Public/Private und stellt sie nicht wieder her | wiederverwendete MGMT-VM bleibt dauerhaft ohne Host-Firewall | Firewall nicht global abschalten; wenn temporär zwingend, vorherigen Zustand erfassen und in `finally` wiederherstellen |
| F016 | P0 | Accelerator-Patch AD/FSLogix | Join-Account wird lokaler Administrator auf MGMT-VM und nicht entfernt | unnötige dauerhafte Privilegierung | Privileg nur temporär und nach Abschluss entfernen; Least-Privilege-Join-Konto verwenden |
| F017 | P0 | Accelerator-Patch AD/FSLogix | DSC `TestScript` betrachtet vorhandenes AD-Computerobjekt als vollständigen Erfolg | Teilfehler nach Domain Join können beim Retry übersprungen werden, z. B. fehlende ACLs | Domain-Join-, Azure-Files- und ACL-Zustand getrennt prüfen; Retry muss fehlende Schritte nachholen |
| F018 | P1/P0 AD/FSLogix | Accelerator-Patch | AzureFilesDomainJoin kann bei erneutem Lauf an vorhandener/gesperrter `DSCStorageScripts.zip` scheitern (Upstream #866) | Wiederholbarkeit/Wartung unzuverlässig | temporären Arbeitsordner pro Lauf oder robusten Cleanup/Retry implementieren |
| F019 | P0 | Feature-Patch Entra Kerberos | Upstream #869: FSLogix-NTFS-Rechte bei EntraIDKerberos zu breit | Benutzer können ggf. andere Profile sehen/mounten | ACL-Modell korrigieren und expliziten Cross-Profile-Denial-Test einführen |
| F020 | P0/P1 | Feature-/Architekturentscheidung | `EntraID` + FSLogix verwendet Storage Account Key/`cmdkey` | langlebiger Storage-Key auf Session Hosts, Rotation/Least Privilege schlecht | für Entra-only + FSLogix modernes Entra-Kerberos-/Identity-Modell als Ziel; Storage-Key-Fallback nicht als MSP-Default |
| F021 | P0/P1 | Feature-Patch Monitoring | AVD Insights DCR sammelt Terminal-Services-Counter auf Win11 24H2/25H2 fehlerhaft | Session-/Capacity-Telemetrie unvollständig | Upstream #867/#870 prüfen; nicht blind übernehmen, sondern gewünschte Counter nach Test korrigieren |
| F022 | P0/P1 | Accelerator-Patch | Pooled Scaling Schedule vollständig hart codiert | Kundenzeiten/Force-Logoff können ungeeignet sein | Schedule vollständig parametrieren; `rampDownForceLogoffUsers` im KMU-Profil nicht ungeprüft `true` |
| F023 | P1 | Accelerator-Patch/Runtime | PowerShell-Module werden ohne feste Version zur Laufzeit installiert | identischer Git-Commit kann später anderes Verhalten haben | benötigte Modulversionen pinnen/paketieren und im Runtime-Manifest führen |
| F024 | P1 | Accelerator-Patch | mehrere Baseline-Ressourcen verwenden Preview-APIs | unnötige Preview-Abhängigkeit und Wartungsrisiko | API-Audit: Preview nur behalten, wenn tatsächlich benötigte Property keine stabile API hat |
| F025 | P1 | Release-Engineering | Marketplace Images verwenden häufig `version: latest` | gleicher Release erzeugt zu anderem Zeitpunkt anderen OS-Build | Approved Image Catalog/Release-Manifest; getestete Image-Versionen dokumentieren |
| F026 | P1 | Accelerator-Patch/Kosten | Workload Key Vault wird in AzureCloud standardmäßig `Premium` | unnötiger Enterprise-Default für reinen Secret-Store | `Standard` als KMU-Default, Premium nur bei HSM-/Key-Anforderung |
| F027 | P1 | Security | Storage Managed Identity erhält relativ breite RG-Rollen | Least-Privilege-Abweichung | benötigte Aktionen und Scope reduzieren; Resource-Level-RBAC bevorzugen, soweit praktikabel |
| F028 | P1 | Security/FSLogix | Storage Shared-Key-Zugriff bleibt nutzbar | zusätzliche Angriffsfläche, obwohl Identity-basierter Betrieb möglich sein kann | nach erfolgreicher Identity-Migration `allowSharedKeyAccess=false` prüfen/testen |
| F029 | P1 | Accelerator-Patch | Defender-FSLogix-Exclusions verwenden `$FslogixFileShareName`, obwohl Parameter `$FSLogixFileShare` heißt | Share-bezogene VHD/VHDX-Exclusions können wirkungslos sein | Variable korrigieren; gleichzeitig aktuelle Defender-/FSLogix-Empfehlungen validieren |
| F030 | P1 | Release-Engineering | Bicep und vorcompilierte ARM-JSONs liegen parallel im Repo | Bicep- und Portal-Pfad können nach Fork-Patches auseinanderlaufen | Bicep als einzige Source of Truth; ARM-JSON ausschließlich reproduzierbar im Build erzeugen oder produktiv nicht nutzen |
| F031 | P1 | Supply Chain | einzelne AVM-Unterabhängigkeiten werden weiterhin über `br/public:` referenziert | Build benötigt externe Registry und kann durch externe Änderungen/Verfügbarkeit beeinflusst werden | `bicep restore` kontrolliert im Build; verwendete Modulversionen pinnen; ggf. für Release vendoren |
| F032 | P1 | Accelerator-Patch | mehrere Security-Groups werden teilweise als Array angeboten, Logik verarbeitet aber nur ersten Eintrag | Konfiguration suggeriert mehr Funktionalität als real angewendet wird | Semantik auf eine Gruppe begrenzen oder Multi-Group-Unterstützung sauber implementieren |
| F033 | P0 Foundation | MSP-Foundation | Existing-VNet-Pfad ändert vorhandenes AVD-Subnetz nicht; Storage/KV werden aber auf VNet-Regeln eingeschränkt | ohne Service Endpoints kann Zugriff bei `Private Endpoint = false` scheitern | Foundation setzt erforderliche `Microsoft.Storage` und ggf. `Microsoft.KeyVault` Service Endpoints |
| F034 | P0 Foundation | MSP-Foundation | Accelerator liefert bei Existing-VNet nicht unser gewünschtes explizites KMU-Outbound-Modell | neue/private Subnetze benötigen planbaren Outbound | NAT Gateway in Foundation bereitstellen und relevante Subnetze anbinden |
| F035 | P0 Foundation | Preflight | SKU-/Zone-/Quota-Verfügbarkeit kann Deployment brechen | fehlgeschlagene Kundenrollouts vermeidbar | vor Deployment VM-SKU, Region, Zone und Quota prüfen; Upstream PR #848 als Referenz |
| F036 | P2 | Feature-Patch GPU | GPU-DINE-Policy-Identity erhält keine erforderliche Rolle (Upstream #872) | GPU-Remediation scheitert | nur im GPU-Profil fixen; Rolle möglichst auf VM Contributor o. ä. minimieren |
| F037 | P2 | Feature-Patch Custom Image | VDOT-Upstreamänderungen haben Accelerator-Image-Build gebrochen (PR #851) | Custom-Image-Pfad aktuell nicht blind nutzbar | erst bei Custom-Image-Feature Upstream-Fix übernehmen und externe VDOT-Version pinnen |
| F038 | P2 | Feature-Patch HA | Availability-Zone-Parameter/Re-Deployment haben bekannte Probleme (PR #847) | HA-/Zonenprofile können bei Regionen/Re-Deployments scheitern | vor `Resilient`-Release entsprechenden Fix übernehmen und testen |
| F039 | strategisch | Architektur | neuer Host-Pool-Pfad `AutomaticSessionHostConfiguration` existiert als Preview (PR #850) | heutiger Registration-Token-Bootstrap könnte langfristig ersetzt werden | Release 1.0 bei `Unmanaged` belassen, aber MSP-Schicht nicht eng an heutigen Bootstrap koppeln |

---

# 8. Verbindliches Accelerator-Patchset für Release 1.0

Diese Punkte werden als notwendige Änderungen am importierten Microsoft-Code betrachtet.

## 8.1 Management-VM Kosten

### P0 – Größe entkoppeln

Ziel:

```bicep
param managementVmSize string
```

Erster Kandidat:

```text
Standard_B2als_v2
```

Vor Festlegung regionale Verfügbarkeit und vollständigen Storage-/DSC-Workflow testen.

### P0 – OS-Disk minimieren

- aktuelles Windows Server Small-Disk-Image,
- Standard_LRS,
- keine Data Disk,
- keine künstlich größere Disk.

### P0 – automatisch deallokieren

Nach erfolgreichem Lauf:

```text
Validation erfolgreich
→ Management VM Deallocate
```

Nicht löschen, da Wiederverwendung für spätere Storage-/FSLogix-Wartung vorgesehen ist.

---

## 8.2 Windows-Server-2022-Image

Aktueller Legacy-Pfad:

```text
MicrosoftWindowsServer / WindowsServer / 2022-datacenter-smalldisk-g2
```

Ziel:

- aktuelles Offer `windowsserver2022`,
- passende Small-Disk-SKU,
- tatsächliche Verfügbarkeit in Germany West Central vor Commit validieren.

---

## 8.3 Deterministisches Naming und Deployment-Drift

`utcNow()` darf nicht in persistente Ressourcennamen einfließen.

Zielprinzip:

```bicep
var namingSeed = uniqueString(
  avdWorkloadSubsId,
  deploymentPrefix,
  deploymentEnvironment,
  avdSessionHostLocation
)
```

Zusätzlich:

- global eindeutige Ressourcen wie Storage/KV mit ausreichend langem stabilen Hash versehen,
- `CreationTimeUTC` nicht bei jedem Re-Deployment neu auf alle Common-Tags schreiben,
- Zeitstempel dürfen für Deployment-History/Logs verwendet werden, aber nicht für dauerhafte Resource Identity.

---

## 8.4 Runtime-Artefakte / Supply Chain

Microsoft-`main` darf nicht die Runtime-Quelle eines freigegebenen MSP-Releases sein.

Ziel für Azure DevOps:

```text
Private Azure Repos
→ Pipeline Validate/Package
→ versionierter Runtime-Release
→ privater Azure Blob Container
→ kurzlebiger Zugriff
→ Ziel-VM
```

Aktueller V1-Favorit:

```text
User-Delegation-SAS
read-only
kurze TTL
```

Managed Identity bleibt eine spätere Alternative, insbesondere wenn verschachtelte PowerShell-Downloads vollständig MI-fähig umgebaut wurden.

Bei SAS-Verwendung müssen SAS-haltige `fileUris` in `protectedSettings` liegen.

Runtime Release Manifest soll mindestens enthalten:

```text
MSP Release
Git Commit
Set-SessionHostConfiguration.ps1 Hash
Manual-DSC-Storage-Scripts.ps1 Hash
DSCStorageScripts.zip Hash
PowerShell Module Versions
Marketplace Image Reference
```

---

## 8.5 Agent-/Bootloader-Bootstrap

Mindestens Upstream-Fix #873 übernehmen:

```text
Bootloader → offizieller go.microsoft.com fwlink
Agent      → offizieller go.microsoft.com fwlink
```

Zusätzlich:

- Downloadfehler sauber abbrechen,
- Exit Codes der MSI-Installationen prüfen,
- vorhandenen Agent/Bootloader-Zustand prüfen,
- unnötige Reinstallationen vermeiden,
- Registration-Token-Lifecycle vom normalen Host-Pool-Re-Deployment trennen.

---

## 8.6 Session-Host Patch-/Update-Verhalten

`NoAutoUpdate = 1` darf nicht unreflektiert unser Standard bleiben.

Vor Release festlegen:

```text
A) Patch in Place
oder
B) Immutable/Replace Session Hosts
```

Bis ein getestetes Replacement-/Image-Lifecycle-Verfahren existiert, wird das vollständige Deaktivieren von Windows Update nicht als sicherer Default betrachtet.

---

## 8.7 Management-VM / Storage Credential Hardening

Vor produktivem ADDS/FSLogix-Einsatz zwingend:

- kein Passwort in geloggten PowerShell-Befehlen,
- kein `Invoke-Expression` mit Secret im zusammengesetzten Kommando,
- keine dauerhafte Plaintext-Credential-Ablage,
- keine sensitiven MOF-/ZIP-/Temp-Artefakte nach Lauf,
- Windows Firewall nicht dauerhaft deaktivieren,
- Join-Konto nicht dauerhaft in Local Administrators belassen,
- Fehlerpfade in `try/finally` so bauen, dass Security-Zustände wiederhergestellt werden,
- Log-Dateien auf Secret-Leaks prüfen.

**Implementierungsentscheidung noch offen:** DSC sicher weiterverwenden vs. Storage-Bootstrap ohne credential-haltige Plaintext-MOF neu strukturieren.

---

## 8.8 Storage-Konfiguration idempotent machen

Ein vorhandenes AD-Computerobjekt ist **nicht** ausreichend als Erfolgsnachweis.

Getrennt prüfen:

```text
AD/Storage Identity vorhanden?
Azure Files Authentication korrekt?
Share erreichbar?
Share RBAC korrekt?
Root ACL korrekt?
FSLogix User Isolation korrekt?
```

Ein Retry muss nur fehlende Schritte nachholen.

Temporäre Download-/DSC-Verzeichnisse müssen pro Lauf sauber vorbereitet werden, damit File-Lock-/Stale-ZIP-Probleme (#866) keinen Retry verhindern.

---

## 8.9 Scaling Plan vollständig parametrieren

Nicht übernehmen als feste Wahrheit:

- Ramp-up 07:00,
- Peak 09:00,
- Ramp-down 18:00/19:00,
- Force Logoff `true`,
- 30 Minuten feste Werte.

Zielparameter mindestens:

```text
powerManagementMode
schedule profile
rampUpStart
peakStart
rampDownStart
offPeakStart
minimumHostsPct
capacityThresholdPct
forceLogoffUsers
waitTimeMinutes
stopHostsWhen
```

Profile z. B.:

```text
BusinessHours
OnDemand
Custom
```

---

# 9. MSP-Foundation-Aufgaben – kein Accelerator-Patch

Diese Punkte sollen bewusst **außerhalb** des Microsoft-Codes gelöst werden.

## 9.1 Existing-VNet Service Endpoints

Wenn:

```text
createAvdVnet = false
Private Endpoints = false
```

muss die Foundation auf den erforderlichen Subnetzen passend konfigurieren:

```text
Microsoft.Storage
Microsoft.KeyVault (wenn die gewählte KV-Netzwerkvariante dies benötigt)
```

## 9.2 Explicit Outbound

Foundation stellt NAT Gateway und Routing/Zuordnung bereit.

## 9.3 Preflight

Vor Deployment mindestens:

- Resource Provider,
- VM SKU in Region,
- Availability-Zone-Kompatibilität,
- vCPU-/Family-Quota,
- Marketplace Image,
- Netzwerk-/Subnetzrechte,
- AVD Enterprise Application IDs,
- erforderliche Endpunkte,
- Identity-Voraussetzungen.

## 9.4 Customer-/Profile-Schicht

Kosten-/Security-Defaults werden primär über Profile gesetzt und nicht als invasive Änderungen in Microsoft-Core-Module eingebaut.

---

# 10. Feature-abhängige Patches

## 10.1 Entra-only + FSLogix

Ziel ist **nicht** Storage Account Key + `cmdkey` als MSP-Standard.

Zu validieren/implementieren:

```text
Entra-joined AVD
+ Entra Kerberos
+ Azure Files identity-based access
+ saubere RBAC-/NTFS-Isolation
```

Expliziter Sicherheitstest:

```text
User A erstellt Profil A
User B erstellt Profil B
User A darf Profil B weder öffnen noch mounten
```

Upstream #869 vor Freigabe berücksichtigen.

## 10.2 Monitoring

Wenn `avdDeployMonitoring = true`:

- DCR auf Win11 24H2/25H2 testen,
- Terminal Services Session-Counter explizit validieren,
- Upstream #867 / PR #870 prüfen,
- tatsächliche Daten in Log Analytics verifizieren,
- Kosten/Retention für KMU festlegen.

## 10.3 Custom Images

Vor Aktivierung:

- PR #851/aktuellen Upstream berücksichtigen,
- VDOT/WDOT-Abhängigkeiten versionsfest machen,
- kein Download von beweglichem `main` ohne Release-Pinning.

## 10.4 Availability Zones / Resilient

Vor Aktivierung:

- PR #847 bzw. aktuellen Upstream-Fix übernehmen,
- Regionen ohne/mit unterschiedlich vielen Zonen testen,
- Re-Deployment bei geänderter Zonenwahl testen.

## 10.5 GPU

Vor Aktivierung:

- Upstream #872 korrigieren,
- Policy Assignment Managed Identity mit notwendiger, möglichst kleiner Rolle ausstatten.

---

# 11. Bewusst zurückgestellte P2-/Strategiethemen

Für Release 1.0 nicht erforderlich:

- GPU-Profile,
- komplexe Custom Image Factory,
- erzwungene Availability Zones,
- vollständiges HA-Profil,
- Azure Firewall,
- SQL Always On,
- zwei APP-Server,
- automatisches `AutomaticSessionHostConfiguration`-Preview als Default.

## Strategische Beobachtung: Host Pool Management

Microsoft arbeitet an `AutomaticSessionHostConfiguration`.

Release 1.0 bleibt zunächst beim klassischen/unmanaged Accelerator-Pfad. Unsere MSP-Orchestrierung darf aber nicht so eng an Registration Tokens und das heutige Bootstrap-Skript gekoppelt werden, dass ein späterer Wechsel unnötig teuer wird.

---

# 12. Release-/Source-of-Truth-Regeln

## 12.1 Bicep ist Source of Truth

- `.bicep` ist der produktive Source Code.
- vorcompilierte ARM-JSON-Dateien werden nicht manuell parallel gepflegt.
- falls Portal-/ARM-JSON weiterhin benötigt wird, wird es reproduzierbar im Build aus Bicep erzeugt.

## 12.2 Externe Module

- verwendete AVM-/Registry-Versionen werden fest gepinnt,
- Pipeline führt kontrolliertes `bicep restore` aus,
- keine unversionierten Modulpfade,
- bei kritischen Runtime-Abhängigkeiten optional vendoren.

## 12.3 Marketplace Images

Langfristig Approved Image Catalog statt unbegrenztem `latest`.

Release-Dokumentation muss festhalten, welcher OS-Build tatsächlich getestet wurde.

---

# 13. Security-Guardrails

- [ ] keine Secrets in Git/Azure Repos.
- [ ] Key Vault bzw. sichere Pipeline-Secret-Verarbeitung.
- [ ] Workload Identity Federation für Azure DevOps.
- [ ] keine Public IP auf AVD Session Hosts.
- [ ] kein offenes Inbound RDP als Standard.
- [ ] Trusted Launch / Secure Boot / vTPM beibehalten, sofern Ziel-SKU/Image kompatibel.
- [ ] Least-Privilege RBAC.
- [ ] Domain-Join-Konto nur in ADDS-Szenarien und nicht pauschal als Domain Admin verwenden.
- [ ] kein Passwort in Extension-/PowerShell-/DSC-Logs.
- [ ] keine persistenten Plaintext-MOFs mit Domain-Credentials.
- [ ] Management-VM Host-Firewall nach Storage-Konfiguration aktiv.
- [ ] temporäre lokale Admin-Berechtigungen werden entfernt.
- [ ] Backup/Restore für DC dokumentieren, wenn AD DS eingesetzt wird.
- [ ] Kundenparameter dürfen keine Passwörter enthalten.
- [ ] keine langlebigen GitHub-/Azure-DevOps-PATs auf Ziel-VMs.
- [ ] SAS-haltige Runtime-URLs nur in geschützten Settings verwenden.
- [ ] FSLogix Cross-User-Isolation explizit testen.
- [ ] Shared-Key-Zugriff bei identity-basiertem Azure Files langfristig deaktivieren, sofern technisch validiert.

---

# 14. Kosten-Guardrails

Standardmäßig vermeiden:

- Azure Firewall,
- DC ohne technische Notwendigkeit,
- 2 DCs ohne HA-Anforderung,
- 2 Session Hosts ohne Bedarf,
- dauerhaft laufende Management-VM,
- unnötige Premium-Disks,
- unnötige Private Endpoints,
- unnötige Defender-Pläne,
- DDoS-Plan ohne Bedarf,
- Hub-Spoke ohne Mehrwert,
- Premium Key Vault für reinen Secret-Store ohne HSM-Bedarf.

Standardmäßig bevorzugen:

- Entra ID Only ohne DC, wenn technisch möglich,
- 1 Session Host,
- bei erforderlichem ADDS: 1 DC im CostOptimized-Profil,
- Single VNet,
- NAT Gateway,
- Start VM on Connect,
- getestetes AVD Autoscale oder alternativ explizit gewählte Custom-AutoShutdown-Variante,
- kleine deallokierte Management-VM,
- Standard_LRS für Management-VM,
- parametrische VM-Größen.

---

# 15. Umsetzungsphasen

## Phase 0 – Repository/Governance

- [x] Accelerator importiert.
- [x] ursprünglichen P-NETRATOR-Stand archiviert.
- [x] Umsetzungsplan angelegt.
- [x] Deep-Dive Findings in Umsetzungsplan konsolidiert.
- [ ] Repository vor realen Kundenparametern privat betreiben.
- [ ] `msp/`, `runtime/`, `pipelines/` sauber strukturieren.
- [ ] Upstream-Remote-/Integrationsstrategie dokumentieren.
- [ ] MIT-Lizenzhinweise erhalten.

## Phase 1 – Baseline / Compiler / Preflight

- [ ] Bicep Build der importierten Baseline.
- [ ] relevante offene Upstream-PRs gegen importierten Stand vergleichen.
- [ ] Marketplace Images Germany West Central prüfen.
- [ ] SKU-/Quota-/Zone-Preflight implementieren.
- [ ] Resource Provider prüfen/registrieren.
- [ ] Minimal-Testparameter erstellen.
- [ ] What-If durchführen.
- [ ] bewusst entscheiden, ob ein vollständig unverändertes Microsoft-Deployment noch sinnvoll ist oder wegen bekannter P0-Bugs direkt eine „Known-Fixes“-Baseline verwendet wird.

## Phase 2 – MSP Foundation

- [ ] Resource Groups.
- [ ] VNet.
- [ ] `snet-avd`.
- [ ] `snet-infra`.
- [ ] optional `snet-private-endpoints`.
- [ ] optional `GatewaySubnet`.
- [ ] NSGs.
- [ ] NAT Gateway.
- [ ] Service Endpoints für Existing-VNet-Modell.
- [ ] Custom DNS Parameter/Outputs.
- [ ] Accelerator Outputs.
- [ ] idempotentes Re-Deployment.

## Phase 3 – Identity

### Entra-only

- [ ] Deployment ohne DC.
- [ ] Entra Join testen.
- [ ] optional Intune Enrollment.

### ADDS NewForest

- [ ] DC01 Modul.
- [ ] statische private IP.
- [ ] AD DS/DNS.
- [ ] VNet DNS.
- [ ] Health Check.
- [ ] Azure Backup.
- [ ] Monitoring.
- [ ] optional DC02 für Resilient vorbereiten.

### Existing Forest Migration

- [ ] temporäre S2S-Option.
- [ ] Azure DC hinzufügen.
- [ ] Replikation/DNS prüfen.
- [ ] FSMO/Demotion-Prozess.
- [ ] On-Prem-Server-Decommission als Abschlusskriterium.

## Phase 4 – P0 Accelerator Compatibility & Hardening

- [ ] F001 Management-VM-Größe entkoppeln.
- [ ] F002 Management-VM-Disk minimieren.
- [ ] F004 Server-Image migrieren.
- [ ] F005 Runtime-Artefakte entkoppeln.
- [ ] F006 deterministisches Resource Naming.
- [ ] F007 Common-Tag Drift entfernen.
- [ ] F008 Agent-/Bootloader-URL fixen.
- [ ] F009 Registration Token Lifecycle korrigieren.
- [ ] F010 Session-Host-Bootstrap idempotent machen.
- [ ] F011 Windows-Update-Verhalten parametrieren.
- [ ] F012–F016 Credential-/Management-VM-Hardening.
- [ ] F017/F018 Storage-DSC-Retry/Idempotenz.
- [ ] F022 Scaling Schedule parametrieren.
- [ ] Patch-Diff dokumentieren.
- [ ] Build/What-If/Testdeployment.

## Phase 5 – KMU-Profile

Mindestens:

```text
entra-small
adds-small
```

- [ ] Session Host Count Default = 1.
- [ ] VM-Größe parametrierbar.
- [ ] Start VM on Connect.
- [ ] `NativeAutoscale` vs. vorhandene Custom-AutoShutdown-Logik testen.
- [ ] Availability Zones Default aus.
- [ ] DDoS Default aus.
- [ ] AVD Private Link Default aus.
- [ ] Key Vault Standard als KMU-Default prüfen/setzen.
- [ ] Kundenparameter von Code trennen.

## Phase 6 – FSLogix / Azure Files

- [ ] FSLogix optionales Profilfeature definieren.
- [ ] Standard vs. Premium bewerten.
- [ ] LRS als CostOptimized Default prüfen.
- [ ] Service Endpoint vs. Private Endpoint festlegen.
- [ ] ADDS Authentication testen.
- [ ] Entra-only + Entra Kerberos Zielmodell testen.
- [ ] Storage-Key/`cmdkey` nicht als Entra-KMU-Default verwenden.
- [ ] F019 ACL-Fix/Isolation.
- [ ] RBAC/NTFS reproduzierbar setzen.
- [ ] Shared Key Deactivation testen.
- [ ] Management-VM Start/Wartung/Deallocate testen.
- [ ] Profilpersistenz nach Host-Rebuild.

## Phase 7 – APP/SQL

- [ ] APP-Modul.
- [ ] SQL-Modul.
- [ ] parametrische Größen/Disks.
- [ ] kein Public IP Default.
- [ ] Domain Join nur wenn benötigt.
- [ ] Backup-Anforderungen.

## Phase 8 – Dependency Automation

- [ ] Start: SQL → Health → APP.
- [ ] Stop: APP vollständig → SQL.
- [ ] korrekte Power States überspringen.
- [ ] Timeout-/Fehlerverhalten.
- [ ] Logging.
- [ ] kein Konflikt mit AVD Autoscale.

## Phase 9 – Azure DevOps

- [ ] Repository nach Azure Repos migrieren.
- [ ] Azure Resource Manager Service Connection mit WIF.
- [ ] Pipeline Stages erstellen.
- [ ] Runtime-Artefakte paketieren.
- [ ] Runtime Release Manifest erzeugen.
- [ ] Pipeline Artifact für interne Nachvollziehbarkeit.
- [ ] Runtime-Artefakte in privaten Azure Storage veröffentlichen.
- [ ] O007 Zugriffsmodell umsetzen.
- [ ] Approval Gate.
- [ ] PR Validation.
- [ ] Bicep Lint/Build/Validation/What-If.
- [ ] ARM JSON nur aus Bicep generieren, falls noch benötigt.

## Phase 10 – Monitoring / Post-Deployment Validation

- [ ] erwartete Ressourcen vorhanden.
- [ ] Identity-spezifische Ressourcen korrekt.
- [ ] Session Host registriert/Available.
- [ ] AVD Agent/Bootloader gesund.
- [ ] Join korrekt.
- [ ] Login.
- [ ] FSLogix.
- [ ] Cross-User-Profilzugriff verweigert.
- [ ] Start VM on Connect.
- [ ] gewähltes Power Management funktioniert.
- [ ] Management-VM deallocated.
- [ ] Management-VM Firewall aktiv.
- [ ] keine sensitiven DSC-/MOF-Artefakte übrig.
- [ ] Monitoring DCR liefert bei aktivierter Überwachung tatsächlich Daten.
- [ ] keine unerwarteten Enterprise-Kostenkomponenten.

---

# 16. Testmatrix

| ID | Szenario | Status |
|---|---|---|
| T01 | Entra ID, 1 Host, kein DC, ohne FSLogix | `[ ]` |
| T02 | Entra ID, 1 Host, kein klassischer DC, FSLogix/Entra Kerberos | `[ ]` |
| T03 | Neue AD-Gesamtstruktur, 1 DC, 1 Host, FSLogix | `[ ]` |
| T04 | Existing/Migrated AD DS, 1 Host, FSLogix | `[ ]` |
| T05 | AD DS, 2 Hosts | `[ ]` |
| T06 | AD DS, AVD + APP + SQL + Dependency Automation | `[ ]` |
| T07 | identisches Re-Deployment / What-If ohne Resource Drift | `[ ]` |
| T08 | Session Host ersetzen / Profil bleibt erhalten | `[ ]` |
| T09 | Management-VM Start → Storage-Wartung → Cleanup → Deallocate | `[ ]` |
| T10 | Management-VM nach Wartung: Firewall aktiv, Join-User kein Local Admin, keine Plaintext-MOFs | `[ ]` |
| T11 | Session-Host Re-Deployment ohne unnötige Agent-Neuinstallation/Reboot | `[ ]` |
| T12 | FSLogix: User A kann Profil von User B nicht öffnen/mounten | `[ ]` |
| T13 | Existing VNet + Service Endpoint + Azure Files/KV erreichbar | `[ ]` |
| T14 | NAT/Outbound nach Deaktivierung impliziter Standardpfade funktionsfähig | `[ ]` |
| T15 | Native Autoscale 1 Host + Start VM on Connect | `[ ]` |
| T16 | Custom AutoShutdown 1 Host als Vergleich/Fallback | `[ ]` |
| T17 | Monitoring Win11 24H2/25H2: DCR/Session Counter vorhanden | `[ ]` |

Für jedes relevante Szenario:

```text
Build
→ Validation
→ What-If
→ Deployment
→ Login/Funktion
→ Security-Test
→ Re-Deployment
→ Kostenprüfung
```

---

# 17. Preflight-Prüfungen

Automatisieren bzw. eindeutig prüfen:

- [ ] `Microsoft.DesktopVirtualization` registriert.
- [ ] `Microsoft.Compute` registriert.
- [ ] `Microsoft.Network` registriert.
- [ ] `Microsoft.Storage` registriert.
- [ ] weitere tatsächlich verwendete Provider registriert.
- [ ] VM SKU in Zielregion vorhanden.
- [ ] SKU in gewünschter Zone vorhanden, wenn AZ aktiviert.
- [ ] ausreichende vCPU-/Family-Quota.
- [ ] Marketplace Image vorhanden.
- [ ] Management-VM Small-Disk-Image vorhanden.
- [ ] AVD Enterprise Application Object IDs ermittelbar.
- [ ] Subnetz-/RBAC-Rechte vorhanden.
- [ ] Service Endpoints vorhanden, wenn Existing VNet + Public Endpoint/VNet Rules.
- [ ] NAT/Outbound vorhanden.
- [ ] erforderliche AVD-/Microsoft-/Runtime-Endpunkte erreichbar.
- [ ] Identity-Modell eindeutig gewählt.
- [ ] AD/DNS Health nur bei ADDS.
- [ ] Join-Berechtigungen nur bei ADDS.
- [ ] AVD-Lizenzierung dokumentiert.

---

# 18. Aufgabenübersicht / Priorisierung

Bestehende IDs bleiben erhalten; Deep-Dive-Aufgaben werden fortlaufend ergänzt.

| ID | Prio | Aufgabe | Status | Abhängigkeit |
|---|---:|---|---|---|
| A001 | P0 | Accelerator-Import | `[x]` | – |
| A002 | P0 | Umsetzungsplan als Single Source of Truth | `[x]` | A001 |
| A003 | P0 | Baseline build/validieren und Known-Upstream-Bugs berücksichtigen | `[ ]` | A001 |
| A004 | P0 | MSP Foundation / Existing-VNet-Modell | `[ ]` | A003 |
| A005 | P0 | Management-VM Größe entkoppeln | `[ ]` | A003 |
| A006 | P0 | Management-VM Disk minimieren | `[ ]` | A003 |
| A007 | P0 | Management-VM automatisch deallokieren | `[ ]` | A005/A006 |
| A008 | P0 | Script-/DSC-Versionen pinnen / private Runtime-Artefakte | `[ ]` | A003 |
| A009 | P0 | optionales New-Forest-DC-Modul | `[ ]` | A004 |
| A010 | P0 | Entra-only Profil ohne DC testen | `[ ]` | A004/A005 |
| A011 | P0 | ADDS/NewForest testen | `[ ]` | A009/A005 |
| A012 | P1 | FSLogix KMU-Default festlegen | `[ ]` | A010/A011 |
| A013 | P1 | APP-Modul | `[ ]` | A004 |
| A014 | P1 | SQL-Modul | `[ ]` | A004 |
| A015 | P1 | Dependency Automation | `[ ]` | A013/A014 |
| A016 | P1 | Azure DevOps Repository-Migration | `[ ]` | stabile Codebasis |
| A017 | P1 | Azure Pipelines / WIF / Approvals | `[ ]` | A016 |
| A018 | P1 | Post-Deployment-Tests | `[ ]` | A010/A011 |
| A019 | P1 | Kostenmodell Standardprofile | `[ ]` | A010/A011/A012 |
| A020 | P2 | Resilient-Profil | `[ ]` | Release-Baseline |
| A021 | P2 | Existing Hub/Hub-Spoke optional | `[ ]` | Release-Baseline |
| A022 | P0 | Windows-Server-2022-Legacy-Offer ersetzen | `[ ]` | A003 |
| A023 | P0 | `utcNow()` aus persistentem Resource-Naming entfernen | `[ ]` | A003 |
| A024 | P0 | `CreationTimeUTC`-/Common-Tag-Drift entfernen | `[ ]` | A023 |
| A025 | P0 | AVD Agent/Bootloader URL-Fix #873 übernehmen | `[ ]` | A003 |
| A026 | P0 | Host-Pool Registration Token Lifecycle entkoppeln | `[ ]` | A003 |
| A027 | P0 | Session-Host Agent Bootstrap idempotent machen | `[ ]` | A025/A026 |
| A028 | P0 | Windows-Update-/Session-Host-Patchmodell festlegen | `[ ]` | A027 |
| A029 | P0 | Management-VM Secret-/Logging-Hardening | `[ ]` | A003 |
| A030 | P0 | Plaintext-DSC-/MOF-Hardening und Cleanup | `[ ]` | A029 |
| A031 | P0 | Management-VM Firewall-/Local-Admin-Zustand sicher restaurieren | `[ ]` | A029 |
| A032 | P0 | Storage-DSC State Checks / Retry idempotent machen | `[ ]` | A030 |
| A033 | P0 | Existing-VNet Service Endpoints in Foundation | `[ ]` | A004 |
| A034 | P0 | Explicit Outbound/NAT in Foundation | `[ ]` | A004 |
| A035 | P0 | SKU-/Zone-/Quota-Preflight | `[ ]` | A003 |
| A036 | P1 | Scaling Plan vollständig parametrieren | `[ ]` | A003 |
| A037 | P1 | NativeAutoscale vs. Custom AutoShutdown im 1-Host-Test vergleichen | `[ ]` | A036/A010 |
| A038 | P0/P1 | Entra-only + FSLogix ohne Storage-Key-Default implementieren | `[ ]` | A012 |
| A039 | P0 | FSLogix ACL-Isolation / Upstream #869 korrigieren | `[ ]` | A038 |
| A040 | P1 | Monitoring DCR 24H2/25H2 validieren/fixen | `[ ]` | Monitoring aktiviert |
| A041 | P1 | PowerShell Runtime Module versionieren | `[ ]` | A008 |
| A042 | P1 | Preview-API-Audit | `[ ]` | A003 |
| A043 | P1 | Approved Marketplace Image Catalog/Manifest | `[ ]` | A003 |
| A044 | P1 | Key Vault Premium → KMU-Default Standard prüfen | `[ ]` | A003 |
| A045 | P1 | Storage Identity/RBAC Scope härten | `[ ]` | A012 |
| A046 | P1 | Shared Key Access nach Identity-Migration deaktivieren/testen | `[ ]` | A038 |
| A047 | P1 | FSLogix Defender-Variable/Exclusions korrigieren | `[ ]` | A012 |
| A048 | P1 | Bicep als einzige Source of Truth; ARM JSON nur Build-Artefakt | `[ ]` | A016/A017 |
| A049 | P1 | `br/public`-/AVM Supply-Chain-Pinning prüfen | `[ ]` | A003 |
| A050 | P1 | Security-Group-Array-Semantik prüfen/korrigieren | `[ ]` | A003 |
| A051 | P2 | GPU Policy Fix #872 | `[ ]` | GPU-Profil |
| A052 | P2 | Custom Image VDOT/WDOT Fix #851 | `[ ]` | Custom-Image-Profil |
| A053 | P2 | Availability-Zone Fix #847 | `[ ]` | Resilient-Profil |
| A054 | P2 | AutomaticSessionHostConfiguration #850 beobachten/PoC | `[ ]` | nach Release 1.0 |

---

# 19. Offene Architekturentscheidungen

## O001 – Entra Synchronisation

Entra Cloud Sync vs. Entra Connect Sync für ADDS NewForest/Migration.

## O002 – FSLogix Default

FSLogix standardmäßig aktiv oder bewusst optional?

## O003 – Azure Files Tier

```text
Standard LRS
vs.
Premium LRS
```

## O004 – Azure Files Netzwerkzugriff

```text
Service Endpoint
vs.
Private Endpoint
```

CostOptimized soll Kosten/Komplexität berücksichtigen.

## O005 – Monitoring Baseline

Welche Logs/Counters/Retention gehören zwingend in jedes KMU-Profil?

## O006 – Standard Session Host SKU

Keine harte Festlegung ohne Messwerte.

## O007 – Runtime-Artefakte bei privatem Repository

Optionen:

```text
A) Azure Storage + kurzlebige read-only User-Delegation-SAS
B) Azure Storage + Managed Identity
C) separates öffentliches Runtime-Repo
```

Aktueller V1-Favorit: **A)**.

## O008 – AVD Power Management Default

```text
NativeAutoscale
vs.
Custom AutoShutdown/Deallocate
```

Native Autoscale ist der bevorzugte Kandidat für neue Standarddeployments, aber die bestehende Custom-Lösung bleibt erhalten, bis der 1-Host-Vergleich abgeschlossen ist.

## O009 – Session Host Patch Lifecycle

```text
Patch in Place
vs.
Immutable/Replace
```

Muss entschieden werden, bevor `NoAutoUpdate=1` als Default zugelassen wird.

## O010 – Sicherer Storage-/Domain-Join-Helper

```text
gehärtetes DSC
vs.
neuer PowerShell-/Managed-Identity-basierter Bootstrap ohne Plaintext-MOF
```

Entscheidung vor produktiver ADDS/FSLogix-Freigabe.

---

# 20. Upstream-Update-Strategie

Nicht nur `main` betrachten.

Zukünftiger Ablauf:

```text
Microsoft main
+
relevante offene Issues
+
relevante offene PRs
        ↓
Review / Klassifizierung
        ↓
Integrations-Branch
        ↓
Patch-Konflikte / eigene Fixes prüfen
        ↓
Bicep Build
        ↓
What-If
        ↓
Testmatrix
        ↓
Freigabe
        ↓
MSP Main
```

Regeln:

- keine automatische ungeprüfte Synchronisation,
- vor Eigenentwicklung prüfen, ob Microsoft bereits einen Fix-PR hat,
- Upstream-PRs nicht blind cherry-picken; gegen unsere Zielprofile testen,
- jedes MSP-Release dokumentiert den zugrunde liegenden Microsoft-Commit und zusätzliche übernommene Fixes.

---

# 21. Entscheidungsprotokoll

## 2026-08-07 – Accelerator statt vollständiger Eigenentwicklung

Microsoft AVD Accelerator bleibt AVD-Core; MSP-spezifische Logik wird darum herum gebaut.

## 2026-08-07 – Keine Azure Firewall als KMU-Default

Kosten und Komplexität sind für die Zielgruppe nicht gerechtfertigt.

## 2026-08-07 – Entra ID Only bevorzugen, AD DS nur bei Bedarf

Kein genereller DC-Zwang. Branchensoftware/technische Anforderungen entscheiden.

## 2026-08-07 – On-Prem-Server nur Migrationsquelle

Zielumgebung ist cloudfähig ohne dauerhafte physische Serverabhängigkeit.

## 2026-08-07 – Netzwerk vor Accelerator

MSP Foundation erstellt VNet/Subnetze; Accelerator läuft im Existing-VNet-Modell.

## 2026-08-07 – APP/SQL separat

Eigene MSP-Module; kein Eingriff in AVD-Core.

## 2026-08-07 – Management-VM behalten, aber deallokieren

Wiederverwendbarer Helper ohne laufende Compute-Kosten. Diese Entscheidung setzt zwingend das neu dokumentierte Security-/Credential-Hardening voraus, weil die OS-Disk dauerhaft bestehen bleibt.

## 2026-08-07 – Management-VM entkoppeln und verkleinern

`Standard_B2als_v2` ist erster Testkandidat, keine endgültige Festlegung ohne Test.

## 2026-08-07 – Azure DevOps als Zielplattform

Azure Repos + Pipelines + WIF + Environments/Approvals.

## 2026-08-07 – Deep-Dive erweitert Minimal-Patch-Konzept

Die ursprüngliche kleine Patch-Liste war unvollständig. Insbesondere Runtime-Bootstrap, Storage-DSC, Secret-Verarbeitung, Idempotenz und aktuelle offene Upstream-Bugs benötigen vor produktivem Einsatz zusätzliche Compatibility-/Hardening-Maßnahmen.

**Entscheidung:** Microsoft AVD-Control-Plane-Logik bleibt Basis. Runtime-/Storage-Schicht wird gezielt gehärtet und modernisiert; keine Rückkehr zur vollständigen Eigenentwicklung.

---

# 22. Definition of Done für Release 1.0

Release 1.0 ist erst erreicht, wenn mindestens:

- [ ] Accelerator mit dokumentiertem Compatibility-/Hardening-Patchset integriert ist.
- [ ] Foundation reproduzierbar/idempotent ohne DC deploybar ist.
- [ ] Entra-only ohne DC funktioniert.
- [ ] optionale ADDS/NewForest-Variante funktioniert.
- [ ] Existing-VNet Service Endpoints und NAT korrekt bereitgestellt werden.
- [ ] Management-VM klein dimensioniert ist.
- [ ] Management-VM minimale sinnvolle Disk nutzt.
- [ ] Management-VM aktuelles Windows-Server-2022-Offer verwendet.
- [ ] Management-VM nach Lauf deallokiert wird.
- [ ] Management-VM nach Lauf Firewall aktiv hat.
- [ ] Join-Konto keine unnötige dauerhafte Local-Admin-Rolle besitzt.
- [ ] keine Plaintext-Domain-Credentials in Logs/MOF/Temp-Artefakten verbleiben.
- [ ] persistente Ressourcennamen deterministisch sind.
- [ ] normales Re-Deployment keinen unnötigen Common-Tag-Drift erzeugt.
- [ ] Runtime-Artefakte versionsfest und privat auslieferbar sind.
- [ ] kaputte Legacy-Agent-/Bootloader-URLs ersetzt sind.
- [ ] Session-Host-Bootstrap idempotent ist.
- [ ] Registration Token nicht bei jedem harmlosen Re-Deployment unnötig rotiert wird.
- [ ] Session-Host Patch-/Update-Modell definiert ist.
- [ ] Storage-Konfiguration retriesicher/idempotent ist.
- [ ] FSLogix-Entscheidung getroffen und mindestens ein produktionsfähiges Profil getestet ist.
- [ ] FSLogix Cross-User-Isolation getestet ist.
- [ ] Start VM on Connect funktioniert.
- [ ] Power-Management-Default nach Vergleich NativeAutoscale/Custom festgelegt ist.
- [ ] bei aktiviertem Monitoring DCR/Counter auf aktuellem Win11 Multi-Session funktionieren.
- [ ] Azure DevOps Pipeline mit WIF funktioniert.
- [ ] Approval Gate eingerichtet ist.
- [ ] Preflight/Post-Deployment Validation existieren.
- [ ] Kostenprofile für typisches Entra-only- und ADDS-KMU dokumentiert sind.
- [ ] README auf diesen Umsetzungsplan verweist.

---

# 23. Pflege dieses Dokuments

Bei jeder zukünftigen Arbeitssitzung zuerst:

1. `UMSETZUNGSPLAN.md` lesen.
2. aktuellen Aufgabenstatus prüfen.
3. neue Findings und Entscheidungen eintragen.
4. Findings klassifizieren: Accelerator-Patch / Foundation / Feature / P2.
5. erst danach Code ändern.
6. nach Abschluss Status, Tests und Entscheidungsprotokoll aktualisieren.
7. bei Upstream-Arbeit `main` **und** relevante offene Issues/PRs prüfen.

Damit bleibt der technische Kontext unabhängig von Chatverläufen dauerhaft im Repository erhalten.
