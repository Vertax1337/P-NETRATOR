# MSP AVD Accelerator – Umsetzungsplan

> **Rolle:** Master-/Steuerungsdokument und Single Source of Truth für **Status, Gates, Reihenfolge und Prioritäten**  
> **Stand:** 2026-08-07  
> **Repository:** `Vertax1337/P-NETRATOR`  
> **Microsoft-Upstream:** `Azure/avdaccelerator`  
> **Importierter Upstream-Stand:** `63c55c857f77783c01812c935957cd8c9c67a5a7`  
> **Ursprünglicher P-NETRATOR-Stand:** `archive/p-netrator-original`

---

# 1. Arbeitsmodell / Source of Truth

Dieses Dokument bleibt die zentrale Projektsteuerung. Fachliche Details werden bewusst in thematische Unterdokumente ausgelagert.

**Nur dieses Dokument ist verbindlich für:**

- aktuellen Projektstatus,
- Reihenfolge der Phasen,
- Gate-Status,
- Priorität,
- Aufgabenstatus,
- nächster Arbeitsschritt.

Die Detaildokumente sind verbindlich für technische Begründungen, Findings, Entscheidungsoptionen und Testanforderungen, führen aber **keinen konkurrierenden Projektstatus**.

## Dokumentenstruktur

| Dokument | Zweck |
|---|---|
| [`UMSETZUNGSPLAN.md`](UMSETZUNGSPLAN.md) | Master: Gates, Phasen, Aufgaben, Prioritäten, Status |
| [`docs/planung/ARCHITEKTUR-ENTSCHEIDUNGEN.md`](docs/planung/ARCHITEKTUR-ENTSCHEIDUNGEN.md) | beschlossene Leitplanken + offene O001–O010 |
| [`docs/planung/AZURE-DEVOPS.md`](docs/planung/AZURE-DEVOPS.md) | Azure-DevOps-Migration, WIF, Pipelines, Runtime-Artefakte, GitHub-Stilllegung |
| [`docs/planung/ACCELERATOR-DEEP-DIVE.md`](docs/planung/ACCELERATOR-DEEP-DIVE.md) | Accelerator-Findings F001–F039 inkl. Auswirkungen und Maßnahmen |
| [`docs/planung/VALIDIERUNG-UND-RELEASE.md`](docs/planung/VALIDIERUNG-UND-RELEASE.md) | Preflight, Testmatrix, Security-/Kosten-Guardrails, Release-DoD |
| [`docs/planung/KUNDEN-ONBOARDING.md`](docs/planung/KUNDEN-ONBOARDING.md) | operativer Standardprozess vom Kunden-Assessment bis Deployment, Validierung und Betriebsübergabe |

## Pflege-Regel

Bei jeder Arbeitssitzung:

1. dieses Masterdokument lesen,
2. aktuelles Gate und `NÄCHSTER SCHRITT` prüfen,
3. nur das für die Aufgabe relevante Detaildokument lesen,
4. neue Erkenntnisse im passenden Detaildokument erfassen,
5. Status/Priorität ausschließlich hier aktualisieren,
6. erst danach Code ändern.

---

# 2. Projektziel

Der Microsoft AVD Accelerator bleibt die AVD Workload Deployment Engine. Darum entsteht eine dünne MSP-Schicht für:

- kostenoptimierte KMU-Defaults,
- Single-VNet-Foundation,
- Entra-only oder optional AD DS,
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
- Branchensoftware kann AD DS erforderlich machen, muss es aber nicht,
- hohe Kostenpriorität,
- keine unnötige Enterprise-Komplexität.

---

# 3. Aktueller Projektstatus

## Aktives Gate: G0 – Architekturentscheidungen

**Status: OFFEN / BLOCKIERT IMPLEMENTIERUNG**

Vor Beginn der eigentlichen Implementierung müssen **alle offenen Architekturentscheidungen O001–O010 geschlossen** werden.

Details und Entscheidungskriterien:

