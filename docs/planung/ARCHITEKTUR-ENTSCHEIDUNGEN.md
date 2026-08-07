# MSP AVD Accelerator – Architekturentscheidungen

> **Rolle dieses Dokuments:** Fachliche Detailquelle für Architekturentscheidungen  
> **Projektstatus und Prioritäten:** ausschließlich in [`../../UMSETZUNGSPLAN.md`](../../UMSETZUNGSPLAN.md)  
> **Stand:** 2026-08-07

---

# 1. Entscheidungs-Gate

Vor Beginn der eigentlichen Implementierung müssen **alle offenen Architekturentscheidungen O001–O010 geschlossen** sein.

Erlaubt sind vor Abschluss des Gates ausschließlich:

- Recherche,
- Kosten-/Security-Bewertung,
- technische Verifikation,
- isolierte und verwerfbare PoCs, soweit sie für eine Entscheidung zwingend erforderlich sind,
- Dokumentation,
- Vorbereitung der Azure-DevOps-Migration.

Nicht begonnen werden sollen vor Abschluss des Gates:

- produktive MSP-Bicep-Module,
- dauerhafte Accelerator-Patches,
- Foundation-Implementierung,
- produktive Kundenprofile,
- Release-1.0-Code.

Der Hintergrund ist, dass Architekturentscheidungen die nachfolgende Implementierung bestimmen sollen und nicht nachträglich an bereits geschriebenen Code angepasst werden dürfen.

---

# 2. Bereits beschlossene Leitplanken

## D001 – Microsoft AVD Accelerator bleibt AVD-Core

Der Microsoft AVD Accelerator bleibt die Basis für:

- Host Pool,
- Workspace,
- Application Group,
- Session-Host-Grunddeployment,
- Scaling-/AVD-Kernintegration.

Keine vollständige Eigenimplementierung des AVD-Control-Planes.

Eigene MSP-Logik wird möglichst außerhalb von `workload/` und `avm/` implementiert.

---

## D002 – KMU-/Kostenprofil statt Enterprise-Default

Standardmäßig nicht erzwingen:

- Azure Firewall,
- DDoS Network Protection Standard,
- AVD Private Link,
- Availability Zones,
- zwei Session Hosts,
- Domain Controller ohne technische Notwendigkeit,
- zwei Domain Controller,
- Hub-Spoke,
- Premium-Komponenten ohne Mehrwert.

---

## D003 – Ein Session Host ist der KMU-Default

```text
Session Host Count = 1
```

Die erste Produktversion soll 1–3 Hosts unterstützen. Die VM-Größe bleibt parametrierbar.

---

## D004 – Entra ID Only ist ein vollständiges Zielmodell

Ein klassischer DC gehört **nicht automatisch** zur Zielarchitektur.

Wenn Anwendungen und Dienste ohne klassisches AD DS funktionieren:

```text
Entra ID
AVD
kein DC
```

AD DS wird nur als zusätzliche Dependency bereitgestellt, wenn z. B. Branchensoftware klassischen Domain Join, LDAP, Kerberos, NTLM oder vergleichbare AD-DS-Funktionen voraussetzt.

---

## D005 – Wenn AD DS benötigt wird, darf CostOptimized mit einem DC arbeiten

Kostenoptimiert:

```text
DC01
```

Optional resilient:

```text
DC01
DC02
```

Ein einzelner DC ist ein bewusst akzeptierter SPOF und benötigt mindestens statische private IP, Backup, Monitoring, Restore-Dokumentation und 24/7-Betrieb.

---

## D006 – On-Premises ist nach Migration keine Betriebsdependency

Der bestehende physische Server ist nur IST-Zustand/Migrationsquelle.

Nach erfolgreicher Migration:

- Azure ist eigenständig betriebsfähig,
- alter physischer Server wird decommissioned,
- NAS darf optional für Backup/Archiv verbleiben, ist aber keine Azure-Betriebsdependency.

---

## D007 – Single VNet und Foundation vor Accelerator

Standard:

```text
VNet Kunde
├── snet-avd
├── snet-infra
├── snet-private-endpoints optional
└── GatewaySubnet optional
```

Die MSP Foundation stellt Netzwerk/Outbound/Service Endpoints bereit.

Der Accelerator wird im Existing-VNet-Modell verwendet:

```bicep
createAvdVnet = false
```

---

## D008 – APP und SQL bleiben separate MSP-Workloads

APP-/SQL-Server werden nicht in den Microsoft AVD-Core eingebaut.

