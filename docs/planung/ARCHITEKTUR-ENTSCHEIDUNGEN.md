# MSP AVD Accelerator – Architekturentscheidungen

> **Rolle dieses Dokuments:** Fachliche Detailquelle für Architekturentscheidungen  
> **Projektstatus und Prioritäten:** ausschließlich in [`../../UMSETZUNGSPLAN.md`](../../UMSETZUNGSPLAN.md)  
> **Stand:** 2026-08-07

---

# 1. Entscheidungs-Gate

Vor Beginn der eigentlichen Implementierung müssen **alle Architekturentscheidungen O001–O010 geschlossen** sein.

Vor Abschluss des Gates erlaubt:

- Recherche und technische Verifikation,
- Kosten-/Security-Bewertung,
- isolierte und verwerfbare PoCs,
- Dokumentation,
- Vorbereitung der Azure-DevOps-Migration.

Vor Abschluss des Gates nicht beginnen:

- produktive MSP-Bicep-Module,
- dauerhafte Accelerator-Patches,
- Foundation-Implementierung,
- produktive Kundenprofile,
- Release-1.0-Code.

---

# 2. Bereits beschlossene Leitplanken

## D001 – Microsoft AVD Accelerator bleibt AVD-Core

Microsoft AVD Accelerator bleibt die Basis für Host Pool, Workspace, Application Group, Session-Host-Grunddeployment und AVD-Kernintegration. Eigene MSP-Logik wird möglichst außerhalb von `workload/` und `avm/` implementiert.

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
- Premium-Komponenten ohne fachlichen Mehrwert.

## D003 – Ein Session Host ist der KMU-Default

```text
Session Host Count = 1
```

Release 1.0 soll 1–3 Hosts unterstützen. VM-Größe bleibt parametrierbar.

## D004 – Entra ID Only ist ein vollständiges Zielmodell

Wenn Anwendungen und Dienste ohne klassisches AD DS funktionieren:

```text
Entra ID
AVD
kein DC
```

AD DS wird nur bei tatsächlicher technischer/fachlicher Notwendigkeit bereitgestellt.

## D005 – CostOptimized AD DS darf mit einem DC arbeiten

```text
CostOptimized: DC01
Resilient:     DC01 + DC02
```

Ein einzelner DC ist ein bewusst akzeptierter SPOF und benötigt mindestens statische private IP, Backup, Monitoring und Restore-Dokumentation.

## D006 – On-Premises ist nach Migration keine Betriebsdependency

Der bestehende physische Server ist nur IST-Zustand/Migrationsquelle. Nach erfolgreicher Migration ist Azure eigenständig betriebsfähig und der alte Server wird decommissioned. NAS darf optional für Backup/Archiv verbleiben.

## D007 – Single VNet und Foundation vor Accelerator

```text
VNet Kunde
├── snet-avd
├── snet-infra
├── snet-private-endpoints optional
└── GatewaySubnet optional
```

Die MSP Foundation stellt Netzwerk, Outbound und Service Endpoints bereit. Accelerator verwendet standardmäßig:

```bicep
createAvdVnet = false
```

## D008 – APP und SQL bleiben separate MSP-Workloads

APP-/SQL-Server werden nicht in den Microsoft AVD-Core eingebaut und erzwingen nicht automatisch AD DS.

```text
Start: SQL → Health → APP
Stop:  APP vollständig → SQL
```

## D009 – Management-VM behalten und deallokieren

Die Management-/Helper-VM wird nach sicherem Abschluss nicht gelöscht, sondern deallokiert. Diese Entscheidung gilt nur zusammen mit Credential-/Firewall-/Cleanup-Hardening.

## D010 – Management-VM entkoppeln und verkleinern

Die Größe darf nicht vom Session Host übernommen werden. Erster Testkandidat:

```text
Standard_B2als_v2
```

## D011 – Azure DevOps ist die dauerhafte Entwicklungs-/Deploymentplattform

```text
Azure Repos
Azure Pipelines
Azure DevOps Environments / Approvals
Workload Identity Federation
```

GitHub ist nur temporäre Migrationsquelle für unser eigenes Repository.

---

# 3. Architekturentscheidungen O001–O010

## O001 – Synchronisationsverfahren bei klassischem AD DS