[`docs/planung/ARCHITEKTUR-ENTSCHEIDUNGEN.md`](docs/planung/ARCHITEKTUR-ENTSCHEIDUNGEN.md)

Aktueller Stand:

| Entscheidung | Thema | Status |
|---|---|---|
| O001 | Entra Cloud Sync vs. Entra Connect Sync | `OFFEN` |
| O002 | FSLogix Default | `OFFEN` |
| O003 | Azure Files Tier | `OFFEN` |
| O004 | Service Endpoint vs. Private Endpoint | `OFFEN` |
| O005 | Monitoring Baseline | `OFFEN` |
| O006 | Standard Session Host SKU / Größenstrategie | `OFFEN` |
| O007 | Runtime-Artefakte bei privatem Repository | `OFFEN` |
| O008 | NativeAutoscale vs. Custom AutoShutdown | `OFFEN` |
| O009 | Patch in Place vs. Immutable/Replace | `OFFEN` |
| O010 | gehärtetes DSC vs. neuer Storage-/Domain-Join-Helper | `OFFEN` |

### NÄCHSTER SCHRITT

> **O001–O010 systematisch entscheiden und dokumentieren.**

Bis G0 geschlossen ist, werden keine dauerhaften Foundation-/Accelerator-/Profil-Implementierungen begonnen.

Recherche und isolierte, verwerfbare PoCs zur Entscheidungsfindung sind zulässig.

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
    Entra-only und optional ADDS-Pfade
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

Ein Gate wird erst verlassen, wenn sein Exit-Kriterium erfüllt ist.

---

# 5. G0 – Architekturentscheidungen

**Priorität: P0 / zuerst**

- [ ] A055 – O001–O010 vollständig entscheiden.
- [ ] Entscheidungen in `ARCHITEKTUR-ENTSCHEIDUNGEN.md` festhalten.
- [ ] Auswirkungen auf Aufgaben/Phasen dieses Masterplans einarbeiten.
- [ ] keine Entscheidung stillschweigend während der Implementierung treffen.

**Exit-Kriterium:** O001–O010 stehen nicht mehr auf `OFFEN`.

---

# 6. G1 – Azure DevOps Migration / Bootstrap

**Priorität: P0 / erste Implementierungsphase**

Azure DevOps wird bewusst **nicht** ans Projektende verschoben. Alle nachfolgenden Implementierungen sollen bereits auf der Zielplattform entstehen.

Details:

[`docs/planung/AZURE-DEVOPS.md`](docs/planung/AZURE-DEVOPS.md)

Aufgaben:

- [ ] A016 – Repository vollständig nach Azure Repos migrieren. **P0**
- [ ] A017 – Azure Pipelines / WIF / Approvals Grundgerüst. **P0**
- [ ] Branch Policies für `main`.
- [ ] Validation-Pipeline als erster funktionierender Pipelinepfad.
- [ ] Service Connection gegen Testsubscription mit Workload Identity Federation.
- [ ] Upstream-Fetch-/Review-Prozess für öffentliches `Azure/avdaccelerator` dokumentieren.
- [ ] Runtime-Artefakt-Grundweg gemäß O007 vorbereiten.
- [ ] A056 – nach vollständiger Validierung eigenes GitHub-Repo archivieren/deaktivieren, sodass kein kostenpflichtiger laufender GitHub-Betrieb erforderlich ist. **P0**

**Exit-Kriterium:** Alle weitere Projektarbeit kann vollständig in Azure DevOps erfolgen und das eigene GitHub-Repo wird technisch nicht mehr benötigt.

---

# 7. G2 – Baseline / Known Fixes / Preflight

**Priorität: P0**

