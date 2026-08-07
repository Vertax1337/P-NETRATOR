# MSP AVD Accelerator – Umsetzungsplan

> **Rolle:** Master-/Steuerungsdokument und Single Source of Truth für **Status, Gates, Reihenfolge und Prioritäten**  
> **Stand:** 2026-08-07  
> **Repository:** `Vertax1337/P-NETRATOR`  
> **Microsoft-Upstream:** `Azure/avdaccelerator`  
> **Importierter Upstream-Stand:** `63c55c857f77783c01812c935957cd8c9c67a5a7`  
> **Ursprünglicher P-NETRATOR-Stand:** `archive/p-netrator-original`

---

# 1. Arbeitsmodell / Source of Truth

Dieses Dokument bleibt die zentrale Projektsteuerung.

**Nur dieses Dokument ist verbindlich für:**

- aktuellen Projektstatus,
- Reihenfolge der Phasen,
- Gate-Status,
- Priorität,
- Aufgabenstatus,
- nächsten Arbeitsschritt.

Detaildokumente enthalten technische Begründungen, Findings, Entscheidungsoptionen und Testanforderungen, aber keinen konkurrierenden Projektstatus.

## Dokumentenstruktur

| Dokument | Zweck |
|---|---|
| [`UMSETZUNGSPLAN.md`](UMSETZUNGSPLAN.md) | Master: Gates, Phasen, Aufgaben, Prioritäten, Status |
| [`docs/planung/ARCHITEKTUR-ENTSCHEIDUNGEN.md`](docs/planung/ARCHITEKTUR-ENTSCHEIDUNGEN.md) | beschlossene Leitplanken + O001–O010 |
| [`docs/planung/AZURE-DEVOPS.md`](docs/planung/AZURE-DEVOPS.md) | Azure-DevOps-Migration, WIF, Pipelines, Runtime-Artefakte, GitHub-Stilllegung |
| [`docs/planung/ACCELERATOR-DEEP-DIVE.md`](docs/planung/ACCELERATOR-DEEP-DIVE.md) | Findings F001–F039 inkl. Auswirkungen und Maßnahmen |
| [`docs/planung/VALIDIERUNG-UND-RELEASE.md`](docs/planung/VALIDIERUNG-UND-RELEASE.md) | Preflight, Testmatrix, Security-/Kosten-Guardrails, Release-DoD |
| [`docs/planung/KUNDEN-ONBOARDING.md`](docs/planung/KUNDEN-ONBOARDING.md) | operativer Standardprozess vom Assessment bis Deployment und Betriebsübergabe |

## Pflege-Regel

Bei jeder Arbeitssitzung:

1. Master lesen.
2. aktuelles Gate und `NÄCHSTER SCHRITT` prüfen.
3. nur relevante Detaildokumente lesen.
4. technische Erkenntnisse dort dokumentieren.
5. Status/Priorität ausschließlich hier aktualisieren.
6. erst danach Code ändern.

---

# 2. Projektziel

Microsoft AVD Accelerator bleibt die AVD Workload Deployment Engine. Darum entsteht eine dünne MSP-Schicht für:

- kostenoptimierte KMU-Defaults,
- Single-VNet-Foundation,
- Entra-only oder optional klassisches AD DS / Entra Domain Services,
- optionale APP-/SQL-Workloads,
- Azure DevOps CI/CD,
- private/versionierte Runtime-Artefakte,
- reproduzierbare/idempotente Deployments,
- Preflight/Post-Deployment-Validierung,
- kontrolliertes Compatibility-/Hardening-Patchset.

Zielgruppe:

- typischerweise `<30 Mitarbeiter`,
- meistens 1 Session Host,
- zunächst 1–3 Hosts unterstützt,
- hohe Kostenpriorität,
- keine unnötige Enterprise-Komplexität.

---

# 3. Aktueller Projektstatus

## Aktives Gate: G0 – Architekturentscheidungen

