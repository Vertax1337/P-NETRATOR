# Microsoft AVD Accelerator – Deep-Dive für das MSP-KMU-Projekt

> **Rolle dieses Dokuments:** Technische Befund-/Maßnahmenquelle für den importierten Accelerator  
> **Projektstatus/Priorität:** [`../../UMSETZUNGSPLAN.md`](../../UMSETZUNGSPLAN.md)  
> **Architekturentscheidungen:** [`ARCHITEKTUR-ENTSCHEIDUNGEN.md`](ARCHITEKTUR-ENTSCHEIDUNGEN.md)  
> **Importierter Upstream-Stand:** `63c55c857f77783c01812c935957cd8c9c67a5a7`  
> **Deep-Dive-Stand:** 2026-08-07

---

# 1. Ziel der Analyse

Der Accelerator wird nicht nur auf syntaktische Korrektheit, sondern auf Eignung für unser konkretes Ziel geprüft:

- kleine/mittlere Kunden,
- häufig 1 Session Host,
- Entra-only oder optional AD DS,
- niedrige Fixkosten,
- privates Azure-DevOps-Repository,
- reproduzierbare Deployments,
- idempotentes Re-Deployment,
- sichere Runtime-/Secret-Verarbeitung,
- minimaler Patch-Delta zum Microsoft-Upstream.

Die Findings werden klassifiziert in:

- **Accelerator-Patch** – Änderung am importierten Microsoft-Code erforderlich.
- **MSP-Foundation** – durch unsere vorgeschaltete Schicht lösen; Microsoft-Code nicht ändern.
- **Feature-Patch** – nur relevant, wenn das Feature aktiviert wird.
- **Release Engineering / Security** – Pipeline-/Supply-Chain-/Betriebsmaßnahme.
- **P2/Strategisch** – nicht Release-1.0-kritisch.

---

# 2. Upstream-Status

Der importierte Stand war am 2026-08-07 weiterhin der aktuelle `main`-Stand des Microsoft-Repositories. Der zugrunde liegende Commit stammt vom 2026-01-21.

Wichtige offene Microsoft-Themen, die beim Deep Dive berücksichtigt wurden:

- Issue `#874` – kaputter AVD-Bootloader-Download,
- PR `#873` – Agent-/Bootloader-URLs auf stabile Microsoft fwlinks,
- Issue `#869` – zu breite FSLogix-NTFS-Rechte bei EntraIDKerberos,
- Issue `#867` / PR `#870` – AVD Insights DCR/Performance Counter auf Win11 24H2/25H2,
- Issue `#866` – AzureFilesDomainJoin/DSCStorageScripts Retry-/File-Lock-Problem,
- PR `#851` – Custom Image/VDOT durch bewegliche Upstream-Abhängigkeit gebrochen,
- PR `#847` – Availability-Zone-/Redeployment-Probleme,
- PR `#850` – `AutomaticSessionHostConfiguration` Preview,
- Issue `#872` – GPU-DINE-Policy ohne erforderliche Role Assignment.

**Konsequenz für Upstream-Pflege:** Nicht nur Microsoft `main`, sondern immer auch relevante offene Issues und PRs prüfen.

---

# 3. Konsolidierte Findings-Matrix

