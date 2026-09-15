# In-Place Upgrading a SQL Server Always On Cluster to Windows Server 2025: Field Notes

We recently ran an in-place operating-system upgrade of a production, two-node **SQL Server Always On Availability Group** from Windows Server 2022 to Windows Server 2025 — with zero planned downtime and **SQL Server left on 2022**. It mostly went to plan. The parts that *didn't* are the reason this post exists. Everything below is anonymized, but the sequence, the commands, and the failures are exactly what we hit.

## The setup

- Two nodes — call them `SQLNODE01` and `SQLNODE02` — running SQL Server 2022 Enterprise on cloud VMs. Our production fleet spans **AWS and Google Cloud (GCP)**, so the tooling handles both. Nothing here is specific to either — the same sequence applies on Azure or an on-prem hypervisor; only the driver and agent names change.

- A single local Availability Group, `ag-prod01`, with **synchronous-commit / automatic failover** and a **file-share witness**.

- Ten databases in the AG. The largest and busiest by far is a high-volume transactional database we'll call `CoreDB`; the rest are settings, reporting, billing, audit, archive, catalog, session and integration databases.

- A file-share witness for quorum, and an in-house PowerShell module (we'll call it **SqlAgOps**) that scripts the drain / suspend / failover / resume choreography and opens a maintenance window in our alerting tool.

## Why in-place, and how the rolling model works

We chose an **in-place** upgrade over build-and-migrate because the databases are large and the app's connection story (a single AG listener) doesn't change. Windows Server supports a **Cluster OS Rolling Upgrade**: you upgrade one node at a time while the cluster runs in a temporary "mixed-OS" mode, then flip a switch at the very end.

The golden rule of the whole exercise: **you never upgrade the primary.** You upgrade the secondary, fail the primary over onto the freshly-upgraded node, then upgrade the old primary. Concretely, per node:

- Bring the still-on-WS2022 secondary fully patched.

- Update the cloud **network and storage drivers** (this one is not optional — more below).

- Run the WS2025 in-place upgrade on the secondary.

- Fail the AG over onto it, and repeat on the other node.

- Only when both nodes are on WS2025 and soaked: raise the cluster functional level (the point of no return).

### The ordering that matters most: suspend data movement before you drain

This one cost us a production incident, so it earns its own heading. Our first version of the automation did the intuitive thing: **drain the cluster node first, then suspend AG data movement.** That ordering is backwards, and it produced a two-to-three minute burst of failed database calls on the *primary* — while we were only touching the *secondary*.

The mechanism: in a synchronous-commit AG, the primary does not acknowledge a commit until the secondary has hardened the log record. If you pause/drain the node while data movement is still **active and synchronous**, the primary is still trying to synchronously commit to a replica that is being pulled out from under it — so writes stall on the primary. Suspending data movement *first* detaches the primary from that replica, and the drain that follows is invisible to the write path.

The correct per-node order:

- **Suspend AG data movement** on the node you're about to work on (`ALTER DATABASE [db] SET HADR SUSPEND` for each AG database). This decouples the primary.

- **Move the cluster core group** off the node, if it currently owns it.

- **Drain / pause the cluster node** (`Suspend-ClusterNode -Drain -Wait`).

- Patch, upgrade, reboot — the node is now fully decoupled.

- On the way back: resume the cluster node, *then* resume data movement, then verify every database returns to `SYNCHRONIZED`.

**Check this before you rely on suspend-first:** suspending is only safe if the AG doesn't *require* that replica to commit. Confirm `required_synchronized_secondaries_to_commit` is `0` — or that enough *other* synchronous secondaries remain to satisfy it. Otherwise suspending your only synchronous secondary will **block** the primary's commits instead of freeing them, which is the opposite of what you want:

`SELECT name, required_synchronized_secondaries_to_commit FROM sys.availability_groups;`

## Pre-flight that actually matters

Skip the generic "take backups" advice — obviously take backups. Here's the list that is specific to this kind of upgrade and that we wish we'd weighted more heavily going in.

### 1. Know which PowerShell you're in

The cluster module (`FailoverClusters`) is a Desktop-edition module. In PowerShell 7 it only loads through the Windows-PowerShell compatibility session, which hands you **deserialized** cluster objects — nested properties like the witness resource name come back empty. Our tooling is written defensively around that, but for one-shot cluster operations (especially the final functional-level bump) run them in native **Windows PowerShell 5.1**. You'll see this warning constantly and it's benign:

```
WARNING: Module FailoverClusters is loaded in Windows PowerShell using WinPSCompatSession
remoting session; please note that all input and output of commands from this module will
be deserialized objects.
```

### 2. Update the platform drivers BEFORE the OS upgrade — this is a boot-safety gate

Your VM's **network and storage drivers** are what let the machine see its NIC and its disks. Boot a brand-new OS on stale ones and you can get a node with **no network or no boot disk** — and on a cluster node that means it never rejoins and you're recovering from a snapshot. Refresh them *while the node is drained*, before you run Setup, and treat a failure here as a **hard stop**, not a warning.

This applies on every platform — only the component names change:

  PlatformNetwork / storage driversGuest agent
  **AWS EC2**ENA (network), NVMe (storage)SSM Agent, EC2Launch v2
  **Azure**Accelerated-networking NIC driver (Mellanox / MANA), StorVSC storageAzure VM Guest Agent (WaAppAgent)
  **Google Cloud**gVNIC (network), virtio-SCSI / NVMe (storage)Google guest environment (delivered via GooGet)
  **VMware / on-prem**VMXNET3 (network), PVSCSI (storage)VMware Tools

Because our fleet spans two clouds, the automation **detects the platform at run time and dispatches**: on AWS it version-checks and installs the signed vendor driver and agent packages; on Google Cloud it goes through `GooGet` against the guest-environment repository. The operator runs one cloud-neutral command and never has to know which cloud a given node is on. If you run more than one platform, this is worth building — it keeps a single runbook valid everywhere, instead of one procedure per cloud that drifts apart.

Whatever the platform, the shape is identical: **version-check the installed driver against the current published package, install it while the node is drained, and gate the OS upgrade on success.** Two refinements worth building in: make the check *version-aware* so an already-current node is a no-op and doesn't earn a pointless reboot, and make it *offline-capable* (install from a pre-staged internal copy) so a firewalled production host isn't depending on vendor endpoints mid-window.

### 3. Budget disk and time for a very large cumulative update

The monthly WS2022 cumulative update we needed was **~26 GB**. That is not a typo, and it will fail on a tight OS drive. Check free space and grow the volume first; a 26 GB download also takes real wall-clock time, so pre-download it while the node is still serving (downloading installs nothing and is completely safe on a live secondary).

### 4. Pin your Windows Update channel so a SQL CU doesn't sneak in

We deliberately kept SQL Server on 2022 with no CU change during the OS work. But our nodes had **Microsoft Update** opted in, which offers SQL Server cumulative updates alongside OS patches. If your patch step installs from the default service with "accept all," you can silently pull a SQL CU you didn't want. The clean fix is to pin the **Windows Update** channel (`-WindowsUpdate`), which serves OS content only — SQL CUs arrive via Microsoft Update, so pinning the WU channel excludes them by construction.

## The rolling upgrade, step by step

### Validate, then fail over

Before draining a node, we validate that a planned, no-data-loss failover is actually possible — witness/quorum healthy, all databases synchronized within thresholds, schema options matched. Our module runs this as a dry run first (note it must run *on the current primary*; run it on a secondary and it correctly refuses):

```
PS C:\> Invoke-AgPlannedFailover        # validate only
=========================================================
 SqlAgOps v2.5  |  Mode: VALIDATE
 Local node: SQLNODE01
=========================================================

[1/5] Detecting Availability Group on local node...
 -> PASS: Primary for AG 'ag-prod01'.

[2/5] Selecting failover target replica...
 -> PASS: Target replica is 'SQLNODE02'.

[3/5] Verifying WSFC quorum...
    WARN: Could not read the quorum witness resource name (cluster object may be
          deserialized); skipped the witness SMB reachability check.
 -> PASS: 2 node(s) Up, witness configured, votes 3/3.

[4/5] Comparing per-database DB_CHAINING between replicas...
 -> PASS: DB_CHAINING matches on all 10 AG database(s).

[5/5] Checking per-database sync state and queue depth on target...

Database      SyncState    LogSendQueueKB RedoQueueKB
--------      ---------    -------------- -----------
TicketsDB     SYNCHRONIZED             60           0
AuditDB       SYNCHRONIZED             60           0
ArchiveDB     SYNCHRONIZED              0           0
IntegrationDB SYNCHRONIZED             60           0
BillingDB     SYNCHRONIZED              0         116
SettingsDB    SYNCHRONIZED             60           0
CoreDB        SYNCHRONIZED             60         460
ReportingDB   SYNCHRONIZED             60           0
SessionDB     SYNCHRONIZED              0          12
CatalogDB     SYNCHRONIZED             60           0

 -> PASS: All 10 database(s) synchronized within thresholds.

=========================================================
 ALL VALIDATIONS PASSED
=========================================================
Validation-only run (no -Failover supplied). Returning.
```

Then the real thing — a planned, no-data-loss failover that prompts before it moves the primary and opens a maintenance window as it goes:

```
PS C:\> Invoke-AgPlannedFailover -Failover
...
[6/8] Opening maintenance window...
 -> Success: Maintenance window created (ID: ********).
[7/8] Issuing planned failover (no data loss) on 'SQLNODE02'...

Name                 PrimaryReplicaServerName
----                 ------------------------
ag-prod01            SQLNODE02
 -> Success: Failover issued.

[8/8] Waiting 10 seconds for AG state to settle...
SQLNODE01  Secondary
SQLNODE02  Primary

=========================================================
 FAILOVER COMPLETE.
   New Primary:   SQLNODE02
   New Secondary: SQLNODE01
=========================================================
```

### Run Setup with real switches

Windows Setup for an in-place upgrade takes a small set of switches — and a few that people copy off the internet don't exist. What actually works:

```
REM Compatibility pre-check only, changes nothing:
setup.exe /auto Upgrade /compat ScanOnly

REM The upgrade itself (keeps apps + data; EULA is implicit in /auto /quiet):
setup.exe /auto Upgrade /quiet /compat IgnoreWarning /dynamicupdate Disable
```

There is no `/eula` or `/migchoice` switch — drop them. If you'd rather watch it, run `setup.exe` interactively and choose "Keep personal files and apps." `/dynamicupdate Disable` keeps Setup from pulling extra content mid-run, which makes the run predictable and offline-safe. Setup returns to the prompt immediately and reboots the node several times; the node stays a **paused** cluster member throughout — do not evict it.

### Resume and let it catch up

After the node returns on WS2025, resume cluster membership and AG data movement. Expect the busy databases to come back `SYNCHRONIZING` (catching up) before they reach `SYNCHRONIZED`:

```
PS C:\> Resume-AgSecondaryPatching
=========================================================
 SqlAgOps v2.1
 Starting Post-Update Resume for Secondary Node: SQLNODE01
=========================================================

[1/5] Resuming Windows Cluster node...
 -> Success: Node is active in the cluster again.

[2/5] Waiting for cluster node to reach stable State='Up'...
 -> Node 'SQLNODE01' stable (State=Up, NodeWeight=1).

[3/5] Resuming SQL Server AG Data Movement locally...
 -> Success: Resumed data movement for TicketsDB
 -> Success: Resumed data movement for AuditDB
 ... (all 10 databases) ...

[4/5] Verifying Database Synchronization State...
 -> CatalogDB: SYNCHRONIZED
 -> AuditDB: SYNCHRONIZED
 -> BillingDB: SYNCHRONIZING
 -> CoreDB: SYNCHRONIZING
 -> ReportingDB: SYNCHRONIZING
 ...

=========================================================
 RESUME COMPLETE: Node is healthy and catching up.
=========================================================
```

## Eight things that bit us (and the fixes)

### 1. We drained the cluster node before suspending data movement — and the primary paid for it

The first production run of our automation drained the secondary's cluster node and **then** suspended AG data movement. Monitoring lit up with a two-to-three minute burst of failed database calls — on the **primary**, a node we hadn't touched at all. Because data movement was still active and synchronous, the primary was still waiting on a replica that was being drained out from under it, and commits stalled until the suspend finally landed.

What made it easy to miss in review: both steps were in the script, both "worked," and the node came back healthy. Only the *order* was wrong, and the damage landed on a different node than the one under maintenance — so it looked like an unrelated blip.

**Fix:** reverse it — **suspend data movement first**, then move the core cluster group, then drain. (Full sequence and the `required_synchronized_secondaries_to_commit` precondition are in the ordering section above.) We fixed it in the tooling rather than the runbook: the suspend is now unconditionally the first step of the prep function, so no one can reorder it by editing a document. Sequencing you actually depend on belongs in code, not in a numbered list a human follows at 2am.

### 2. The OS cumulative update quietly didn't install

Our combined "patch + drivers" step refreshed the cloud drivers first, which set a *pending reboot*, and the subsequent "install Windows Update and auto-reboot" call then rebooted for that pending state — sometimes **before the 26 GB OS CU was applied**. On top of that, the update call used the box's default service, which (after we'd removed the Microsoft Update opt-in) had nothing to install. Net result: drivers updated, machine rebooted, OS CU never landed — and the version number looked almost right, so it was easy to miss.

