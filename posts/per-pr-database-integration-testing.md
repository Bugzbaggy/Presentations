# Every Pull Request Gets Its Own SQL Server

Database changes are reviewed differently from application changes, and worse.
An application PR gets compiled, tested, and exercised before a human looks at
it. A database PR historically gets read.

This is how we closed that gap: **every pull request stands up a brand-new,
disposable SQL Server 2022**, publishes the branch's schema into it, and
reports back on the PR — without any shared test database, and without any
step that can block a correct change for the wrong reason.

## The flow

```
1. The branch's DB projects compile to DACPACs
2. A fresh, disposable SQL Server 2022 container starts
3. The schema is published into it and integrity-checked   ← the gate
4. Auto-generated tests run for each changed object        ← report-only
5. Curated hand-written tests run                          ← report-only
6. A ready-to-finish test starter is generated per object
7. A plain-English summary comment is posted on the PR
```

Nothing is shared between runs. The container is discarded afterwards, so
there is no drifting "test database" that slowly stops resembling production.

## The one decision that made it survivable

**Only step 3 can block the PR.**

That was not the original instinct. The instinct is that if you've gone to the
trouble of generating tests, failures should block — otherwise what's the
point?

The point is trust. An auto-generated test that fails usually means the
*generator* guessed the wrong expected value, not that the change is broken.
If that blocks a correct PR even occasionally, engineers learn to distrust the
whole pipeline, and then they route around it. Report-only tests get read.
Blocking tests that cry wolf get disabled.

So the gate is narrow and unambiguous: **does your schema apply cleanly to an
empty database and pass integrity checks?** If yes, you're through. Everything
else is information for the reviewer.

## What the PR comment says

| Row | Meaning | Blocks? |
|---|---|---|
| Deploys cleanly & passes integrity checks | Schema applies to an empty database with no errors | **Yes** |
| Auto-generated unit tests | Tests a machine wrote and ran | No |
| Code exercised by those tests | Line coverage of changed procedures | No |
| Interface contract | Changed procs still take the same params, return the same columns | No — Tier 1 |
| Behaviour | Changed deterministic functions still return the same output | No — Tier 2 |

Tier 1 and Tier 2 are the rows reviewers actually read. A contract change
shows up as a diff in the comment, which is far easier to notice than in a
600-line `.sql` file.

## The blocker nobody warns you about: encrypted config

If your schema uses symmetric keys — API keys, channel credentials, anything
wrapped in `EncryptByKey` — a fresh container **cannot decrypt your production
ciphertext**. The certificate's private key isn't in source control, and
restoring a production backup doesn't help, because the key still isn't there.

This stops containerised database testing dead, and it stopped us for a while.

The way through is to stop trying to decrypt production data at all:

1. Let the publish generate a fresh certificate and symmetric key in the
   container.
2. Seed **known plaintext** test values.
3. Re-encrypt them under *that container's* key, using the same
   `OPEN KEY` / `EncryptByKey` pattern the application uses.

The application's own retrieval path then works unmodified — it opens the key,
decrypts, and gets the seeded test value. Auth-dependent tests run fully
locally.

The trade-off is explicit and worth stating: your tests now assert against a
seeded test key, not a production one. That has to be a deliberate decision by
whoever owns the test harness, not something the pipeline quietly does.

## Sampling real enum values without leaking anything

Auto-generated tests are much better when they use *realistic* values — real
status codes, real channel types — rather than `'test'` in every column.

We sample those from a read-only replica, guarded by a deliberately broad
column-name denylist. A column is sampled only if it does **not** match the
denylist **and** has cardinality ≤ 50.

That second condition does most of the work: a status column has eight
distinct values, a message-body column has millions. But cardinality alone
isn't enough — a low-cardinality `Country` or `Gender` column is still PII —
so the name denylist covers bodies, identifiers, tokens, contact details,
financial fields, and demographics.

When in doubt, deny. Falling back to a synthetic value costs nothing; coverage
is unaffected. Leaking a message body into a test fixture is not recoverable.

## What I'd tell someone starting this

- **Make the gate narrow.** One unambiguous check that is always right beats
  five checks that are usually right.
- **Generate starters, not verdicts.** A generated test that a human finishes
  is worth more than one a human has to argue with.
- **Solve the encryption problem early.** It's the thing that stops most teams,
  and it's solvable.
- **Throw the database away every time.** A shared test database drifts, and
  the day it drifts is the day the pipeline stops telling you the truth.

---

*The pipeline described here is open source:
[tsqlt-integration-pipeline](https://github.com/Bugzbaggy/tsqlt-integration-pipeline).*