**Status: OFFEN / BLOCKIERT IMPLEMENTIERUNG**  
**Fortschritt:** `1/10` Architekturentscheidungen geschlossen.

Alle O001–O010 müssen vor der eigentlichen Implementierung geschlossen sein.

Details:

[`docs/planung/ARCHITEKTUR-ENTSCHEIDUNGEN.md`](docs/planung/ARCHITEKTUR-ENTSCHEIDUNGEN.md)

| Entscheidung | Thema | Status |
|---|---|---|
| O001 | Synchronisationsverfahren bei klassischem AD DS | `BESCHLOSSEN` – Cloud Sync Standard, Connect Sync Feature-Fallback |
| O002 | FSLogix Default | `OFFEN` |
| O003 | Azure Files Tier | `OFFEN` |
| O004 | Service Endpoint vs. Private Endpoint | `OFFEN` |
| O005 | Monitoring Baseline | `OFFEN` |
| O006 | Standard Session Host SKU / Größenstrategie | `OFFEN` |
| O007 | Runtime-Artefakte bei privatem Repository | `OFFEN` |
| O008 | NativeAutoscale vs. Custom AutoShutdown | `OFFEN` |
| O009 | Patch in Place vs. Immutable/Replace | `OFFEN` |
| O010 | gehärtetes DSC vs. neuer Storage-/Domain-Join-Helper | `OFFEN` |

### O001 Kurzentscheidung

Für klassisches AD DS gilt:

```text
Cloud Sync = MSP-Standard
Connect Sync = Ausnahme, wenn konkrete Cloud-Sync-Funktionslücke vorliegt
```

Microsoft Entra Domain Services ist von O001 ausgenommen; gegen die Managed Domain wird kein eigener Connect-/Cloud-Sync-Agent betrieben.

### NÄCHSTER SCHRITT

> **O002 – FSLogix Default entscheiden.**

Bis G0 geschlossen ist, werden keine dauerhaften Foundation-/Accelerator-/Profil-Implementierungen begonnen. Recherche und isolierte PoCs zur Entscheidungsfindung sind zulässig.

---

# 4. Verbindliche Gate-/Phasenreihenfolge

```text
G0  ARCHITEKTUR-ENTSCHEIDUNGEN
    O001–O010 vollständig schließen
            ↓
G1  AZURE DEVOPS MIGRATION / BOOTSTRAP
    Azure Repos + Pipelines + WIF + Policies
    GitHub anschließend stilllegbar
            ↓
G2  BASELINE / PREFLIGHT
    Accelerator Build + Known Upstream Fixes + Preflight
            ↓
G3  MSP FOUNDATION
    VNet + Subnets + NAT + Service Endpoints
            ↓
G4  IDENTITY
    Entra-only und optionale ADDS-/EntraDS-Pfade
            ↓
G5  ACCELERATOR COMPATIBILITY & HARDENING
    P0-Patchset / Idempotenz / Security
            ↓
G6  KMU-PROFILE / FSLOGIX
            ↓
G7  APP / SQL / DEPENDENCY AUTOMATION
            ↓
G8  VALIDIERUNG / RELEASE 1.0
            ↓
G9  OPTIONALE P2-FEATURES
```

---

# 5. G0 – Architekturentscheidungen

**Priorität: P0 / zuerst**

- [ ] A055 – O001–O010 vollständig schließen. `1/10 erledigt`
- [x] O001 – Cloud Sync als Standard für klassisches AD DS; Connect Sync als begründeter Feature-Fallback.
- [ ] O002 – FSLogix Default.
- [ ] O003 – Azure Files Tier.
- [ ] O004 – Azure Files Netzwerkzugriff.
- [ ] O005 – Monitoring Baseline.
- [ ] O006 – Session Host Größenstrategie.
- [ ] O007 – Runtime-Artefakt-Zugriff.
- [ ] O008 – AVD Power Management.
- [ ] O009 – Session Host Patch Lifecycle.
- [ ] O010 – sicherer Storage-/Domain-Join-Helper.