**Status:** `BESCHLOSSEN`  
**Entschieden:** 2026-08-07

### Geltungsbereich

O001 gilt ausschließlich für **klassisches Active Directory Domain Services**, insbesondere:

```text
NewForest
MigrateExistingForest
ExistingAzureForest
```

wenn Benutzer-/Gruppenidentitäten zwischen klassischem AD DS und Microsoft Entra ID synchronisiert werden sollen.

### Entscheidung

**Microsoft Entra Cloud Sync ist der MSP-Standard.**

```text
Klassisches AD DS
        ↓
Benötigt der Kunde eine Funktion,
die Cloud Sync nicht unterstützt?
        │
   Nein │                   Ja
        ▼                    ▼
 Entra Cloud Sync      Entra Connect Sync
 = Standard            = Feature-Fallback
```

Microsoft Entra Connect Sync bleibt unterstützt, wird aber nur eingesetzt, wenn eine **konkrete technische Kundenanforderung** Cloud Sync ausschließt.

Die Verwendung von Connect Sync muss im Kunden-Onboarding mit dem ausschlaggebenden Feature dokumentiert werden.

### Typische Gründe für Connect Sync

Vor Einsatz prüfen, ob mindestens eine tatsächlich benötigte Funktion vorliegt, beispielsweise:

- Hybrid Entra Join / Device-Synchronisation,
- komplexe/custom Synchronisierungsregeln oder Filter,
- Topologien oder Objektmengen außerhalb der Cloud-Sync-Grenzen,
- andere Connect-spezifische Funktionen, die Cloud Sync zum jeweiligen Projektzeitpunkt nicht unterstützt.

Die Featurematrix wird beim Kunden-Onboarding gegen den **aktuellen Microsoft-Stand** geprüft; die Liste ist keine dauerhaft abschließende Produktmatrix.

### Betriebsmodell bei klassischem AD DS

Weder Cloud Sync noch Connect Sync erzwingen in unserem KMU-Modell automatisch eine zusätzliche VM.

Bei klassischem DC kann die gewählte Sync-Komponente, sofern vom Produkt unterstützt und betrieblich sinnvoll, auf vorhandener Infrastruktur betrieben werden:

```text
DC01
├── AD DS
├── DNS
└── Entra Sync-Komponente
```

Eine zusätzliche Server-VM wird nur aufgrund konkreter technischer, Verfügbarkeits- oder Betriebsanforderungen vorgesehen.

### Nicht Teil von O001 – Microsoft Entra Domain Services

Bei `EntraDS`/Microsoft Entra Domain Services wird **kein eigener Cloud-Sync- oder Connect-Sync-Agent gegen die Managed Domain betrieben**.

```text
Microsoft Entra ID
        ↓
Microsoft-managed Synchronisierung
        ↓
Microsoft Entra Domain Services
```

Eine APP01 oder andere Member-Server-VM wird nicht allein für Entra Connect Sync gegen die Managed Domain benötigt.

Wenn während einer Migration parallel klassisches AD DS existiert, kann O001 weiterhin für die Synchronisierung **dieses klassischen AD DS nach Entra ID** relevant sein.

### Onboarding-Regel

Für jeden klassischen ADDS-Kunden:

1. Cloud Sync als Ausgangspunkt wählen.
2. Anforderungen gegen aktuelle Cloud-Sync-Funktionsmatrix prüfen.
3. Nur bei konkreter Funktionslücke auf Connect Sync wechseln.
4. Grund der Ausnahme im Kundenprofil dokumentieren.

---

## O002 – FSLogix Default

**Status:** `BESCHLOSSEN`  
**Entschieden:** 2026-08-07

### Entscheidung

**FSLogix Profile Container ist die Standard-Profillösung für produktive pooled AVD-Deployments – auch bei nur einem Session Host.**

Ziel ist die Entkopplung des persistenten Benutzerzustands vom Lebenszyklus des Session Hosts:

```text
Session Host = austauschbarer Compute
Benutzerprofil = persistenter Zustand außerhalb der VM
```

Damit bleiben Rebuild, Replacement und spätere Skalierung möglich, ohne lokale Benutzerprofile an eine konkrete Session-Host-VM zu binden.

### Standardkonfiguration