**Fix:** reorder to *install the OS update first* (pinned to the Windows Update channel, with reboot suppressed), *then* refresh drivers, *then* reboot exactly once. Verify with the receipt, not the eyeball: `Get-HotFix -Id KB5122882` and the build revision (`UBR`), and re-scan Windows Update to confirm the CU is no longer offered.

### 3. SQL Server didn't auto-start after the WS2025 upgrade

On **both** nodes, the in-place upgrade left the `MSSQLSERVER` service stopped, so the resume step failed at its AG stage with "could not open a connection to SQL Server," and from the partner the replica simply **vanished from the AG**. Don't assume the engine came up.

**Fix:** right after the upgrade, before resuming: `Get-Service MSSQLSERVER` → if stopped, set it back to Automatic and start it, then confirm TCP/IP is still enabled (an in-place upgrade can reset SQL network protocols):

```
Set-Service MSSQLSERVER -StartupType Automatic
Start-Service MSSQLSERVER
Get-DbaNetworkConfiguration -SqlInstance SQLNODE01   # TCP/IP enabled, port 1433
```

### 4. A background job stalled the failover and left databases un-synchronizing

This was the ugly one. A weekly statistics job was **mid-run** on `CoreDB` when we failed over. Its in-flight write transaction became a blocker for the AG state change: the AG repeatedly killed it (you'll see `ABORT_AFTER_WAIT = BLOCKERS` in the log) and looped on a rollback stuck at 0%, so four of the busiest databases sat in **NOT SYNCHRONIZING** on the new secondary while the primary served happily. No data loss — but no redundancy on those four, either.

**Fix:** quiesce heavy jobs on the AG databases *before* any failover. A guard step at the start of a job only checks "am I primary?" at launch — it does nothing if a failover happens *during* the run. We added a pre-flight gate to our tooling that aborts the maintenance if it finds a `KILLED/ROLLBACK` session, a sustained block, or a long-running *write* transaction on the node — and prints the offending session and job so you can stop it first. If you're caught out mid-incident, the recovery is to stop/disable the owning job (once the node is a secondary its own guard then holds it off) and let the databases re-join.

### 5. Watch the SQL build across replicas — direction matters

Our two nodes were one cumulative update apart (one on CU24, one on CU25). An AG tolerates a **secondary at an equal-or-higher** build than the primary; the *reverse* is not supported and the lower-build secondary can fail to redo the primary's log. When we failed onto the higher-build node, the lower-build node became a secondary *behind* the primary — the unsupported direction — which contributed to databases refusing to synchronize.

**Fix:** get both replicas onto the *same* SQL build before you're done. Patch the secondary (fail over first if the node you need to patch is currently primary), then fail back. And confirm the actual build with `SERVERPROPERTY('ProductVersion')` / `('ProductUpdateReference')` — don't assume.

### 6. A .NET assembly conflict blocked the module from loading

Late in the window, importing our module failed with:

```
Could not load file or assembly 'Microsoft.Data.SqlClient, Version=6.0.0.0 ...'.
Assembly with same name is already loaded
```

Cause: two SQL toolsets (dbatools and the `SqlServer` module) both ship `Microsoft.Data.SqlClient`, and every `Invoke-Sqlcmd` we'd run earlier had already loaded one version into the session. You can't unload a loaded assembly, so the session was poisoned.

**Fix:** open a **fresh** PowerShell session and import your AG module *first*, before any `Invoke-Sqlcmd` / `SqlServer` command. Don't mix the two in a session where `SqlServer` loaded first. (And if the operation you need is just resuming AG data movement, you can always do it directly with a small `ALTER DATABASE ... SET HADR RESUME` loop — no module required.)

### 7. The secret store had a forgotten password

Our tooling reads a maintenance-window API token from a DPAPI-protected SecretStore vault. On these nodes the vault had been configured with a master password nobody had, and reconfiguring it for unattended use demanded that password. The recovery is a reset — and the trap is that a session which has already loaded the vault caches the old config, so an in-session reset silently fails.

**Fix:** reset from a **brand-new** session that hasn't touched the vault — wipe the local store files, confirm the folder is empty, then re-run your token setup so it initializes fresh with no password.

### 8. Maintenance-window API calls are best-effort, not blocking

Small thing, but worth designing for: our close-the-window call came back `400 Bad Request` and the resume script (correctly) treated it as non-blocking and carried on:

```
[5/5] Closing maintenance window...
 -> WARNING: Failed to close maintenance window (non-blocking): 400 (Bad Request).
```

Never let a cosmetic alerting-API failure abort a database maintenance step. Log it and move on.

## Automatic failover during maintenance: leave it on

A fair question mid-upgrade: should you switch the AG to **manual** failover while you work? For this rolling model, no. When you're patching the secondary, it's suspended/drained and therefore **not a valid failover target** — so automatic and manual behave identically, and an ill-timed automatic failover onto the half-patched node simply can't happen. And the moment you resume and the secondary re-synchronizes, you *want* automatic back on so HA is restored. The real cost of manual is forgetting to turn it back — silently running with HA disabled.

**The honest risk isn't the failover mode.** While one node is mid-upgrade, the other is a temporary single point of failure — there is simply no synchronized secondary to fail to. Nothing about automatic-vs-manual changes that. What mitigates it: verified backups, a tight disruptive window (especially the reboots, when a simultaneous primary loss can drop cluster quorum), a go/no-go that the surviving node is healthy before you start, and — structurally — a third replica so you're never down to one copy.

## Finalizing: the point of no return

Once **both** nodes are on WS2025, healthy, and soaked, you raise the cluster functional level. After this, no down-level node can rejoin — so make it deliberate: pre-check that every node is `Up` and on the new build, dry-run first, then commit. In native Windows PowerShell 5.1:

```
Get-ClusterNode | Select Name, State                 # all Up
Get-Cluster | Select ClusterFunctionalLevel          # 11 (WS2022) now
Update-ClusterFunctionalLevel -WhatIf                # dry run, changes nothing
Update-ClusterFunctionalLevel -Force -Verbose        # commit
Get-Cluster | Select ClusterFunctionalLevel          # 12 (WS2025)
```

It's non-disruptive — no reboot, no SQL/AG interruption; it just raises the WSFC metadata level so the cluster can use WS2025 features. Functional levels, for reference: WS2016 = 9, WS2019 = 10, WS2022 = 11, WS2025 = 12.

## The condensed checklist

- **Never upgrade the primary.** Secondary → fail over → old primary. Validate a no-data-loss failover before every drain.

- **Suspend AG data movement before you drain the node** — never the reverse. Draining while the replica is still synchronous stalls commits on the *primary*. Confirm `required_synchronized_secondaries_to_commit = 0` before relying on it.

- **Refresh cloud NIC/disk drivers while drained, before Setup.** Stale drivers = a WS2025 node with no network or no disk.

- **Install the OS update *first*, pinned to the Windows Update channel, then drivers, then one reboot.** Verify with `Get-HotFix` and the build UBR — not the version string.

- **After the upgrade, confirm SQL Server actually started** and TCP/IP is enabled before you resume.

- **Quiesce heavy jobs on the AG databases before failover.** A long-running write transaction will stall the AG state change and strand databases as NOT SYNCHRONIZING.

- **Keep both replicas on the same SQL build.** A secondary behind the primary is unsupported and won't reliably redo.

- **Import your AG module first in a clean session** to avoid the dbatools / SqlServer `Microsoft.Data.SqlClient` assembly clash.

- **Leave automatic failover on.** Mitigate the single-node exposure with backups, a tight window, and ideally a third replica — not by flipping to manual.

- **Raise the cluster functional level last**, from native PowerShell 5.1, dry-run then commit. It's irreversible.

An in-place OS upgrade of a live AG cluster is very doable with zero data loss — the failovers and Setup are the easy 80%. The other 20% is the boring operational plumbing: driver order, patch channel, service auto-start, background jobs, and build parity. Get those right and the "hard" part takes care of itself.