- [ ] A003 – Accelerator-Baseline mit aktuellem Bicep-Tooling build/validieren.
- [ ] relevante offene Microsoft-Issues/PRs erneut gegen dann aktuellen Upstream prüfen.
- [ ] A025 – kaputte Agent-/Bootloader-URLs / PR #873 berücksichtigen.
- [ ] A035 – SKU-/Zone-/Quota-Preflight.
- [ ] Resource Provider Preflight.
- [ ] Marketplace Image Preflight Germany West Central.
- [ ] entscheiden, ob Testbaseline bereits bekannte P0-Upstream-Fixes enthalten muss.
- [ ] erstes What-If aus Azure DevOps.

**Exit-Kriterium:** Bekannter, reproduzierbarer technischer Ausgangsstand für weitere MSP-Arbeiten.

---

# 8. G3 – MSP Foundation

**Priorität: P0**

- [ ] A004 – MSP Foundation / Existing-VNet-Modell.
- [ ] Resource Groups.
- [ ] Single VNet.
- [ ] `snet-avd`.
- [ ] `snet-infra`.
- [ ] optional `snet-private-endpoints`.
- [ ] optional `GatewaySubnet`.
- [ ] NSGs.
- [ ] A034 – NAT Gateway / Explicit Outbound.
- [ ] A033 – erforderliche Service Endpoints bei Existing VNet.
- [ ] DNS Parameter/Outputs.
- [ ] Accelerator erhält `createAvdVnet = false` und vorhandene Subnet IDs.
- [ ] idempotentes Re-Deployment.

**Exit-Kriterium:** Netzwerk/Foundation funktioniert unabhängig davon, ob später Entra-only oder ADDS verwendet wird.

---

# 9. G4 – Identity

Die genaue Umsetzung hängt von den G0-Entscheidungen ab.

## Entra-only

- [ ] A010 – Entra-only-Profil ohne DC technisch testen.
- [ ] Entra Join.
- [ ] optional Intune Enrollment gemäß Profil.

## ADDS – nur wenn benötigt

- [ ] A009 – optionales New-Forest-DC-Modul.
- [ ] DC01 statische private IP.
- [ ] AD DS/DNS.
- [ ] VNet DNS.
- [ ] Health Check.
- [ ] Azure Backup.
- [ ] Monitoring gemäß O005.
- [ ] A011 – ADDS/NewForest Pfad testen.

## Existing Forest Migration

- [ ] temporäre S2S-Option.
- [ ] Azure DC hinzufügen.
- [ ] Replikation/DNS.
- [ ] FSMO/Demotion.
- [ ] alter On-Prem-Server wird nach Migration decommissioned.

**Exit-Kriterium:** Die für Release 1.0 vorgesehenen Identity-Pfade sind reproduzierbar.

---

# 10. G5 – Accelerator Compatibility & Hardening

Technische Details und alle Findings F001–F039:

[`docs/planung/ACCELERATOR-DEEP-DIVE.md`](docs/planung/ACCELERATOR-DEEP-DIVE.md)

## Release-1.0-P0-Arbeitspakete

- [ ] A005 – Management-VM Größe entkoppeln. F001
- [ ] A006 – Management-VM Disk minimieren. F002
- [ ] A007 – Management-VM automatisch deallokieren. F003
- [ ] A022 – Windows-Server-2022-Legacy-Offer ersetzen. F004
- [ ] A008 – Runtime-Skripte/DSC versionsfest/private Runtime-Artefakte. F005
- [ ] A023 – `utcNow()` aus persistentem Naming entfernen. F006
- [ ] A024 – Common-Tag-/`CreationTimeUTC`-Drift entfernen. F007
- [ ] A025 – Agent-/Bootloader-URL-Fix. F008
- [ ] A026 – Registration Token Lifecycle entkoppeln. F009
- [ ] A027 – Session-Host-Bootstrap idempotent. F010
- [ ] A028 – Windows-Update-Verhalten gemäß O009. F011
- [ ] A029 – Management-VM Secret-/Logging-Hardening. F012
- [ ] A030 – DSC/MOF-Credential-Hardening + Cleanup. F013/F014
- [ ] A031 – Firewall-/Local-Admin-Zustand sicher. F015/F016
- [ ] A032 – Storage-State-Checks und Retry idempotent. F017/F018
- [ ] A036 – Scaling-Plan-Parameterisierung, falls gemäß O008 benötigt. F022