| ID | Prio | Kategorie | Befund | Auswirkung | Empfohlene Maßnahme |
|---|---:|---|---|---|---|
| F001 | P0 | Accelerator-Patch | Management-VM übernimmt Session-Host-Größe | unnötige Compute-Kosten | eigene `managementVmSize`; kleine Burst-SKU testen |
| F002 | P0 | Accelerator-Patch | Management-VM OS-Disk nicht konsequent kostenminimal | unnötige persistente Kosten | Small-Disk-Image, Standard_LRS, minimale technisch sinnvolle Disk |
| F003 | P0 | Pipeline/MSP | Management-VM bleibt nach Arbeit aktiv | unnötige Compute-Kosten | nach erfolgreicher Konfiguration automatisch **deallokieren**, nicht löschen |
| F004 | P0 | Accelerator-Patch | Management-VM nutzt Windows Server 2022 Legacy-Offer `WindowsServer` | veraltete Marketplace-Referenz | auf aktuelles `windowsserver2022` Offer migrieren und Ziel-SKU in Germany West Central validieren |
| F005 | P0 | Accelerator-Patch | Script-/DSC-Quellen zeigen auf öffentliche `main`-/GitHub-Artefakte | getesteter Commit kann später anderen Code ausführen | Runtime-Artefakte versionieren und aus eigener Release-Quelle bereitstellen |
| F006 | P0 | Accelerator-Patch | persistente Ressourcennamen hängen indirekt von `utcNow()` ab | Re-Deployment kann neue KV-/Storage-Ressourcen erzeugen | deterministisches Naming ohne Zeitstempel |
| F007 | P0 | Accelerator-Patch | Common-Tag `CreationTimeUTC` wird bei Re-Deployment neu berechnet | unnötiger What-If-/Tag-Drift | Creation-Timestamp nicht als bei jedem Deployment neu berechneten Common-Tag führen; Deploymentzeit separat protokollieren |
| F008 | P0 | Accelerator-Patch | AVD Bootloader URL `RWrxrH` ist bereits kaputt | Session-Host-Provisionierung kann scheitern | Upstream PR #873 übernehmen; Bootstrap-URLs zentralisieren |
| F009 | P0 | Accelerator-Patch | Host-Pool-Modul erzeugt bei jedem Deployment neuen Registration Token (`utcNow` + `Update`) | harmloses Re-Deployment verändert Session-Host-Bootstrap | Token nur erzeugen/rotieren, wenn neue Hosts registriert oder explizit repariert werden |
| F010 | P0 | Accelerator-Patch | Session-Host-Skript installiert Agent/Bootloader ohne Vorprüfung erneut | unnötige MSI-Läufe/Reboots, schlechte Idempotenz | Zustand/Version/Registration prüfen; Bootstrap idempotent gestalten |
| F011 | P0 | Accelerator-Patch | Session-Host-Skript setzt `NoAutoUpdate = 1` | produktiver Host kann ohne Replacement-/Patchmodell ungepatcht bleiben | Verhalten parametrieren; Default erst nach Architekturentscheidung O009 festlegen |
| F012 | P0 | Accelerator-Patch AD/FSLogix | Domain-Join-Passwort wird in PowerShell-Befehlsstring eingebaut und geloggt | Credential-Leak in Logs/Extension-Kontext | keine Secrets in Kommando-/Logstrings; `Invoke-Expression` mit Secret entfernen |
| F013 | P0 | Accelerator-Patch AD/FSLogix | DSC erlaubt Plaintext Password (`PSDscAllowPlainTextPassword = true`) | Secret kann in MOF/temporären Dateien persistieren | sicheren Ausführungsweg; DSC-Credential-Hardening oder Neuaufbau gemäß O010 |
| F014 | P0 | Accelerator-Patch AD/FSLogix | MOF-/DSC-Cleanup auskommentiert | sensitive Artefakte bleiben auf deallokierter MGMT-VM | MOF, ZIP, Temp und sensitive Logs sicher entfernen |
| F015 | P0 | Accelerator-Patch AD/FSLogix | Storage-Skript deaktiviert Windows Firewall für alle Profile und stellt sie nicht wieder her | wiederverwendete MGMT-VM bleibt ohne Host-Firewall | Firewall nicht global abschalten; falls temporär erforderlich: Zustand sichern und in `finally` wiederherstellen |
| F016 | P0 | Accelerator-Patch AD/FSLogix | Join-Account wird lokaler Administrator auf MGMT-VM und nicht entfernt | unnötige dauerhafte Privilegierung | nur temporär; anschließend entfernen; Least-Privilege-Join-Konto |
| F017 | P0 | Accelerator-Patch AD/FSLogix | DSC `TestScript` betrachtet vorhandenes AD-Computerobjekt als vollständigen Erfolg | Teilfehler nach Domain Join können beim Retry übersprungen werden | Domain-Join-, Share-, RBAC- und ACL-Zustand getrennt prüfen |
| F018 | P0/P1 | Accelerator-Patch AD/FSLogix | AzureFilesDomainJoin kann bei Retry an vorhandener/gesperrter ZIP scheitern (#866) | Wiederholbarkeit unzuverlässig | Lauf-spezifischer Temp-Pfad oder robuster Cleanup/Retry |
| F019 | P0 | Feature-Patch Entra Kerberos | #869: FSLogix-NTFS-Rechte bei EntraIDKerberos zu breit | Cross-User-Profilzugriff möglich | ACL-Modell korrigieren + expliziter Isolationstest |
| F020 | P0/P1 | Architektur/Feature | `EntraID` + FSLogix verwendet Storage Account Key/`cmdkey` | langlebiger Storage-Key auf Session Hosts | Entra-Kerberos-/Identity-basiertes Zielmodell; Key-Fallback nicht MSP-Default |
| F021 | P0/P1 | Feature-Patch Monitoring | AVD Insights DCR sammelt aktuelle Win11-Session-Counter fehlerhaft | unvollständige Telemetrie | #867/#870 prüfen und tatsächliche Daten validieren |
| F022 | P0/P1 | Accelerator-Patch | Pooled Scaling Schedule hart codiert | Kundenzeiten/Force-Logoff ungeeignet | Schedule vollständig parametrieren; O008 entscheidet Default-Power-Modell |
| F023 | P1 | Runtime/Release | PowerShell-Module ohne feste Version zur Laufzeit | identischer Commit verhält sich später anders | Modulversionen pinnen/paketieren und im Release-Manifest führen |
| F024 | P1 | Accelerator-Patch | mehrere Baseline-Ressourcen verwenden Preview-APIs | unnötige Preview-Abhängigkeit | API-Audit; Preview nur wenn benötigte Property keine stabile API hat |
| F025 | P1 | Release Engineering | Marketplace Images häufig `version: latest` | gleicher Release kann anderen OS-Build erzeugen | Approved Image Catalog/Release-Manifest |
| F026 | P1 | Kosten/Accelerator | Workload Key Vault in AzureCloud standardmäßig `Premium` | unnötiger Enterprise-Default | `Standard` als KMU-Default; Premium nur bei HSM-/Key-Anforderung |
| F027 | P1 | Security | Storage Managed Identity erhält relativ breite RG-Rollen | Least-Privilege-Abweichung | Aktionen/Scope reduzieren; Resource-Level-RBAC bevorzugen |
| F028 | P1 | Security/FSLogix | Storage Shared-Key-Zugriff bleibt nutzbar | zusätzliche Angriffsfläche | nach Identity-Migration `allowSharedKeyAccess=false` prüfen/testen |
| F029 | P1 | Accelerator-Patch | FSLogix Defender-Exclusions referenzieren `$FslogixFileShareName`, Parameter heißt `$FSLogixFileShare` | Share-VHD/VHDX-Exclusions wahrscheinlich unwirksam | Variable korrigieren + aktuelle Defender-Empfehlung validieren |
| F030 | P1 | Release Engineering | Bicep und vorcompilierte ARM-JSONs parallel | Fork-Pfade können auseinanderlaufen | Bicep einzige Source of Truth; JSON nur Build-Artefakt |
| F031 | P1 | Supply Chain | einzelne AVM-Unterabhängigkeiten weiterhin `br/public:` | Build benötigt öffentliche Registry | kontrolliertes `bicep restore`, Versionen pinnen, ggf. vendoren |
| F032 | P1 | Accelerator-Patch | mehrere Security-Groups werden als Array angeboten, Logik nutzt nur ersten Eintrag | UI/Parameter versprechen mehr als umgesetzt | Semantik auf eine Gruppe begrenzen oder Multi-Group korrekt implementieren |
| F033 | P0 | MSP-Foundation | Existing-VNet-Pfad ändert AVD-Subnetz nicht; Storage/KV nutzen VNet Rules | ohne Service Endpoint kann Zugriff scheitern | Foundation setzt `Microsoft.Storage` und ggf. `Microsoft.KeyVault` Service Endpoints |
| F034 | P0 | MSP-Foundation | Existing-VNet-Pfad liefert nicht unser explizites KMU-Outbound-Modell | private Subnetze brauchen planbaren Outbound | NAT Gateway in Foundation und relevante Subnetze anbinden |
| F035 | P0 | Preflight/Foundation | SKU-/Zone-/Quota-Verfügbarkeit kann Deployment brechen | vermeidbare Rolloutfehler | VM-SKU, Region, Zone, Quota vor Deployment validieren; PR #848 als Referenz |
| F036 | P2 | Feature-Patch GPU | GPU-DINE-Policy-Identity ohne erforderliche Rolle (#872) | GPU-Remediation scheitert | nur GPU-Profil; Rolle möglichst eng setzen |
| F037 | P2 | Feature-Patch Custom Image | VDOT-Upstreamänderung hat Image-Build gebrochen (PR #851) | Custom-Image-Pfad nicht blind nutzbar | Upstream-Fix + externe VDOT/WDOT-Version pinnen |
| F038 | P2 | Feature-Patch HA | Availability-Zone-/Re-Deployment-Probleme (PR #847) | Resilient-/Zonenprofil kann scheitern | Fix vor Resilient-Release übernehmen und testen |
| F039 | Strategisch | Architektur | `AutomaticSessionHostConfiguration` existiert als Preview (PR #850) | heutiger Token-/Bootstrap-Pfad könnte langfristig ersetzt werden | Release 1.0 `Unmanaged`, MSP-Schicht nicht eng an heutigen Bootstrap koppeln |

---

# 4. Verbindliches Compatibility-/Hardening-Patchset für Release 1.0

Die genaue Projektpriorität steht im Masterplan. Technisch sind vor Release 1.0 insbesondere folgende Bereiche zu behandeln.

## 4.1 Management-VM

- F001 Größe vom Session Host entkoppeln.
- F002 OS-Disk minimieren.
- F003 nach erfolgreichem Lauf deallokieren.
- F004 aktuelles Windows-Server-2022-Offer.
- F012–F016 Security-/Credential-Hardening.

Zielzustand:

```text
kleine Burst-VM
+ Small Disk / Standard_LRS
+ kein Public IP
+ keine dauerhaften Secret-Artefakte
+ Firewall aktiv
+ keine unnötigen Local-Admin-Rechte
+ Normalzustand Deallocated
```

---

## 4.2 Determinismus und Idempotenz

### Naming

`utcNow()` darf nicht persistente Resource Identity bestimmen.

Beispielprinzip:

```bicep
var namingSeed = uniqueString(
  avdWorkloadSubsId,
  deploymentPrefix,
  deploymentEnvironment,
  avdSessionHostLocation
)
```

Zeitstempel dürfen für Logs/Deployment History, nicht für langlebige Resource-Namen verwendet werden.

### Tags

`CreationTimeUTC` darf nicht bei jedem Re-Deployment als Common-Tag neu geschrieben werden.

### Registration Token

Token-Lifecycle von normalen Host-Pool-Updates entkoppeln.

### Agent Bootstrap

Vor Neuinstallation prüfen:

- Agent vorhanden?
- Bootloader vorhanden?
- Version/Zustand gesund?
- Host bereits registriert?

Ein Re-Deployment darf keinen unnötigen MSI-Lauf/Reboot auslösen.

---

## 4.3 Runtime-/Supply-Chain

Microsoft `main` ist keine reproduzierbare Runtime-Quelle eines MSP-Releases.

Ziel ist eine eigene versionierte Runtime-Auslieferung gemäß O007 und `AZURE-DEVOPS.md`.

Mindestens versionieren/hashen:

- `Set-SessionHostConfiguration.ps1`,
- `Manual-DSC-Storage-Scripts.ps1`,
- `DSCStorageScripts.zip`,
- verwendete PowerShell-Module,
- Marketplace-Image-Referenz,
- übernommene Upstream-Fixes.

AVD-Agent-/Bootloader-URLs mindestens auf den Microsoft-Fix aus PR #873 heben.

---

## 4.4 Session-Host Update/Patching

F011 wird erst nach O009 endgültig umgesetzt.

Bis ein klares Modell existiert, darf `NoAutoUpdate=1` nicht unreflektiert MSP-Default sein.

---

## 4.5 ADDS-/FSLogix-Storage-Helper

F012–F018 sind zusammen als ein Security-/Idempotenzproblem zu betrachten.

Sollzustand:

```text
Secret sicher übernehmen
→ Domain-/Storage-Schritt ausführen
→ Share/RBAC/ACL separat validieren
→ fehlende Schritte bei Retry nachholen
→ Temp/MOF/ZIP bereinigen
→ temporäre Rechte entfernen
→ Firewall-Zustand sicher
→ Validation
→ Deallocate
```

O010 entscheidet, ob das bestehende DSC stark gehärtet oder der Bootstrap neu strukturiert wird.

---

## 4.6 Scaling Plan

Hardcodierte Zeiten und Force-Logoff-Werte werden nicht als MSP-Standard übernommen.

Mindestens parametrieren:

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

O008 entscheidet, ob NativeAutoscale der Default wird oder die eigene AutoShutdown-Lösung.

---

# 5. MSP-Foundation-Aufgaben statt Accelerator-Patch

## F033 – Service Endpoints

Bei Existing VNet und ohne Private Endpoint muss unsere Foundation die erforderlichen Service Endpoints konfigurieren.

## F034 – Explicit Outbound

NAT Gateway/Zuordnung ist Aufgabe unserer Foundation.

## F035 – Preflight

SKU-/Quota-/Zone-Prüfung wird vor Deployment automatisiert.

Diese drei Punkte sind bewusst **kein Grund**, den Microsoft-Netzwerkcode invasiv umzubauen.

---

# 6. Feature-abhängige Maßnahmen

## Entra-only + FSLogix

Vor produktiver Aktivierung:

- O002/O003/O004 schließen,
- O007 Runtime-Modell berücksichtigen,
- F019 ACL-Fix,
- F020 kein Storage-Key/`cmdkey` als gewünschter Entra-KMU-Default,
- Cross-User-Isolation explizit testen.

## Monitoring

Vor Aktivierung:

- F021/#870 prüfen,
- Win11 24H2/25H2 reale DCR-Daten verifizieren,
- Retention/Kosten gemäß O005 festlegen.

## Custom Image

Vor Aktivierung:

- F037/PR #851,
- VDOT/WDOT pinnen,
- keine beweglichen `main`-Downloads.

## Availability Zones / Resilient

Vor Aktivierung:

- F038/PR #847,
- Regionen mit 0/1/2/3 Zonen testen,
- Re-Deployment bei geänderter Zonenwahl testen.

## GPU

Vor Aktivierung F036/#872 korrigieren.

---

# 7. API-/Image-/Supply-Chain-Audit P1

Nach den P0-Blockern:

- Preview-API-Audit F024,
- Approved Image Catalog F025,
- Key Vault Standard/Premium F026,
- Storage RBAC Scope F027,
- Shared Key Access F028,
- Defender Variable F029,
- Bicep-only Source of Truth F030,
- `br/public` Pinning F031,
- Security-Group-Semantik F032.

---

# 8. Upstream-Review-Regel

Vor jedem eigenen Fix:

```text
Problem bestätigen
→ Microsoft main prüfen
→ offene Issues prüfen
→ offene PRs prüfen
→ vorhandenen Microsoft-Fix bewerten
→ gegen unsere Zielprofile testen
→ erst dann eigener Patch
```

Upstream-PRs werden nicht blind cherry-picked.

Jedes MSP-Release dokumentiert:

- Microsoft Upstream Commit,
- zusätzlich übernommene noch nicht gemergte Fixes,
- eigenes Patch-Delta,
- Runtime-Artefakt-Hashes.
