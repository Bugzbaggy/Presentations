# A Read-Only SQL MCP, Powered by dbatools

Every production database investigation I've ever done started the same way: cold. An alert fires — "DB Call Failures," a blocking spike, a node that just failed over — and the clock starts. I open a dozen SSMS tabs, remember which server is the primary this week, paste in the same wait-stats query I've pasted a thousand times, and start piecing together what happened. The knowledge of *how* to investigate lived in my head and a folder of `.sql` snippets. None of it was available to the AI tools I'd started leaning on everywhere else.

So I built the bridge: a **read-only Model Context Protocol (MCP) server** that lets an AI assistant — Claude Code, in my case — ask real questions of a live, multi-region SQL Server fleet and get real answers, in plain English, without ever being able to change a thing.

It has since become the first thing I reach for during an incident. It's cracked crashes I would have spent days on. And the part I'm proudest of isn't the power — it's that I can hand it to anyone on the team and know, structurally, that it cannot break production. This is the story of how it works, the incidents it solved, and why "read-only" turned out to be a feature, not a limitation.

## The Problem: Powerful AI, Zero Access to the Thing That Matters

AI assistants are great at reasoning over context you give them. The trouble is that the most important context for a DBA — the live state of production — is exactly the context you're most afraid to expose. The naive options are both bad:

- **Give the AI a real login and hope.** One hallucinated `UPDATE` without a `WHERE`, one `DROP` it thought was a good idea, and you're restoring from backup. No thanks.

- **Copy-paste query results into a chat window by hand.** Safe, but you're the bottleneck. You've replaced a fast tool with a slow human, and the AI only ever sees the slice you happened to paste.

I wanted a third path: the AI runs the diagnostics itself, across every server, but the *only* thing it is physically capable of doing is reading. Not "we told it to be careful." Not "the prompt says read-only." *Structurally incapable of writing.*

## What It Is

The server is a TypeScript MCP server (Node) that exposes the fleet to an AI client as a set of tools. Point your assistant at it, and you can ask things like *"Is anything blocking on the primary right now? Show the head blocker and its SQL,"* or *"Compare wait stats across every region,"* and it chains the right tools to answer.

A few numbers to set the scale:

- **~50 diagnostic tools** — blocking chains, wait stats, missing/unused indexes, index fragmentation, Query Store regressions, top queries, plan-cache pollution, tempdb, latches, AG health (including cross-region distributed AGs), backup status, Agent jobs, CPU/memory history, buffer pool by object, file I/O, loaded modules, a crash-dump reader, the error log, and an on-demand cluster-log generator.

- **A whole-fleet reach** — a dozen instances across four regions, each addressable by name, plus a synthetic `<region>-primary` alias that resolves to whichever node is the current Availability Group primary at query time (so a failover never leaves you pointed at a stale server).

- **A 40+ check health report** built on the `dbatools` PowerShell module — availability, performance, recoverability, reliability, security, configuration, maintenance, and host/OS — that also runs headless on a schedule and posts a weekly summary.

- **An offline schema knowledge layer** so the AI knows what a column *means* and how coded values decode, instead of guessing.

- **222 automated checks** that validate the tool sets and every read-only guardrail on every build.

## Why It's Safe: Read-Only at Three Independent Layers

This is the part I care about most, so let me be specific. "Read-only" here isn't a promise — it's enforced at three layers that a write would have to defeat *all* of, simultaneously.

Layer
What it enforces

**1. The login**
Every operation runs under the *caller's own* least-privilege SQL login. There is no shared service account. SQL Server enforces that person's permissions and audits every query back to them.

**2. The connection**
`ApplicationIntent=ReadOnly` with `Encrypt=True`. The connection itself declares it's there to read, and routes to readable secondaries where possible.

**3. The query allowlist**
Server-side, before anything touches the database: only `SELECT` / `WITH` / `DECLARE` T-SQL and only `Get-`/`Test-`/`Measure-`/`Find-` `dbatools` cmdlets are accepted. DML, DDL, DCL, `EXEC`, `BACKUP`, `RESTORE`, `DBCC`, `KILL`, `WAITFOR`, dynamic SQL, and batch-smuggling (`SELECT 1; DROP TABLE x`) are all rejected.