```text
FSLogix Profile Container       = ON
Azure Files                     = Ziel-Storage
Single Profile Container        = ON
separater ODFC Container        = OFF als Default
FSLogix Cloud Cache             = OFF als Default
```

Ein normaler FSLogix Profile Container ist damit unser Standard. Zusätzliche ODFC-Container oder Cloud Cache werden nur bei konkreter fachlicher/technischer Notwendigkeit aktiviert.

### Profilregeln

#### `adds-small`

```text
FSLogix = ON
```

FSLogix ist Standard, auch bei einem Session Host.

#### `entra-small`

FSLogix ist ebenfalls der **technische Zieldefault**, darf aber nur aktiviert werden, wenn in der gewählten Azure-Region ein **unterstützter identity-basierter Azure-Files-Pfad** für das konkrete Entra-only-/cloud-only-Szenario verfügbar ist und dessen Kosten akzeptiert wurden.

Der im aktuellen Accelerator vorhandene Pfad:

```text
Storage Account Key
+ cmdkey
```

wird **nicht als MSP-Default** verwendet.

Wenn identity-basierter Azure-Files-Zugriff für das konkrete Profil/Region/Supportmodell nicht sinnvoll oder nicht unterstützt ist, wird nicht automatisch auf Storage Account Key zurückgefallen. Stattdessen wird entweder:

- FSLogix für dieses Sonderprofil deaktiviert,
- die Architektur bewusst geändert,
- oder das Kundenprofil nicht in dieser Form freigegeben.

### Zulässige Ausnahme – Cost-Minimum / Persistent Single Host

FSLogix bleibt parametrierbar und darf für einen bewusst gewählten Sonderfall deaktiviert werden, wenn beispielsweise:

- genau ein persistenter Session Host betrieben wird,
- kein Host-Replacement-/Immutable-Modell vorgesehen ist,
- Profilbindung an die VM bewusst akzeptiert wird,
- keine fachliche Anforderung dagegen spricht,
- die Folgen für OneDrive/VDI, Rebuild und spätere Skalierung dokumentiert und akzeptiert sind,
- Kostenminimierung ausdrücklich Vorrang hat.

Das ist **kein Standardprofil**, sondern eine dokumentierte Ausnahme.

### Onboarding-Regel

Bei jedem Kunden wird festgehalten:

```text
FSLogix = Standard / Ausnahme deaktiviert
Storage-/Identity-Modell
Region
Support-/Verfügbarkeitsprüfung
Kostenfolge
```

Bei deaktiviertem FSLogix muss das Onboarding ausdrücklich dokumentieren:

- Profile liegen lokal auf dem Session Host,
- Host-Rebuild erhält die Benutzerprofile nicht automatisch,
- spätere Skalierung kann eine Profilmigration erfordern,
- non-persistent VDI-/OneDrive-Anforderungen müssen separat bewertet werden.

### Abhängigkeit zu O003

O002 entscheidet **dass** FSLogix der Standard ist, aber noch nicht **welches Azure-Files-Tier** dafür verwendet wird.

Die Analyse zu O002 hat gezeigt, dass O003 **nicht global mit nur einem Tier für alle Profile entschieden werden darf**.

Insbesondere ist bei O003 zwingend profilspezifisch zu prüfen:

```text
adds-small
→ Standard/HDD dürfte für die typische kleine AVD-Last wirtschaftlich und technisch ausreichend sein
→ muss in O003 noch final validiert werden

entra-small + cloud-only Entra Kerberos
→ regionale Feature-Unterstützung kann bestimmte Azure-Files-Tiers erzwingen
→ Germany West Central ist nach aktuellem Prüfstand besonders zu beachten
```

**Verbindliche Bearbeitungsregel für O003:**

> Nicht „Standard vs. Premium“ abstrakt entscheiden, sondern pro Identity-/Kundenprofil die technisch unterstützten Tiers ermitteln und daraus den kostengünstigsten freigegebenen Default ableiten.

---

## O003 – Azure Files Tier

**Status:** `OFFEN`

### Bereits aus O002 übernommene Prüfvorgabe

O003 wird **profilspezifisch** entschieden und nicht als globaler Azure-Files-Default.

Mindestens getrennt bewerten:

```text
adds-small
entra-small
EntraDS-Profil, falls produktisiert
```

Dabei müssen jeweils geprüft werden:

- technisch unterstützte Identity-/Authentication-Kombination,
- regionale Unterstützung, insbesondere Germany West Central,
- Standard/HDD vs. SSD/Premium bzw. aktuelles Azure-Files-Bereitstellungsmodell,
- IOPS und Latenz,
- Profilanzahl und typische Concurrency,
- Mindestbereitstellung / Provisionierungsmodell,
- tatsächliche monatliche Kosten,
- spätere Skalierung oder Tierwechsel.

**Arbeitsannahme aus O002, noch keine Entscheidung:**

- `adds-small`: Standard/HDD ist voraussichtlich der CostOptimized-Kandidat.
- `entra-small`: cloud-only Entra-Kerberos-Unterstützung kann regional ein SSD/Premium-Tier erforderlich machen; für Germany West Central ist dies vor der Entscheidung anhand des aktuellen Microsoft-Supportstands zu verifizieren.

**Entscheidung muss dokumentieren:** kostengünstigster unterstützter Default pro freigegebenem Profil + klare Schwelle/Ausnahme für höheres Tier.

---

## O004 – Azure Files Netzwerkzugriff

**Status:** `OFFEN`

```text
Service Endpoint + VNet Rules
vs.
Private Endpoint
```

Bewertung: Fixkosten, DNS-Komplexität, Security/Compliance, Public Endpoint mit Netzwerkrestriktion, Troubleshooting-Aufwand.

**Entscheidung muss dokumentieren:** CostOptimized Default + Kriterien für Private Endpoint.

---

## O005 – Monitoring Baseline

**Status:** `OFFEN`

Zu bewerten:

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

## O006 – Standard Session Host SKU / Größenstrategie

**Status:** `OFFEN`

Mögliche Entscheidung:

```text
kein universeller fixer SKU-Default,
sondern Größenprofil + empfohlener Startwert
```

Bewertung: Benutzer/Concurrency, Office/Sage/LOB, CPU/RAM, Burstable vs. D-/E-Serie, Region/Quota, Kosten.

---

## O007 – Runtime-Artefakte bei privatem Repository

**Status:** `OFFEN`

Optionen:

```text
A) Azure Storage + kurzlebige read-only User-Delegation-SAS
B) Azure Storage + Managed Identity
C) separates öffentliches Runtime-Repository
```

Aktueller V1-Favorit aus der Analyse: A. Entscheidung muss vor finalem Azure-DevOps-Pipeline-Design geschlossen sein.

---

## O008 – AVD Power Management Default

**Status:** `OFFEN`

```text
NativeAutoscale + Start VM on Connect
vs.
Custom AutoShutdown/Deallocate
```

Unabhängig von der Entscheidung: nie beide Lösungen parallel für denselben Session Host. APP-/SQL-Dependency-Automation bleibt separat.

---

## O009 – Session Host Patch Lifecycle

**Status:** `OFFEN`

```text
Patch in Place
vs.
Immutable/Replace Session Hosts
```

Hintergrund: Accelerator setzt derzeit `NoAutoUpdate = 1`. Kein solcher MSP-Default, solange O009 offen ist.

---

## O010 – Sicherer Storage-/Domain-Join-Helper

**Status:** `OFFEN`

```text
A) vorhandenes DSC konsequent härten
vs.
B) neuer PowerShell-/Identity-basierter Bootstrap ohne Plaintext-MOF
```

Auslöser: Deep-Dive-Findings F012–F018.

---

# 4. Abschlusskriterium des Architektur-Gates

- [x] O001 entschieden und dokumentiert.
- [x] O002 entschieden und dokumentiert.
- [ ] O003 entschieden und dokumentiert.
- [ ] O004 entschieden und dokumentiert.
- [ ] O005 entschieden und dokumentiert.
- [ ] O006 entschieden und dokumentiert.
- [ ] O007 entschieden und dokumentiert.
- [ ] O008 entschieden und dokumentiert.
- [ ] O009 entschieden und dokumentiert.
- [ ] O010 entschieden und dokumentiert.
- [ ] resultierende Änderungen an Projektphasen/Aufgaben in `UMSETZUNGSPLAN.md` übernommen.

Erst nach Abschluss aller Punkte beginnt die eigentliche Implementierung. Erste Implementierungsphase bleibt Azure DevOps.