**Exit-Kriterium:** O001–O010 stehen nicht mehr auf `OFFEN`.

---

# 6. G1 – Azure DevOps Migration / Bootstrap

**Priorität: P0 / erste Implementierungsphase**

Details: [`docs/planung/AZURE-DEVOPS.md`](docs/planung/AZURE-DEVOPS.md)

- [ ] A016 – Repository vollständig nach Azure Repos migrieren.
- [ ] A017 – Azure Pipelines / WIF / Approvals Grundgerüst.
- [ ] Branch Policies für `main`.
- [ ] Validation-Pipeline.
- [ ] WIF-Service-Connection gegen Testsubscription.
- [ ] Upstream-Fetch-/Review-Prozess für `Azure/avdaccelerator`.
- [ ] Runtime-Artefakt-Grundweg gemäß O007.
- [ ] A056 – eigenes GitHub-Repo nach validierter Migration stilllegen/archivieren.

**Exit-Kriterium:** Alle weitere Projektarbeit kann vollständig in Azure DevOps erfolgen.

---

# 7. G2 – Baseline / Known Fixes / Preflight

- [ ] A003 – Accelerator-Baseline mit aktuellem Bicep-Tooling build/validieren.
- [ ] relevante Microsoft-Issues/PRs erneut gegen dann aktuellen Upstream prüfen.
- [ ] A025 – Agent-/Bootloader-URL-Fix / PR #873.
- [ ] A035 – SKU-/Zone-/Quota-Preflight.
- [ ] Resource Provider Preflight.
- [ ] Marketplace Image Preflight Germany West Central.
- [ ] erstes What-If aus Azure DevOps.

**Exit-Kriterium:** reproduzierbarer technischer Ausgangsstand.

---

# 8. G3 – MSP Foundation

- [ ] A004 – MSP Foundation / Existing-VNet-Modell.
- [ ] Resource Groups.
- [ ] Single VNet.
- [ ] `snet-avd`.
- [ ] `snet-infra`.
- [ ] optional `snet-private-endpoints`.
- [ ] optional `GatewaySubnet`.
- [ ] NSGs.
- [ ] A034 – NAT Gateway / Explicit Outbound.
- [ ] A033 – erforderliche Service Endpoints.
- [ ] DNS Parameter/Outputs.
- [ ] Accelerator mit `createAvdVnet = false`.
- [ ] idempotentes Re-Deployment.

---

# 9. G4 – Identity

## Entra-only

- [ ] A010 – Entra-only-Profil ohne DC testen.
- [ ] Entra Join.
- [ ] optional Intune Enrollment.

## Klassisches AD DS – nur bei Bedarf

- [ ] A009 – optionales New-Forest-DC-Modul.
- [ ] DC01 statische private IP.
- [ ] AD DS/DNS.
- [ ] VNet DNS.
- [ ] Health Check.
- [ ] Azure Backup.
- [ ] Monitoring gemäß O005.
- [ ] Cloud Sync gemäß O001 als Standard integrieren.
- [ ] Connect Sync als dokumentierte Ausnahme ermöglichen, wenn benötigtes Feature Cloud Sync ausschließt.
- [ ] A011 – ADDS/NewForest testen.

## Existing Forest Migration

- [ ] temporäre S2S-Option.
- [ ] Azure DC hinzufügen.
- [ ] Replikation/DNS.
- [ ] O001-Syncverfahren anhand Kundenanforderungen anwenden.
- [ ] FSMO/Demotion.
- [ ] On-Prem-Server decommissionen.

## Entra Domain Services

- kein O001-Sync-Agent gegen die Managed Domain,
- Microsoft-managed Synchronisierung aus Entra ID berücksichtigen.

---

# 10. G5 – Accelerator Compatibility & Hardening

Details: [`docs/planung/ACCELERATOR-DEEP-DIVE.md`](docs/planung/ACCELERATOR-DEEP-DIVE.md)

## P0