## P1-Hardening nach P0

- [ ] A041 – PowerShell-Runtime-Module versionieren. F023
- [ ] A042 – Preview-API-Audit. F024
- [ ] A043 – Approved Marketplace Image Catalog/Manifest. F025
- [ ] A044 – Key Vault Premium → Standard als KMU-Default prüfen. F026
- [ ] A045 – Storage Identity/RBAC Scope härten. F027
- [ ] A046 – Shared Key Access nach Identity-Migration deaktivieren/testen. F028
- [ ] A047 – FSLogix Defender-Variable/Exclusions korrigieren. F029
- [ ] A048 – Bicep einzige Source of Truth; ARM JSON nur Build-Artefakt. F030
- [ ] A049 – `br/public`-/AVM Supply-Chain-Pinning. F031
- [ ] A050 – Security-Group-Array-Semantik. F032

**Exit-Kriterium:** Relevantes P0-Patchset ist implementiert, dokumentiert und im Testprofil validiert.

---

# 11. G6 – KMU-Profile / FSLogix

Die tatsächlichen Defaults werden durch O002–O006/O008/O009 bestimmt.

Mindestens geplante Profile:

```text
entra-small
adds-small
```

Aufgaben:

- [ ] Session Host Count Default 1.
- [ ] VM-Größe parametrierbar.
- [ ] Start VM on Connect gemäß O008.
- [ ] A037 – ausgewähltes Power Management aus O008 testen.
- [ ] A012 – FSLogix-Default aus O002 umsetzen.
- [ ] A038 – Entra-only + FSLogix ohne Storage-Key-Default, falls im Profil vorgesehen. F020
- [ ] A039 – FSLogix ACL-Isolation / Upstream #869. F019
- [ ] A040 – Monitoring DCR, falls gemäß O005 Bestandteil. F021
- [ ] Availability Zones Default aus, sofern Architekturentscheidung nichts anderes vorgibt.
- [ ] DDoS Default aus.
- [ ] AVD Private Link Default aus, sofern O004 nichts anderes bestimmt.

**Exit-Kriterium:** Freizugebende KMU-Profile besitzen eindeutige technische Defaults und funktionieren reproduzierbar.

---

# 12. G7 – APP / SQL / Dependency Automation

- [ ] A013 – APP-Modul.
- [ ] A014 – SQL-Modul.
- [ ] parametrierbare Größen/Disks.
- [ ] kein Public IP Default.
- [ ] Domain Join nur bei fachlicher Notwendigkeit.
- [ ] Backup-Anforderungen.
- [ ] A015 – Dependency Automation.

Runtime:

```text
Start: SQL → Health → APP
Stop:  APP vollständig → SQL
```

Eigene Dependency-Automation darf nicht denselben Session Host parallel zu AVD Autoscale steuern.

---

# 13. G8 – Validierung / Release 1.0

Details:

[`docs/planung/VALIDIERUNG-UND-RELEASE.md`](docs/planung/VALIDIERUNG-UND-RELEASE.md)

Master-Aufgaben:

- [ ] A018 – Post-Deployment-Tests.
- [ ] A019 – Kostenmodelle der freigegebenen Standardprofile.
- [ ] Preflight vollständig.
- [ ] relevante Testmatrix bestanden.
- [ ] Security-Gates bestanden.
- [ ] Idempotenz/Re-Deployment bestanden.
- [ ] Release-Manifest erzeugt.
- [ ] Production Approval eingerichtet.

**Exit-Kriterium:** Release-DoD im Detaildokument erfüllt.

---

# 14. G9 – Optionale / P2-Themen

Nicht blockierend für Release 1.0:

