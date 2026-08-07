# MSP AVD Accelerator – Azure DevOps Zielarchitektur und Migration

> **Rolle dieses Dokuments:** Detailplan für die erste Implementierungsphase nach Abschluss des Architektur-Gates  
> **Projektstatus und Reihenfolge:** [`../../UMSETZUNGSPLAN.md`](../../UMSETZUNGSPLAN.md)  
> **Architektur-Gate:** [`ARCHITEKTUR-ENTSCHEIDUNGEN.md`](ARCHITEKTUR-ENTSCHEIDUNGEN.md)

---

# 1. Ziel

Azure DevOps ist nicht eine spätere Betriebsoptimierung, sondern die **erste technische Implementierungsbasis** des Projekts.

Nach Abschluss aller offenen Architekturentscheidungen wird das Projekt sofort von GitHub nach Azure DevOps verlagert, damit alle weiteren Arbeiten bereits auf der späteren Zielplattform entstehen.

Zielplattform:

```text
Azure DevOps
├── Azure Repos
├── Azure Pipelines
├── Service Connections mit Workload Identity Federation
├── Environments / Approvals
└── Pipeline Artifacts
```

Das eigene GitHub-Repository wird nach erfolgreicher Verifikation der Migration aus Kostengründen nicht mehr für den laufenden Projektbetrieb benötigt.

---

# 2. Grundprinzip

```text
Microsoft GitHub Upstream (öffentlich)
          │
          │ nur Upstream-Quelle
          ▼
Private Azure Repos
          │
          ├── Microsoft Accelerator Fork
          ├── MSP Foundation
          ├── Runtime Source
          ├── Kundenprofile
          └── Pipelines
          │
          ▼
Azure Pipelines
          │
          ├── Validate
          ├── Bicep Build / Restore
          ├── What-If
          ├── Runtime Package
          ├── Deploy
          └── Post-Deployment Validation
          │
          ▼
Kunden-Azure-Subscription
```

Das Microsoft-Upstream-Repository bleibt öffentlich erreichbar. Dafür ist **kein eigenes kostenpflichtiges GitHub-Repository** erforderlich.

Upstream-Updates können später beispielsweise durch:

```text
git fetch https://github.com/Azure/avdaccelerator.git
```

oder einen kontrollierten Azure-Pipeline-/Admin-Workflow geprüft werden.

---

# 3. Migrationsreihenfolge

## M01 – Azure DevOps Projekt vorbereiten

- Azure-DevOps-Organisation/Projekt festlegen.
- Berechtigungsmodell definieren.
- Azure Repos aktivieren.
- Branch Policies vorbereiten.
- Naming für Service Connections und Environments festlegen.

## M02 – Repository nach Azure Repos migrieren

Der vollständige aktuelle Git-Verlauf soll erhalten bleiben, einschließlich:

- importiertem Microsoft Accelerator,
- bisheriger MSP-Planungscommits,
- Archiv-Branch `archive/p-netrator-original`, sofern weiterhin gewünscht.

Nach Migration müssen mindestens vorhanden sein:

```text
main
archive/p-netrator-original
```

und die vollständige Commit-Historie.

## M03 – Integrität verifizieren

Prüfen:

- Commit-SHAs/History nachvollziehbar,
- Dateien vollständig,
- Tags/Branches soweit benötigt vorhanden,
- `UMSETZUNGSPLAN.md` und Detaildokumente vorhanden,
- Bicep-/AVM-Dateien unverändert übertragen.

## M04 – Azure Resource Manager Service Connection

Ziel:

```text
Workload Identity Federation
```

Keine langlebigen Client Secrets als Standard.

Für MSP-Betrieb prüfen:

- Service Connection pro Kunde/Subscription oder geeignetes alternatives Isolationsmodell,
- Least-Privilege RBAC,
- Trennung Test/Prod,
- nachvollziehbare Naming-Konvention.

Beispiel:

```text
sc-kunde-a-prod
sc-kunde-a-test
```

## M05 – Pipeline-Grundgerüst

Die Pipeline wird früh aufgebaut, bevor produktive Bicep-Implementierung startet.

Zielstages:

```text
VALIDATE
   ↓
PREFLIGHT
   ↓
RUNTIME ARTIFACT PREPARE
   ↓
FOUNDATION
   ↓
IDENTITY (conditional)
   ↓
AVD ACCELERATOR
   ↓
WORKLOAD (conditional)
   ↓
CONFIGURATION
   ↓
VALIDATION
   ↓
MGMT VM DEALLOCATE (wenn vorhanden)
```

Zu Beginn dürfen noch nicht implementierte Stages als Templates/Platzhalter existieren.

## M06 – Branch Policies

Für `main` mindestens prüfen/einrichten:

- kein direkter Push,
- Pull Request erforderlich,
- erfolgreiche Validation Pipeline erforderlich,
- mindestens ein Reviewer, soweit organisatorisch sinnvoll,
- Kommentare/Review-Konflikte vor Merge auflösen.

## M07 – Production Approval

Produktive Kundendeployments dürfen nicht automatisch bei jedem Merge starten.