- [ ] A005 – Management-VM Größe entkoppeln. F001
- [ ] A006 – Management-VM Disk minimieren. F002
- [ ] A007 – Management-VM deallokieren. F003
- [ ] A022 – Windows-Server-2022-Legacy-Offer ersetzen. F004
- [ ] A008 – Runtime-Skripte/DSC versionsfest/private Runtime-Artefakte. F005
- [ ] A023 – `utcNow()` aus persistentem Naming entfernen. F006
- [ ] A024 – `CreationTimeUTC`-/Common-Tag-Drift entfernen. F007
- [ ] A025 – Agent-/Bootloader-URL-Fix. F008
- [ ] A026 – Registration Token Lifecycle entkoppeln. F009
- [ ] A027 – Session-Host-Bootstrap idempotent. F010
- [ ] A028 – Windows-Update-Verhalten gemäß O009. F011
- [ ] A029 – Management-VM Secret-/Logging-Hardening. F012
- [ ] A030 – DSC/MOF-Credential-Hardening + Cleanup. F013/F014
- [ ] A031 – Firewall-/Local-Admin-State sicher. F015/F016
- [ ] A032 – Storage-State-Checks/Retry idempotent. F017/F018
- [ ] A036 – Scaling-Plan-Parameterisierung, falls gemäß O008 benötigt. F022

## P1

- [ ] A041 – PowerShell-Runtime-Module versionieren. F023
- [ ] A042 – Preview-API-Audit. F024
- [ ] A043 – Approved Marketplace Image Catalog. F025
- [ ] A044 – Key Vault Standard als KMU-Default prüfen. F026
- [ ] A045 – Storage Identity/RBAC Scope härten. F027
- [ ] A046 – Shared Key Access deaktivieren/testen. F028
- [ ] A047 – FSLogix Defender-Variable/Exclusions. F029
- [ ] A048 – Bicep einzige Source of Truth. F030
- [ ] A049 – `br/public`-/AVM Supply-Chain-Pinning. F031
- [ ] A050 – Security-Group-Array-Semantik. F032

---

# 11. G6 – KMU-Profile / FSLogix

Geplante Profile mindestens:

```text
entra-small
adds-small
```

- [ ] Session Host Count Default 1.
- [ ] VM-Größe parametrierbar.
- [ ] Start VM on Connect gemäß O008.
- [ ] A037 – ausgewähltes Power Management validieren.
- [ ] A012 – FSLogix-Default aus O002 umsetzen.
- [ ] A038 – Entra-only + FSLogix ohne Storage-Key-Default, falls vorgesehen. F020
- [ ] A039 – FSLogix ACL-Isolation / #869. F019
- [ ] A040 – Monitoring DCR, falls gemäß O005 Bestandteil. F021
- [ ] Availability Zones Default aus, sofern nicht anders entschieden.
- [ ] DDoS Default aus.
- [ ] AVD Private Link gemäß O004.

---

# 12. G7 – APP / SQL / Dependency Automation

- [ ] A013 – APP-Modul.
- [ ] A014 – SQL-Modul.
- [ ] parametrierbare Größen/Disks.
- [ ] kein Public IP Default.
- [ ] Domain Join nur bei fachlicher Notwendigkeit.
- [ ] Backup-Anforderungen.
- [ ] A015 – Dependency Automation.

```text
Start: SQL → Health → APP
Stop:  APP vollständig → SQL
```

---

# 13. G8 – Validierung / Release 1.0

Details: [`docs/planung/VALIDIERUNG-UND-RELEASE.md`](docs/planung/VALIDIERUNG-UND-RELEASE.md)

- [ ] A018 – Post-Deployment-Tests.
- [ ] A019 – Kostenmodelle Standardprofile.
- [ ] Preflight vollständig.
- [ ] relevante Testmatrix bestanden.
- [ ] Security-Gates bestanden.
- [ ] Idempotenz/Re-Deployment bestanden.
- [ ] Release-Manifest.
- [ ] Production Approval.

---

