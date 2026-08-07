# MSP AVD Accelerator – Validierung, Guardrails und Release

> **Rolle dieses Dokuments:** Fachliche Detailquelle für Preflight, Tests, Security, Kosten und Release-DoD  
> **Projektstatus/Priorität:** [`../../UMSETZUNGSPLAN.md`](../../UMSETZUNGSPLAN.md)

---

# 1. Preflight

Vor jedem Deployment sollen technische Voraussetzungen soweit möglich automatisiert geprüft werden.

Mindestens:

- [ ] `Microsoft.DesktopVirtualization` registriert.
- [ ] `Microsoft.Compute` registriert.
- [ ] `Microsoft.Network` registriert.
- [ ] `Microsoft.Storage` registriert.
- [ ] weitere tatsächlich verwendete Provider registriert.
- [ ] VM SKU in Zielregion vorhanden.
- [ ] SKU in gewünschter Availability Zone vorhanden, falls AZ aktiviert.
- [ ] ausreichende vCPU-/Family-Quota.
- [ ] gewünschte Marketplace Images vorhanden.
- [ ] Management-VM Small-Disk-Image vorhanden.
- [ ] AVD Enterprise Application Object ID ermittelbar.
- [ ] Subnetz-/RBAC-Rechte vorhanden.
- [ ] Service Endpoints vorhanden, wenn Existing VNet + VNet Rules/Public Endpoint.
- [ ] NAT/Outbound vorhanden.
- [ ] erforderliche AVD-/Microsoft-/Runtime-Endpunkte erreichbar.
- [ ] Identity-Modell eindeutig gewählt.
- [ ] AD/DNS Health nur bei ADDS.
- [ ] Join-Berechtigungen nur bei ADDS.
- [ ] AVD-Lizenzierung dokumentiert.

---

# 2. Testmatrix

| ID | Szenario | Release-Relevanz |
|---|---|---|
| T01 | Entra ID, 1 Host, kein DC, ohne FSLogix | Release 1.0 |
| T02 | Entra ID, 1 Host, kein klassischer DC, FSLogix/Entra Kerberos | abhängig von O002 |
| T03 | NewForest, 1 DC, 1 Host, FSLogix | Release 1.0 wenn `adds-small` enthalten |
| T04 | Existing/Migrated AD DS, 1 Host, FSLogix | Migrationspfad |
| T05 | AD DS, 2 Hosts | Resilient/P2 |
| T06 | AD DS, AVD + APP + SQL + Dependency Automation | App/SQL-Profil |
| T07 | identisches Re-Deployment / What-If ohne Resource Drift | Release 1.0 |
| T08 | Session Host ersetzen / Profil bleibt erhalten | bei FSLogix/Replace-Modell |
| T09 | MGMT-VM Start → Storage-Wartung → Cleanup → Deallocate | ADDS/FSLogix |
| T10 | MGMT-VM nach Wartung: Firewall aktiv, Join-User kein Local Admin, keine Plaintext-MOFs | ADDS/FSLogix |
| T11 | Session-Host Re-Deployment ohne unnötige Agent-Neuinstallation/Reboot | Release 1.0 |
| T12 | FSLogix: User A kann Profil von User B nicht öffnen/mounten | FSLogix Pflicht |
| T13 | Existing VNet + Service Endpoint + Azure Files/KV erreichbar | CostOptimized-Netzprofil |
| T14 | NAT/Explicit Outbound funktionsfähig | Release 1.0 |
| T15 | Native Autoscale 1 Host + Start VM on Connect | O008-PoC/ggf. Release |
| T16 | Custom AutoShutdown 1 Host | O008-PoC/Fallback |
| T17 | Monitoring Win11 24H2/25H2: DCR/Session Counter vorhanden | wenn Monitoring Default |

Für jedes relevante Szenario:

```text
Bicep Restore/Build
→ Validation
→ What-If
→ Deployment
→ Login/Funktion
→ Security-Test
→ Re-Deployment
→ Kostenprüfung
```