Bevorzugt:

```text
Merge nach main
→ kein automatischer Kunden-Deploy
→ manuell gestartete Deployment Pipeline
→ What-If
→ Azure DevOps Environment Approval
→ Deploy
```

## M08 – GitHub stilllegen

Erst wenn folgende Punkte erfüllt sind:

- Azure-Repos-Historie geprüft,
- alle benötigten Branches vorhanden,
- Pipeline kann Repo lesen/builden,
- Upstream-Update-Verfahren dokumentiert,
- keine Dokumentation verweist zwingend auf das eigene GitHub-Repo,
- benötigte Runtime-Skripte werden nicht aus dem eigenen GitHub-Repo geladen.

Dann kann das eigene GitHub-Repository entsprechend der Kostenstrategie archiviert/deaktiviert werden.

---

# 4. Runtime-Artefakte

Das Quellcode-Repository soll privat bleiben können.

Ziel-VMs sollen **nicht direkt auf Azure Repos zugreifen** und keine Azure-DevOps-PATs erhalten.

Die endgültige Variante wird durch Architekturentscheidung O007 festgelegt.

Der aktuell analysierte V1-Kandidat ist:

```text
Private Azure Repos
→ Azure Pipeline
→ Runtime-Dateien paketieren
→ Release-Manifest + Hashes
→ privater Azure Storage
→ kurzlebige read-only User-Delegation-SAS
→ Custom Script Extension / Helper
```

Pipeline Artifacts dienen dabei der internen Nachvollziehbarkeit/Stage-Übergabe, nicht als direkte Runtime-Downloadquelle der Ziel-VM.

Runtime Release Manifest mindestens:

```text
MSP Release Version
Git Commit SHA
Microsoft Upstream Commit SHA
Set-SessionHostConfiguration.ps1 SHA256
Manual-DSC-Storage-Scripts.ps1 SHA256
DSCStorageScripts.zip SHA256
PowerShell Module Versions
Marketplace Image Reference
übernommene Upstream-Fixes
```

---

# 5. Pipeline-Parameter

Ziel ist eine zentrale Codebasis mit kundenbezogener Parametrisierung.

Mögliche Runtime-Parameter:

```yaml
parameters:
- name: customer
  type: string

- name: environment
  type: string
  default: prod

- name: deploymentMode
  type: string
  values:
    - WhatIf
    - Deploy
```

Produktive Deployment-Pipeline initial bevorzugt:

```yaml
trigger: none
pr: none
```

PR-/Validation-Pipeline separat automatisiert.

---

# 6. Secrets und Kundenparameter

Kundenparameter dürfen keine Kennwörter enthalten.

Nicht in Repository-Dateien:

- Domain Join Password,
- Storage Keys,
- PATs,
- Client Secrets,
- SAS mit langer Laufzeit.

Bevorzugte Quellen:

- Workload Identity Federation,
- Azure Key Vault,
- geschützte Pipeline-Variablen nur wenn erforderlich,
- kurzlebige, zur Laufzeit erzeugte Tokens/SAS.

---

# 7. Bicep als Source of Truth

Nach der Migration ist die produktive Entwicklungsregel:

```text
Bicep = Source of Truth
```

Vorcompilierte ARM-JSON-Dateien werden nicht parallel manuell gepflegt.

Wenn sie für Portal-/Kompatibilitätszwecke benötigt werden:

```text
Bicep
→ Pipeline Build
→ ARM JSON Artifact
```

---

# 8. Upstream-Integration ohne eigenes GitHub-Repo

Das Microsoft-Repository bleibt die Upstream-Quelle:

```text
https://github.com/Azure/avdaccelerator
```

Bei einem Upstream-Review sind zu prüfen:

- `main`,
- relevante offene Issues,
- relevante offene PRs.

Ablauf:

```text
Microsoft Upstream
→ Fetch/Compare
→ Azure-Repos Integration Branch
→ Diff gegen MSP-Patchset
→ Build
→ What-If/Testmatrix
→ Review
→ main
```

Kein automatisches ungeprüftes Merge.

---

# 9. Abschlusskriterium der Azure-DevOps-Migrationsphase

- [ ] Azure DevOps Projekt/Repo vorhanden.
- [ ] vollständige Repository-Historie übertragen.
- [ ] Branches validiert.
- [ ] `main` geschützt.
- [ ] Validation-Pipeline funktioniert.
- [ ] WIF-Service-Connection funktioniert gegen Testsubscription.
- [ ] Pipeline kann `bicep restore`/`bicep build` ausführen.
- [ ] What-If kann technisch gestartet werden.
- [ ] Environment/Approval-Konzept eingerichtet.
- [ ] Runtime-Artefakt-Grundweg gemäß O007 vorbereitet.
- [ ] Upstream-Fetch-/Review-Prozess dokumentiert.
- [ ] eigenes GitHub-Repo kann anschließend ohne Funktionsverlust stillgelegt werden.

Erst nach diesem Gate wird die eigentliche Accelerator-/Foundation-Implementierung in Azure DevOps fortgeführt.