On top of those three, a few things I learned to add the hard way:

- **No secret ever lives on disk — and it's *enforced*.** The server refuses to start if it finds a plaintext password in its config. The real credential comes only from the OS credential store (Windows Credential Manager / macOS Keychain / libsecret) or a genuine environment variable, resolved at startup. A password sitting in a config file is a startup error, not a warning.

- **A sensitive-table guardrail.** The fleet has a message-log table with billions of rows and PII in it. An unbounded scan of it — the kind an eager AI might try for a "how many messages today" question — is blocked at the query layer and redirected to the pre-aggregated reporting tables. You physically cannot ask the tool to table-scan the crown jewels.

- **Even the one write path is a dry run by default.** There's an opt-in profile for engineers that adds a single `execute_write` tool — and it is *dry-run by default* (the change is executed, the row count reported, then rolled back). It commits only when a human re-runs it with `confirm:true`, it's bounded by the engineer's own SQL role, and it still hard-blocks DDL, settings changes, principals, backups, and Agent jobs. The default read profiles don't have it at all.

The payoff of this design is organizational, not just technical: I can give the ops team and engineers their own profiles, each scoped to their own login, and let them ask production questions in plain English — without handing anyone a foot-gun. The safety isn't a rule people have to remember. It's the shape of the tool.

## Why It's Powerful: One Move That Beats All the Others

If I could keep only one tool, it would be **fan-out** — run a single diagnostic query across every node in the fleet at once and get the results side by side. Almost every non-trivial investigation resolves into a comparison: *this* server is misbehaving, so what's different about it versus the ones that aren't?

```
-- Inventory every non-Microsoft DLL injected into the SQL Server process,
-- fanned out across the whole fleet. Anything present on the crashing node
-- but not on a healthy peer is a suspect.
SELECT name, file_version, company, base_address
FROM sys.dm_os_loaded_modules
WHERE company NOT LIKE 'Microsoft%' OR company IS NULL
ORDER BY name;
```

A few others earn their place:

- **Post-incident query history.** The server reads a persisted, ~1-minute-cadence snapshot of "what was running" (a stored `sp_WhoIsActive` history). So when an alert says something broke at `09:00:55` and you're looking three hours later, you can still ask *"what was executing on that node at that second, and who was the head blocker?"* That single capability has ended more finger-pointing than anything else I've built.

- **A crash-dump reader.** Point it at a stack dump and it pulls the exception code, the faulting module, and the input buffer — the actual SQL statement that tripped the fault.

- **Live-process verification.** It can read what's actually mapped into the running `sqlservr.exe` right now — not what's on disk. (More on why that distinction saved me below.)

- **A semantic layer.** An offline, bundled corpus of schema documentation — table meanings, column purposes, and enum decodings — so when the AI sees `StatusId = 3` it knows that means "Rejected," instead of guessing.

And a lot of that depth isn't hand-rolled — it's **`dbatools`** under the hood. The brilliant open-source PowerShell module is the engine behind the entire 40+ check health sweep and the host/OS-level checks (power plan, firewall, disk, machine spec) that raw T-SQL can't reach. The server exposes *only* its read-only cmdlets (`Get-`/`Test-`/`Measure-`/`Find-Dba*`), so I got years of battle-tested DBA tooling as an AI-callable surface without writing a fraction of it myself. dbatools does the heavy lifting; the MCP server just makes it safe and conversational.

## Why It's Useful: Four Investigations It Actually Cracked

Power is nothing without judgment. What makes this tool genuinely useful is that it's wired to a specific investigative discipline — and it keeps me honest. Here are four real incidents (details anonymized), and what the tool did in each.

### 1. The third-party driver that crashed the engine — and the "upgrade" that was already done

One morning a region's primary node hard-crashed with a fatal fail-fast exception, crash-looped three times in thirty seconds, and triggered an automatic Availability Group failover — a two-minute outage that spiked to tens of thousands of call failures per minute. The Windows event log named the faulting module: a **third-party the cloud data warehouse ODBC driver** running *in-process* inside SQL Server (it has to, for the linked servers to work), so when the driver faulted, it took the whole engine down with it.