Sie erzwingen nicht automatisch AD DS.

Falls fachlich benötigt:

```text
Start: SQL → Health → APP
Stop:  APP vollständig → SQL
```

---

## D009 – Management-VM behalten und deallokieren

Die Accelerator-Management-/Helper-VM wird nach sicherem Abschluss nicht gelöscht, sondern deallokiert.

Diese Entscheidung gilt nur zusammen mit dem im Deep Dive identifizierten Credential-/Firewall-/Cleanup-Hardening.

---

## D010 – Management-VM wird entkoppelt und verkleinert

Die Größe darf nicht vom Session Host übernommen werden.

Erster Testkandidat:

```text
Standard_B2als_v2
```

Keine endgültige SKU-Festlegung ohne Region-/Workflow-Test.

---

## D011 – Azure DevOps ist die dauerhafte Entwicklungs-/Deploymentplattform

Ziel:

```text
Azure Repos
Azure Pipelines
Azure DevOps Environments / Approvals
Workload Identity Federation
```

GitHub ist nur noch temporäre Migrationsquelle. Nach validierter Migration soll das eigene GitHub-Repository aus Kostengründen deaktiviert/archiviert werden können.

---

# 3. Offene Architekturentscheidungen – Pflicht vor Implementierung

## O001 – Entra-Synchronisation bei ADDS

**Status:** OFFEN

Zu entscheiden:

```text
Entra Cloud Sync
vs.
Entra Connect Sync
```

Gilt nur für NewForest/MigrateExistingForest, wenn eine Synchronisierung nach Entra ID erforderlich ist.

Bewertungskriterien:

- Betrieb auf einem einzelnen Azure-DC vs. separatem Server,
- HA-/Recovery-Anforderungen,
- benötigte Features,
- Wartungsaufwand,
- Microsoft-Zielrichtung,
- Kosten,
- Automatisierbarkeit.

**Entscheidung muss dokumentieren:** Standardverfahren + Ausnahmen.

---

## O002 – FSLogix Default

**Status:** OFFEN

Zu entscheiden:

```text
FSLogix standardmäßig aktiv
vs.
FSLogix bewusst optional
```

Bewertungskriterien:

- 1-Host-KMU vs. 2+ Hosts,
- Rebuild-/Replace-Fähigkeit,
- Benutzerprofilpersistenz,
- zusätzliche Storage-/Identity-Komplexität,
- Kosten,
- OneDrive-/Known-Folder-Nutzung.

**Entscheidung muss dokumentieren:** Default je Profil (`entra-small`, `adds-small`) und Abweichungsregel.

---

## O003 – Azure Files Tier

**Status:** OFFEN

Zu entscheiden:

```text
Standard LRS
vs.
Premium LRS
```

Bewertungskriterien:

- reale IOPS-/Latency-Anforderungen der Zielgruppe,
- Profilanzahl,
- Kostendifferenz,
- Wachstum,
- Mindest-/Provisionierungsmodell,
- mögliche spätere Umstellung.

**Entscheidung muss dokumentieren:** CostOptimized Default + Schwelle für Premium.

---

## O004 – Azure Files Netzwerkzugriff

**Status:** OFFEN

Zu entscheiden:

```text
Service Endpoint + VNet Rules
vs.
Private Endpoint
```

Bewertungskriterien:

- monatliche Fixkosten,
- DNS-Komplexität,
- Security-Anforderung,
- Public Endpoint mit Netzwerkrestriktion akzeptabel oder nicht,
- Customer Compliance,
- Betriebs-/Troubleshooting-Aufwand.

**Entscheidung muss dokumentieren:** CostOptimized Default + Kriterien für Private Endpoint.

---

## O005 – Monitoring Baseline

**Status:** OFFEN

Zu entscheiden:

Welche Monitoring-Komponenten sind verpflichtend und welche optional?

Mindestens bewerten:

- Log Analytics Workspace,
- AMA/DCR,
- AVD Insights,
- Session-/Capacity-Counter,
- VM Heartbeat,
- Alerts,
- Retention,
- Kostenbegrenzung.

Bekannter Accelerator-Befund F021 muss berücksichtigt werden.

**Entscheidung muss dokumentieren:** verpflichtende Datenquellen, Retention und Kosten-Guardrail.

---

## O006 – Standard Session Host SKU

**Status:** OFFEN

Keine feste SKU aus Gewohnheit übernehmen.