---

# 3. Idempotenz-/Re-Deployment-Tests

Ein produktionsfähiges IaC-Deployment muss wiederholbar sein.

Explizit prüfen:

- gleicher Parametersatz erzeugt keine neuen Storage Accounts/Key Vaults wegen Zeitstempel,
- `CreationTimeUTC` erzeugt keinen unnötigen globalen Tag-Drift,
- Host-Pool-Registration-Token rotiert nicht ohne Grund,
- Agent/Bootloader werden nicht unnötig neu installiert,
- kein unnötiger Session-Host-Reboot,
- Storage Domain Join kann nach Teilfehler sauber fortgesetzt werden,
- ACLs/RBAC werden bei Retry vervollständigt,
- Runtime Temp-/ZIP-Dateien blockieren keinen zweiten Lauf,
- Management-VM endet wieder `Deallocated`.

---

# 4. Security-Guardrails

Verbindliche Zielregeln:

- [ ] keine Secrets in Git/Azure Repos.
- [ ] sichere Secret-Verarbeitung über WIF/Key Vault/kurzlebige Tokens.
- [ ] keine Public IP auf AVD Session Hosts.
- [ ] kein offenes Inbound RDP als Standard.
- [ ] Trusted Launch / Secure Boot / vTPM beibehalten, sofern Ziel-SKU/Image kompatibel.
- [ ] Least-Privilege RBAC.
- [ ] Domain-Join-Konto nicht pauschal Domain Admin.
- [ ] kein Passwort in Extension-/PowerShell-/DSC-Logs.
- [ ] keine persistenten Plaintext-MOFs mit Domain-Credentials.
- [ ] Management-VM Host-Firewall nach Storage-Konfiguration aktiv.
- [ ] temporäre lokale Admin-Berechtigungen werden entfernt.
- [ ] keine langlebigen GitHub-/Azure-DevOps-PATs auf Ziel-VMs.
- [ ] SAS-haltige Runtime-URLs nur in geschützten Settings.
- [ ] FSLogix Cross-User-Isolation explizit testen.
- [ ] Storage Shared-Key-Zugriff bei identitätsbasiertem Betrieb nach technischer Validierung deaktivieren.
- [ ] Backup/Restore für DC dokumentieren, wenn AD DS eingesetzt wird.

---

# 5. Kosten-Guardrails

## Standardmäßig vermeiden

- Azure Firewall,
- DC ohne technische Notwendigkeit,
- zwei DCs ohne HA-Anforderung,
- zwei Session Hosts ohne Bedarf,
- dauerhaft laufende Management-VM,
- unnötige Premium-Disks,
- unnötige Private Endpoints,
- unnötige Defender-Pläne,
- DDoS-Plan ohne Bedarf,
- Hub-Spoke ohne Mehrwert,
- Premium Key Vault für reinen Secret-Store ohne HSM-Anforderung.

## Standardmäßig bevorzugen

- Entra ID Only ohne DC, wenn technisch möglich,
- ein Session Host,
- bei erforderlichem AD DS ein DC im CostOptimized-Profil,
- Single VNet,
- NAT Gateway,
- kleine deallokierte Management-VM,
- Standard_LRS für die Management-VM,
- parametrierbare VM-Größen,
- Feature-Kosten transparent je Profil.

Vor Release sollen Kostenprofile für mindestens folgende Fälle dokumentiert sein:

```text
entra-small
adds-small
```

Fixkosten und laufzeitabhängige Kosten getrennt ausweisen.

---

# 6. Feature-spezifische Release-Gates

## FSLogix

Vor Freigabe:

- Authentication erfolgreich,
- RBAC korrekt,
- NTFS ACL korrekt,
- Cross-User-Zugriff verweigert,
- Rebuild/Replace-Profilpersistenz getestet,
- Storage-Tier/Netzwerkmodell gemäß Architekturentscheidung.

## Monitoring

Wenn aktiviert:

- DCR tatsächlich deployed,
- Log Analytics erhält Daten,
- Session-/Capacity-Counter auf Ziel-Windows-Version verifiziert,
- Retention/Kosten dokumentiert.

## ADDS

Vor Freigabe:

- DNS/Domain Join stabil,
- Backup vorhanden,
- Restore-Prozess dokumentiert,
- Helper-VM credential-sicher,
- Storage-Join/ACL Retry getestet.

## APP/SQL

Vor Freigabe:

- Startreihenfolge SQL → APP,
- Stoppreihenfolge APP → SQL,
- bereits korrekte Power States werden übersprungen,
- Timeout/Fehlerhandling,
- kein Konflikt mit AVD Power Management.

---

# 7. Release-Manifest

Jedes freigegebene MSP-Release soll mindestens dokumentieren:

```text
MSP Release Version
Git Commit SHA
Microsoft Upstream Commit SHA
zusätzlich übernommene Upstream PR/Fixes
Bicep Version
AVM-/Registry-Versionen
Runtime-Artefakt-Hashes
PowerShell-Modulversionen
Marketplace Image Version/Build
getestete Azure Region(en)
getestete Profile
bekannte Einschränkungen
```

---

# 8. Definition of Done – Release 1.0

Release 1.0 ist erst erreicht, wenn die für den freigegebenen Profilumfang relevanten Bedingungen erfüllt sind:

- [ ] alle Architekturentscheidungen O001–O010 geschlossen.
- [ ] Projekt vollständig auf Azure DevOps arbeitsfähig.
- [ ] Azure DevOps Pipeline nutzt Workload Identity Federation.
- [ ] Approval Gate für produktive Deployments vorhanden.
- [ ] Accelerator mit dokumentiertem Compatibility-/Hardening-Patchset integriert.
- [ ] Foundation reproduzierbar/idempotent ohne DC deploybar.
- [ ] Entra-only-Profil funktioniert.
- [ ] ADDS-Profil funktioniert, sofern Teil von Release 1.0.
- [ ] Existing-VNet Service Endpoints/NAT korrekt.
- [ ] Management-VM klein dimensioniert und kostenminimal.
- [ ] Management-VM aktuelles Windows-Server-2022-Offer.
- [ ] Management-VM nach Lauf deallokiert.
- [ ] Management-VM Firewall aktiv.
- [ ] keine unnötige dauerhafte Local-Admin-Rolle für Join-Konto.
- [ ] keine Plaintext-Domain-Credentials in Logs/MOF/Temp.
- [ ] persistente Resource-Namen deterministisch.
- [ ] Re-Deployment ohne unnötigen Tag-/Resource-Drift.
- [ ] Runtime-Artefakte versionsfest und privat auslieferbar.
- [ ] Legacy-Agent-/Bootloader-URLs ersetzt.
- [ ] Session-Host-Bootstrap idempotent.
- [ ] Registration-Token-Lifecycle kontrolliert.
- [ ] Session-Host Patch-/Update-Modell definiert.
- [ ] Storage-Konfiguration retriesicher/idempotent.
- [ ] FSLogix-Isolation getestet, falls FSLogix Teil des Profils.
- [ ] Start VM on Connect funktioniert.
- [ ] Power-Management-Default gemäß O008 festgelegt und getestet.
- [ ] Monitoring funktioniert, falls als Default aktiviert.
- [ ] Preflight/Post-Deployment-Validation vorhanden.
- [ ] Kostenmodell für die freigegebenen Standardprofile dokumentiert.

---

# 9. Release-Freigabeprinzip

```text
Pull Request
→ Validation
→ Merge
→ kein automatischer Kundendeploy
→ Release/Deployment manuell auswählen
→ Preflight
→ What-If
→ Approval
→ Deploy
→ Post-Deployment Tests
→ Release-/Kundenprotokoll
```

Ein fehlgeschlagener Security-/Idempotenztest blockiert die produktive Freigabe des betroffenen Profils.