- [ ] A020 – Resilient-Profil.
- [ ] A021 – Existing Hub/Hub-Spoke.
- [ ] A051 – GPU Policy Fix #872. F036
- [ ] A052 – Custom Image VDOT/WDOT Fix #851. F037
- [ ] A053 – Availability-Zone Fix #847. F038
- [ ] A054 – `AutomaticSessionHostConfiguration` #850 beobachten/PoC. F039

---

# 15. Master-Aufgabenregister

Dieses Register ist die Statusquelle. Detaildokumente dürfen die Aufgaben erläutern, aber den Status nicht eigenständig verändern.

| ID | Prio | Aufgabe | Status | Gate |
|---|---:|---|---|---|
| A001 | P0 | Accelerator importieren | `[x]` | Vorarbeit |
| A002 | P0 | Master-Umsetzungsplan anlegen | `[x]` | Vorarbeit |
| A055 | P0 | O001–O010 vollständig schließen | `[ ]` | G0 |
| A016 | **P0** | Repository nach Azure Repos migrieren | `[ ]` | G1 |
| A017 | **P0** | Azure Pipelines / WIF / Approvals Grundgerüst | `[ ]` | G1 |
| A056 | **P0** | eigenes GitHub-Repo nach validierter Migration stilllegen/archivieren | `[ ]` | G1 |
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
| A028 | P0 | Session-Host Patchmodell/Windows Update gemäß O009 | `[ ]` | G5 |
| A029 | P0 | Management-VM Secret-/Logging-Hardening | `[ ]` | G5 |
| A030 | P0 | Plaintext-DSC-/MOF-Hardening/Cleanup | `[ ]` | G5 |
| A031 | P0 | Management-VM Firewall-/Local-Admin-State | `[ ]` | G5 |
| A032 | P0 | Storage-DSC State Checks/Retry idempotent | `[ ]` | G5 |
| A036 | P1* | Scaling Plan vollständig parametrieren | `[ ]` | G5/G6 |
| A012 | P1* | FSLogix KMU-Default umsetzen | `[ ]` | G6 |
| A037 | P0/P1 | ausgewähltes Power Management im 1-Host-Profil validieren | `[ ]` | G6 |
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

`P1*` bedeutet: tatsächliche Priorität kann nach G0 auf P0 steigen, wenn die betreffende Architekturentscheidung das Feature zum Release-1.0-Default macht.

---

# 16. Zusammenfassung der bereits beschlossenen Architektur

Die vollständige Begründung steht in `ARCHITEKTUR-ENTSCHEIDUNGEN.md`.

Kurzfassung:

- Microsoft Accelerator bleibt AVD-Core.
- kein genereller DC-Zwang.
- Entra-only ist vollwertiges Zielmodell.
- ADDS nur bei technischer/fachlicher Notwendigkeit.
- bei ADDS CostOptimized ist ein DC zulässig; zwei DCs optional resilient.
- On-Prem-Server ist nur Migrationsquelle.
- Single VNet als KMU-Default.
- Foundation vor Accelerator; `createAvdVnet = false`.
- kein Azure Firewall Default.
- 1 Session Host Default.
- APP/SQL separate MSP-Workloads.
- Management-VM bleibt erhalten, wird aber verkleinert, gehärtet und deallokiert.
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

Vor Eigenentwicklung prüfen, ob Microsoft bereits einen Fix vorbereitet hat.

Kein ungeprüftes automatisches Upstream-Merge.

Jedes MSP-Release dokumentiert Upstream-Commit + zusätzliche Fixes + eigenes Patch-Delta.

---

# 18. Änderungsregel für diesen Masterplan

Neue technische Details gehören grundsätzlich in das passende Detaildokument.

In diesen Master gehören nur:

- neue/geschlossene Gates,
- neue Aufgaben-IDs,
- Prioritätsänderungen,
- Statusänderungen,
- geänderte Reihenfolge,
- neue Architekturentscheidung als Kurzreferenz,
- nächster Arbeitsschritt.

Damit soll der Master dauerhaft lesbar bleiben und nicht erneut zu einem technischen Sammeldokument anwachsen.