The obvious fix — "upgrade the driver" — was where the AI + MCP earned its keep. We kept upgrading, restarting, repairing the install, and every check came back "no change." Here's why: the newest build was *already on disk* on every node, and had been for weeks. The crash happened only because the crashed node's *process* was still running an old build it had loaded into memory long ago and never restarted to release. Proof came from one query, fanned out:

```
-- What version is LOADED in the live process (authoritative),
-- vs. what's sitting on disk? These can differ for weeks.
SELECT name, file_version   -- the mapped, in-memory build
FROM sys.dm_os_loaded_modules
WHERE name LIKE '%the cloud data warehouse%';
```

The healthy nodes showed the new version loaded and running clean; the crashed one had the old build mapped. So the fix was never "install the driver" — it was a plain SQL Server restart on each node still running a stale in-memory build, so it would reload the version already on disk. Without the ability to read the *loaded* module (not the disk file), we'd have kept reinstalling something that was already there.

### 2. Recurring engine crashes — and the trap of "same symptom, same cause"

Several nodes were throwing access-violation dumps. The lazy conclusion — "they're all crashing, it's all the driver" — was wrong, and the tool proved it. Reading each dump's exception code and faulting module showed *three different signatures*: one was the in-process driver; another was a null-pointer AV in a core engine module; a third was a bad-argument fault in the expression services. Same symptom, three unrelated causes.

Then the input buffers told the real story. Cross-referencing each dump's SPID against the persisted query history revealed the actual triggers: our own once-a-minute monitoring wrapper (an outdated build of a popular free diagnostic proc), and a nightly backup-maintenance job scanning a bloated history table. Neither had anything to do with linked servers. The remediation was mundane and correct — update the vendored scripts, trim the history table — and it only surfaced because the tool let us read the exact statement behind each crash instead of pattern-matching on "AV."

**The discipline baked in:** co-occurrence is not causation. A module being loaded doesn't mean it's guilty. A query that was running doesn't mean it's the cause — if it runs every minute, it'll be present at every crash by base rate. The tool is instructed to demand a plausible mechanism *and* a control-group difference before blaming anything. That single rule has stopped me chasing more wrong hypotheses than I'd like to admit.

### 3. High CPU on a small box — don't add cores, fix the query

A modestly-sized node kept alerting on "processor load too high." The reflex is to throw hardware at it. Instead, the top-queries tool surfaced the culprit in seconds: a parameterless "get-everything" lookup procedure being called constantly, each call full-scanning a table and fanning in on a box with only a handful of vCPUs. The fix wasn't more cores — it was caching and parameterizing that one call path. Cheaper, faster, and it actually addressed the cause. The tool turned "we need a bigger server" into "we need to fix this one proc."

### 4. The alert that self-cleared — and wasn't SQL at all

A "SQL connection error" alarm kept firing on two nodes simultaneously and clearing itself a minute later. Classic wild-goose-chase material. But the persisted query history let me prove the engine was *up and serving queries* at the exact seconds the alert claimed it was unreachable — no dumps, no failover, nothing wrong on the database side. That pointed the investigation where it belonged: the monitoring probe's own network path, not SQL Server. We stopped tuning a database that was never the problem.

## The Bar for a Recommendation

Because the tool is used during real incidents, it holds every recommendation to a standard before it presents one. Any fix has to clear three checks, stated out loud in the answer:

- **Mechanism** — "X causes the failure because *this specific chain of events*."

- **Control-group difference** — "X is present (or different) on the failing node and not on this named healthy peer."

- **Blast radius** — "if this fix is wrong, recovery looks like *this*."

If it can't fill in the first two, it labels the suggestion "hygiene," not a fix — so nobody mistakes a plausible guess for a diagnosis. That honesty is what makes the output trustworthy enough to act on at 2 a.m.

## Where This Goes Next: An MCP-Powered PR Reviewer and Impact Analyzer