Kandidaten sollen anhand der Zielgruppe und Messwerte bewertet werden.

Entscheidungsprinzip kann auch lauten:

```text
kein universeller fester SKU-Default,
sondern Größenprofil + empfohlener Startwert
```

Bewertungskriterien:

- 1–30 Mitarbeiter,
- typische gleichzeitige Sessions,
- Office/Sage/LOB-Workload,
- CPU/RAM,
- Burstable vs. D-/E-Serie,
- Region/Quota,
- Kosten.

---

## O007 – Runtime-Artefakte bei privatem Repository

**Status:** OFFEN

Optionen:

```text
A) Azure Storage + kurzlebige read-only User-Delegation-SAS
B) Azure Storage + Managed Identity
C) separates öffentliches Runtime-Repository
```

Aktueller V1-Favorit aus der Analyse:

```text
Private Azure Repos
→ Pipeline
→ versionierter Runtime-Release
→ privater Azure Blob Container
→ kurzlebige read-only User-Delegation-SAS
→ Ziel-VM
```

Managed Identity ist langfristig attraktiv, verlangt aber beim aktuellen Accelerator weitere Änderungen für verschachtelte PowerShell-/DSC-Downloads.

**Entscheidung muss vor Azure-DevOps-Pipeline-Design geschlossen sein.**

---

## O008 – AVD Power Management Default

**Status:** OFFEN

Zu entscheiden:

```text
NativeAutoscale + Start VM on Connect
vs.
Custom AutoShutdown/Deallocate
```

Dabei gilt unabhängig von der Entscheidung:

- nie AVD Autoscale und eigene Automation parallel für denselben Session Host,
- APP-/SQL-Dependency-Automation bleibt ein separates Thema.

Bewertungskriterien:

- 1-Host-Verhalten,
- Disconnected Sessions,
- RDS-GPOs,
- Wartungsmodus,
- Abend-/Wochenendnutzung,
- Reproduzierbarkeit,
- Wartungsaufwand,
- vorhandene funktionierende Custom-AutoShutdown-Logik.

Ein isolierter 1-Host-PoC ist zulässig, wenn für die Entscheidung erforderlich.

---

## O009 – Session Host Patch Lifecycle

**Status:** OFFEN

Zu entscheiden:

```text
Patch in Place
vs.
Immutable/Replace Session Hosts
```

Hintergrund: Der Accelerator setzt aktuell `NoAutoUpdate = 1` auf dem produktiven Session Host.

Bewertungskriterien:

- Patch-Verantwortung,
- Wartungsfenster,
- Golden Image / Marketplace Image,
- Session-Host-Replacement,
- FSLogix-Profilpersistenz,
- Rollback,
- Betrieb bei nur einem Host.

**Kein `NoAutoUpdate=1` als MSP-Default, solange diese Entscheidung offen ist.**

---

## O010 – Sicherer Storage-/Domain-Join-Helper

**Status:** OFFEN

Zu entscheiden:

```text
A) vorhandenes DSC konsequent härten
vs.
B) neuen PowerShell-/Identity-basierten Bootstrap ohne Plaintext-MOF aufbauen
```

Auslöser sind die Deep-Dive-Funde F012–F018.

Bewertungskriterien:

- Secret-Verarbeitung,
- Idempotenz,
- Retry,
- Patch-Delta zum Upstream,
- Wartbarkeit,
- Wiederverwendung der deallokierten Management-VM,
- Migration auf Managed Identity.

**Entscheidung muss vor produktivem ADDS/FSLogix-Code geschlossen sein.**

---

# 4. Abschlusskriterium des Architektur-Gates

Das Gate ist geschlossen, wenn:

- [ ] O001 entschieden und dokumentiert.
- [ ] O002 entschieden und dokumentiert.
- [ ] O003 entschieden und dokumentiert.
- [ ] O004 entschieden und dokumentiert.
- [ ] O005 entschieden und dokumentiert.
- [ ] O006 entschieden und dokumentiert.
- [ ] O007 entschieden und dokumentiert.
- [ ] O008 entschieden und dokumentiert.
- [ ] O009 entschieden und dokumentiert.
- [ ] O010 entschieden und dokumentiert.
- [ ] daraus resultierende Änderungen an Projektphasen/Aufgaben in `UMSETZUNGSPLAN.md` übernommen.

Erst danach beginnt die eigentliche Implementierung. Die erste Implementierungsphase ist die Migration/Einrichtung in Azure DevOps.