# 14. G9 – Optionale / P2-Themen

- [ ] A020 – Resilient-Profil.
- [ ] A021 – Existing Hub/Hub-Spoke.
- [ ] A051 – GPU Policy Fix #872. F036
- [ ] A052 – Custom Image VDOT/WDOT Fix #851. F037
- [ ] A053 – Availability-Zone Fix #847. F038
- [ ] A054 – `AutomaticSessionHostConfiguration` #850 beobachten/PoC. F039

---

# 15. Master-Aufgabenregister

Dieses Register ist die Statusquelle.

| ID | Prio | Aufgabe | Status | Gate |
|---|---:|---|---|---|
| A001 | P0 | Accelerator importieren | `[x]` | Vorarbeit |
| A002 | P0 | Master-Umsetzungsplan anlegen | `[x]` | Vorarbeit |
| A055 | P0 | O001–O010 vollständig schließen | `[ ]` – 1/10 | G0 |
| A016 | P0 | Repository nach Azure Repos migrieren | `[ ]` | G1 |
| A017 | P0 | Azure Pipelines / WIF / Approvals | `[ ]` | G1 |
| A056 | P0 | eigenes GitHub-Repo stilllegen/archivieren | `[ ]` | G1 |
| A003 | P0 | Baseline build/validieren / Known Upstream Bugs | `[ ]` | G2 |
| A035 | P0 | SKU-/Zone-/Quota-Preflight | `[ ]` | G2 |
| A004 | P0 | MSP Foundation / Existing-VNet-Modell | `[ ]` | G3 |
| A033 | P0 | Existing-VNet Service Endpoints | `[ ]` | G3 |
| A034 | P0 | Explicit Outbound/NAT | `[ ]` | G3 |
| A009 | P0 | optionales New-Forest-DC-Modul | `[ ]` | G4 |
| A010 | P0 | Entra-only-Profil ohne DC testen | `[ ]` | G4 |
| A011 | P0 | ADDS/NewForest testen | `[ ]` | G4 |
| A005 | P0 | Management-VM Größe entkoppeln | `[ ]` | G5 |
| A006 | P0 | Management-VM Disk minimieren | `[ ]` | G5 |
| A007 | P0 | Management-VM deallokieren | `[ ]` | G5 |
| A008 | P0 | Runtime-Skripte/DSC versionieren/privat ausliefern | `[ ]` | G5 |
| A022 | P0 | Windows-Server-2022-Legacy-Offer ersetzen | `[ ]` | G5 |
| A023 | P0 | `utcNow()` aus persistentem Naming entfernen | `[ ]` | G5 |
| A024 | P0 | `CreationTimeUTC`-/Common-Tag-Drift entfernen | `[ ]` | G5 |
| A025 | P0 | Agent-/Bootloader URL-Fix #873 | `[ ]` | G2/G5 |
| A026 | P0 | Registration Token Lifecycle entkoppeln | `[ ]` | G5 |
| A027 | P0 | Session-Host Agent Bootstrap idempotent | `[ ]` | G5 |
| A028 | P0 | Session-Host Patchmodell/Windows Update | `[ ]` | G5 |
| A029 | P0 | Management-VM Secret-/Logging-Hardening | `[ ]` | G5 |
| A030 | P0 | Plaintext-DSC-/MOF-Hardening/Cleanup | `[ ]` | G5 |
| A031 | P0 | Management-VM Firewall-/Local-Admin-State | `[ ]` | G5 |
| A032 | P0 | Storage-DSC State Checks/Retry idempotent | `[ ]` | G5 |
| A036 | P1* | Scaling Plan vollständig parametrieren | `[ ]` | G5/G6 |
| A012 | P1* | FSLogix KMU-Default umsetzen | `[ ]` | G6 |
| A037 | P0/P1 | Power Management im 1-Host-Profil validieren | `[ ]` | G6 |
| A038 | P0/P1 | Entra-only + FSLogix ohne Storage-Key-Default | `[ ]` | G6 |
| A039 | P0 | FSLogix ACL-Isolation / #869 | `[ ]` | G6 |
| A040 | P1 | Monitoring DCR 24H2/25H2 | `[ ]` | G6 |
| A041 | P1 | PowerShell Runtime Module versionieren | `[ ]` | G5 |
| A042 | P1 | Preview-API-Audit | `[ ]` | G5 |
| A043 | P1 | Approved Marketplace Image Catalog | `[ ]` | G5 |
| A044 | P1 | Key Vault Standard als KMU-Default prüfen | `[ ]` | G5/G6 |
| A045 | P1 | Storage Identity/RBAC Scope härten | `[ ]` | G6 |
| A046 | P1 | Shared Key Access deaktivieren/testen | `[ ]` | G6 |
| A047 | P1 | FSLogix Defender-Variable/Exclusions | `[ ]` | G5/G6 |
| A048 | P1 | Bicep einzige Source of Truth | `[ ]` | G1/G5 |
| A049 | P1 | `br/public`-/AVM Supply-Chain-Pinning | `[ ]` | G2/G5 |
| A050 | P1 | Security-Group-Array-Semantik | `[ ]` | G5 |
| A013 | P1 | APP-Modul | `[ ]` | G7 |
| A014 | P1 | SQL-Modul | `[ ]` | G7 |
| A015 | P1 | Dependency Automation | `[ ]` | G7 |
| A018 | P1 | Post-Deployment-Tests | `[ ]` | G8 |
| A019 | P1 | Kostenmodelle Standardprofile | `[ ]` | G8 |
| A020 | P2 | Resilient-Profil | `[ ]` | G9 |
| A021 | P2 | Existing Hub/Hub-Spoke | `[ ]` | G9 |
| A051 | P2 | GPU Policy Fix #872 | `[ ]` | G9 |
| A052 | P2 | Custom Image VDOT/WDOT Fix #851 | `[ ]` | G9 |
| A053 | P2 | Availability-Zone Fix #847 | `[ ]` | G9 |
| A054 | P2 | AutomaticSessionHostConfiguration #850 PoC | `[ ]` | G9 |