Here's the direction I'm most excited about, and where this connects to [the AI migration-script generator I wrote about previously](#). That pipeline turns a schema change into a deployment-ready migration script automatically. But it reasons about the *code* — the repo. It can't see what a change will do to *live production*. The read-only MCP can. Put the two together and the pull request stops being a code review and becomes an **impact analysis grounded in the real database.**

Concretely, when an engineer opens a PR that changes the schema, an agent with the read-only MCP can post a comment that answers questions the diff alone never could:

- **"You're dropping this index — but it's hot."** Check live index-usage stats. An index the code thinks is unused might be serving thousands of seeks a day. Flag it before it's dropped.

- **"This proc you're rewriting has a live regression history."** Pull Query Store for the affected object and show whether the current plan is already unstable — context a code reviewer simply can't see.

- **"That table you're altering has two billion rows."** Read the real row count and partitioning, and warn that the `ALTER` needs `ONLINE = ON` / `RESUMABLE = ON` or it'll lock the table in production.

- **"The table you're deleting is still referenced — live."** Beyond a repo grep, query the actual dependency metadata and module text across every database to show the true blast radius of a drop.

- **"This new index duplicates one that already exists."** Compare against live index definitions so you don't ship redundant, write-penalizing indexes.

Every one of those checks is read-only. None of them can touch the database. But together they turn "looks good to me" into a reviewer who has actually *looked at production* — automatically, on every PR, before anything merges. The same discipline that makes the tool a good incident responder makes it a ruthless, evidence-based PR gate.

## The Tech Stack

Component
Technology

ServerTypeScript on Node (Model Context Protocol server)
Transportsstdio (per-user local launch) and HTTP (shared service)
DatabasesSQL Server 2022, multi-region Availability Groups (incl. distributed AGs)
Health checkPowerShell + `dbatools` (read-only cmdlets only)
SecretsOS credential store or env var — never on disk (enforced at startup)
AI clientsClaude Code, Claude Desktop, VS Code + Copilot, Cursor
Quality bar222 automated checks (Node + PowerShell) over tools, guardrails, and topology
Built withClaude Code (the server, the health check, and the runbooks)

## What I Learned

### Read-only is a feature, not a compromise

I expected "read-only" to feel limiting. It's the opposite. Because the tool *cannot* write, I stopped worrying about who uses it and started using it for everything — during incidents, in reviews, for onboarding. Removing the ability to cause harm removed the friction that kept AI away from production in the first place.

### Enforce the guardrail; don't document it

"Don't put passwords in the config" is a rule people break. "The server won't start if there's a password in the config" is a guarantee. Every place I turned a policy into a structural constraint — the query allowlist, the no-secrets-on-disk check, the sensitive-table block, the dry-run-by-default write tool — the tool got safer *and* easier to trust.

### The loaded module, not the disk file

The driver-crash investigation taught me a lesson I now apply everywhere: for anything injected into a running process, the file on disk is irrelevant until the process restarts. Verify against the live process, always. A tool that can read what's actually mapped into memory is worth ten that read the filesystem.

### Make the AI show its evidence

The most valuable thing I encoded wasn't a query — it was the discipline: find a healthy control, demand a mechanism, distinguish co-occurrence from cause, and label a guess as a guess. An AI that can run 50 diagnostics is impressive. An AI that runs the *right three* and tells you why is useful.

## Closing

The gap between "AI that could help with your database" and "AI you'd actually let near production" isn't model capability. It's trust — and trust comes from structure. Per-user identity so every action is owned and audited. Three independent read-only layers so a write has to beat all of them. Secrets that can't touch disk. A write path that's a dry run until a human says otherwise. Build those in, and you get something rare: a production tool powerful enough to crack a failover-inducing crash, and safe enough to hand to the whole team.

It started as a way to stop investigating cold. It's turned into the most-used, least-scary tool I own — and, next, an automated reviewer that reads production so my pull requests don't have to guess.

*The tool described here is a read-only MCP server for a multi-region SQL Server 2022 fleet, built with Claude Code, validated by 222 automated checks, and used daily for incident response, health reporting, and — increasingly — live impact analysis on pull requests. If you're building something similar, or wondering how to give AI safe access to production data, feel free to reach out.*

**— Anonymous**