`P1*`: kann nach G0 zu P0 werden, falls die Architekturentscheidung das Feature zum Release-1.0-Default macht.

---

# 16. Kurzfassung beschlossener Architektur

- Microsoft Accelerator bleibt AVD-Core.
- 1 Session Host ist KMU-Default.
- kein genereller DC-Zwang.
- Entra-only ist vollwertiges Zielmodell.
- klassisches AD DS nur bei Bedarf; CostOptimized darf mit einem DC arbeiten.
- **bei klassischem AD DS: Entra Cloud Sync Standard, Connect Sync nur bei konkreter Funktionslücke.**
- Entra Domain Services betreibt keinen eigenen O001-Sync-Agent gegen die Managed Domain.
- On-Prem-Server ist nur Migrationsquelle.
- Single VNet als KMU-Default.
- Foundation vor Accelerator; `createAvdVnet = false`.
- kein Azure Firewall Default.
- APP/SQL separate MSP-Workloads.
- Management-VM bleibt erhalten, wird verkleinert, gehärtet und deallokiert.
- Azure DevOps ist dauerhafte Entwicklungs-/Deploymentplattform.

---

# 17. Upstream-Regel

Bei Accelerator-Arbeit immer prüfen:

```text
Microsoft main
+
relevante offene Issues
+
relevante offene PRs
```

Kein ungeprüftes automatisches Merge. Jedes MSP-Release dokumentiert Upstream-Commit + zusätzliche Fixes + eigenes Patch-Delta.

---

# 18. Änderungsregel für diesen Masterplan

Neue technische Details gehören in das passende Detaildokument.

In den Master gehören nur:

- Gate-/Statusänderungen,
- Aufgaben-IDs,
- Prioritätsänderungen,
- Reihenfolge,
- Architekturentscheidungen als Kurzreferenz,
- nächster Arbeitsschritt.
