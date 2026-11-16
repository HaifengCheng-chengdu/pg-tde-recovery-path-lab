---
title: "Which Keys Does a pg_tde Restore Need? A Nine-Case Rotation Lab"
date: "2026-11-13T18:17:22+00:00"
draft: true
tags: ['PostgreSQL', 'Backups', 'Security', 'pg_tde', 'pgBackRest']
categories: ['PostgreSQL', 'Community']
authors:
  - haifeng_cheng
images:
  - blog/2026/11/pg-tde-recovery-path-lab-cover.png
slug: pg-tde-recovery-path-key-matrix
description: "A reproducible pg_tde and pgBackRest lab tests nine backup-to-target key combinations and separates repository verification, WAL recovery and complete encrypted-data validation."
---

<!-- Draft date records manuscript preparation, not an agreed publication slot. -->

A restore can pass its repository checks, copy every backup file and still fail when PostgreSQL tries to recover encrypted data. In this lab, all nine pgBackRest file restores succeeded in each of three rounds. Five of the nine paths reached the requested target with the complete data intact; four stopped because a required database principal key was unavailable.

The useful unit for a key-retention decision is the **path from a particular backup to a particular recovery target**. With the WAL key and repository passphrase available throughout, the lab produced three results:

- Old backup → target before rotation: K0 was sufficient.
- Old backup → target after rotation: both K0 and K1 were needed.
- New full backup → target after rotation: K1 was sufficient.

The second path is the reason to test more than the key active when a backup was taken. Starting under K0 does not remove the later need for K1 when replay crosses the rotation.

The results come from a small synthetic lab. Execution and checks were automated. The [reproducibility companion](/blog/2026/11/pg-tde-recovery-path-lab/percona-pg-tde-recovery-path-lab-20261113.zip) contains the tested runner, recorded outputs and checksums. Its [SHA-256 file](/blog/2026/11/pg-tde-recovery-path-lab/SHA256SUMS) identifies the download.

## What the matrix adds to existing guidance

Percona already documents that [older encrypted backups need accessible older principal keys](https://docs.percona.com/pg-tde/how-to/restore-backups.html), and recommends [rotating outside backup windows and taking a new full backup afterwards](https://docs.percona.com/pg-tde/global-key-provider-configuration/overview.html). This lab follows that guidance.

The earlier [pg_tde tutorial](https://percona.community/blog/2025/09/01/encrypting-postgresql-tables-with-pg_tde-step-by-step-guide-for-beginners/) proposes restore-time key-access and rotation experiments. The [Percona pgBackRest walkthrough](https://percona.community/blog/2026/03/10/running-pgbackrest-with-pg_tde-a-practical-percona-walkthrough/) covers configuration and restoration, while [Stefan Fercot's experiments](https://pgstef.github.io/2026/06/04/pgbackrest_and_pg_tde.html) demonstrate pg_tde with pgBackRest and point-in-time recovery.

The contribution here is a nine-case key-availability matrix with explicit backup selection, exact target checks and complete data comparisons. It separates three outcomes that an operational check needs to distinguish: the repository is readable, PostgreSQL reaches the requested target, and the recovered encrypted data matches that target.

## Keep three kinds of key separate

The original three rounds used:

- Percona Server for PostgreSQL 18.7.1, based on PostgreSQL 18.7.
- pg_tde 2.2.2, confirmed with SELECT pg_tde_version().
- pgBackRest 2.60.0.
- Ubuntu 24.04.5 LTS, x86_64. The host openssl command reported 3.0.13.

The official distribution tarball and individual binary hashes are recorded in the companion. Its filename contains ssl3.5; the host version above is not proof of the library loaded by each database binary. Dynamic-library resolution was not preserved for the original processes. A separate 14 November observation found PostgreSQL loading the Ubuntu OpenSSL 3.0.13 libraries.

The [official tarball guidance](https://docs.percona.com/postgresql/18/tarball.html) maps ssl3.5 to OpenSSL 3.5.x. To make the installation repeatable on a matching environment, the separately dated [clean-install supplement](/blog/2026/11/pg-tde-recovery-path-lab/percona-pg-tde-clean-install-20261115.zip) uses Debian 13 and OpenSSL 3.5.7 with the same PostgreSQL, pg_tde and pgBackRest binary hashes. Two fresh container runs reproduced the matrix and controls. Runtime process maps captured PostgreSQL with pg_tde, and pgBackRest, loading that environment's libssl and libcrypto. These are new observations, not a reconstruction of the original libraries or installation commands. Neither setup is a production installation recommendation.

The lab replaces database principal key **K0**, in file provider p0, with **K1**, in file provider p1. It keeps the server/WAL principal key **W0** and the pgBackRest repository passphrase unchanged.

Both providers and both database keys existed before F0. The rotation changed the active key from K0/p0 to K1/p1; it did not create a provider during replay. The restore cases vary access to those key files along this fixed timeline. Because the rotation also changes provider identity, this design does not isolate every effect of rotation within a single provider.

Those are different layers. Principal-key replacement rewraps the database's internal keys; it does not mean every table page is rewritten. The [architecture documentation](https://docs.percona.com/pg-tde/architecture/key-provider-management.html) explains that hierarchy.

Local file providers make the experiment self-contained. They are not a recommended production KMS. The negative cases withhold replaceable copies of key-bearing files while retaining the originals. They test availability through those configured paths, not a remote KMS's access-control, deletion or versioning behaviour.

WAL uses the [currently documented supported configuration](https://docs.percona.com/pg-tde/wal-encryption.html): archive/restore wrappers, archive-async=n and the specified header/checksum settings. The repository is encrypted. The wrapper path temporarily handles plaintext WAL in shared-memory storage, so this is not a claim that plaintext never exists anywhere in the process.

## Build two backups and two precise targets

The encrypted ledger uses tde_heap, a primary-key index and large externally stored payload values. Every payload is included in the final comparison.

The sequence is deliberately ordered:

1. Create 256 synthetic rows under K0 and finish full backup F0.
2. Commit a deterministic update batch and create the named restore point before_rotation.
3. Replace K0 with K1, outside the completed backup window.
4. Commit another update batch, add 16 rows and finish full backup F1.
5. Record the complete after-state and create after_rotation.
6. Wait for the target WAL segment to be archived, then stop the source.

Each restore names its backup label explicitly. It cannot silently choose F1 when the case is supposed to exercise F0. Each also names its recovery target and starts in a new data directory with a cold server process.

The source verification reads are part of the workload too: they can generate WAL through page maintenance. In these runs, a post-F1 PRUNE_ON_ACCESS record became relevant to one negative case. The failure location is therefore a property of this fixture, not a universal promise about when every missing-key restore fails.

## The observed matrix

The same frozen runner completed three fresh rounds on one host. Each round created its own source, backups and keys.

| Starting backup and target | K0 and K1 available | Only K0 available | Only K1 available |
|---|---|---|---|
| F0 → before_rotation | **K0 + K1:** Pass | **K0 only:** Pass | **K1 only:** Fails, K0 unavailable |
| F0 → after_rotation | **K0 + K1:** Pass | **K0 only:** Fails, K1 unavailable | **K1 only:** Fails, K0 unavailable |
| F1 → after_rotation | **K0 + K1:** Pass | **K0 only:** Fails, K1 unavailable | **K1 only:** Pass |

W0 and the repository passphrase remained available in every column.

All 27 core verify commands and all 27 file-restore commands returned zero. The nine cases in each round reused that round's repository; these were three repositories, not 27 independent ones. The 15 successful recoveries matched their complete source data; the other 12 failed with the relevant missing-key error. No unrelated connection error was accepted as a successful negative test.

The missing-key failures occurred after the repository checks and file copies succeeded. They do not, by themselves, establish backup-file corruption. The pgBackRest [verify command checks repository contents](https://pgbackrest.org/command.html#command-verify); recovery and encrypted reads exercise additional dependencies.

## Read the crossing-rotation cases together

In round jp-round02, old_after_both restored F0 with both database keys available. It reached after_rotation at replay LSN 0/7006968 and returned the complete after-state: 272 rows and 1,116,466 CSV bytes, matching the source SHA-256. This is the successful crossing-rotation control.

With the same starting backup and target, old_after_k0 retained K0 but withheld K1. Its repository verification, file restore and pg_ctl start commands all returned zero. Recovery subsequently failed. The server recorded:

```text

FATAL: failed to retrieve principal key "K1" from key provider "p1" for database 5
CONTEXT: WAL redo at ... for pg_tde/ROTATE_PRINCIPAL_KEY: db: 5
```

The quoted context abbreviates the LSN with an ellipsis. The full log records redo at 0/5001F78; the error and context are adjacent entries from the same process. These LSNs identify this round, not a value to expect in a new run.

The [2.2.2 replay implementation](https://raw.githubusercontent.com/percona/pg_tde/2.2.2/src/catalog/tde_principal_key.c) explains the dependency: rotation redo obtains the current principal key, retrieves the replacement key from its provider and rewraps internal keys. The [WAL dispatcher](https://raw.githubusercontent.com/percona/pg_tde/2.2.2/src/access/pg_tde_xlog.c) sends the rotation record to that implementation. This is a source-based explanation of the logged failure; the successful control separately shows that the target and full data were reachable with both keys.

K0 supplied the old side of that path, but could not supply K1 at the transition. Keeping K1 alone also failed for F0: it did not provide the K0 needed to replay earlier encrypted changes. F1 was taken after the transition and recovered with K1 alone in this fixture. That result applies to the F1 path. It does not remove the K0 dependency of a retained F0 path.

## Define success beyond “the command finished”

The acceptance checks were separate:

1. Repository verification and file restore both returned zero.
2. The server log named the requested restore point.
3. Recovery completed the requested promotion.
4. pg_last_wal_replay_lsn() equalled that run's recorded target LSN.
5. The target marker and row count matched.
6. An ordered CSV export of every row and full payload matched the source SHA-256.
7. The table remained encrypted tde_heap, and an index-based range query matched the reference aggregate.

For the range-query check, the runner set enable_seqscan=off and recorded an index-based EXPLAIN JSON plan. The separately executed aggregate matched the reference. This exercises the index-access check; it is not evidence of an unconstrained planner choice or index performance. EXPLAIN ANALYZE was not used, so the saved plan is not a measured execution trace. Complete-data equality did not depend on matching plan shapes.

The before-state contained 256 rows and 1,050,512 exported bytes; the after-state contained 272 rows and 1,116,466 bytes. All three rounds produced the same two data hashes. Target LSNs were checked against each run's own source, not copied from another round.

Even pg_ctl start returned zero in the recorded missing-key cases before recovery subsequently failed. Connection readiness was therefore insufficient as an acceptance condition. The runner continued checking recovery and data, and it stopped each instance before changing key availability.

## Check the control-file state before a provider-path repair

A missing key can also be a path problem: the key exists, but the provider points somewhere else. Two additional cases per round examined that condition separately from the nine-case matrix.

The stale-path case failed to recover because its configured path could not supply K0. A second case attempted to edit the provider on a fresh restore directory before its first startup. The offline tool returned:

```text

pg_tde_change_key_provider: error: cluster must be shut down
```

No server was running against that directory. However, pg_controldata reported **in production**, the state inherited from the online physical backup. An absent process was therefore insufficient to satisfy the tool's control-file check.

The [2.2.2 tool implementation](https://raw.githubusercontent.com/percona/pg_tde/2.2.2/src/bin/pg_tde_change_key_provider.c) checks the control-file CRC and accepts DB_SHUTDOWNED or DB_SHUTDOWNED_IN_RECOVERY. The restored directory's in production state matched neither. The experiment preserved that refusal; it did not modify pg_control to bypass the check.

The positive control started from a different state: a copy of an F1 instance that had already completed recovery and then shut down normally. It performed these steps:

- Verify the complete after-state before changing the path.
- Move the K1 file and observe the encrypted read fail.
- Shut down normally and confirm the control-file state.
- Update p1's path with the offline tool.
- Start a new server process and repeat the complete data and index-query checks.

The edit returned zero and the data checks passed. This demonstrates relocation for an already recovered, cleanly stopped instance. It does not demonstrate repair of the fresh online-backup restore that the tool refused. Although the implementation also accepts DB_SHUTDOWNED_IN_RECOVERY, this lab did not exercise that state as a successful repair case.

When using the [provider-repair procedure](https://docs.percona.com/pg-tde/command-line-tools/pg-tde-change-key-provider.html), check both the instance state and the recovered data. A successful configuration edit alone does not establish that a particular recovery path can obtain all of its keys.

## Reproduce or audit the result

Download the unchanged [original companion ZIP](/blog/2026/11/pg-tde-recovery-path-lab/percona-pg-tde-recovery-path-lab-20261113.zip) and the [15 November clean-install supplement](/blog/2026/11/pg-tde-recovery-path-lab/percona-pg-tde-clean-install-20261115.zip). Check both against [SHA256SUMS](/blog/2026/11/pg-tde-recovery-path-lab/SHA256SUMS). Start with the supplement's README for the pinned Debian image, package versions, fresh extraction commands, resource limits and cleanup. It does not require the original lab's vendor directory.

The follow-up provisioned dependencies once, then used that frozen dependency image for two separate non-root, network-disabled test containers. Each extracted the original inputs into an empty directory and ran the unchanged runner and provider control. Both produced the original before/after data hashes, five successful core recoveries and four expected missing-key failures. The original three-round evidence remains unchanged; the supplement records the new tests separately. Statements in the frozen original companion about installation and runtime-library checks describe its earlier status. The supplement provides a tested installation entry point on a new environment; it cannot recover the original installation commands or original process maps.

For the original three-round workflow, after preparing the lab directory, run:

```sh

mkdir -p runs artifacts
for n in r1 r2 r3; do python3 src/run_matrix.py --name "$n" || exit 1; done
python3 src/run_clean_provider_control.py --base r1 --name clean1
python3 src/verify_recorded_evidence.py --runs-root runs --rounds r1 r2 r3 --control clean1 --allow-unmanifested
```

Use a disposable non-root environment that permits local Unix sockets, and keep the execution session alive until completion. Do not use real keys or production data. The same verifier can audit the bundled recorded results without starting a database.

## Make key retention follow the promised recovery paths

The matrix supports a specific operational check: start with the backup and target that must remain recoverable, then establish which keys that path needs. The current active key is only one entry in that record.

For each retained path, keep:

- The backup label and recovery target used in the test.
- The database principal-key identifiers and provider configuration required along the path.
- The separate WAL/server-key and repository-credential requirements.
- The repository result, target evidence and complete application-data comparison.
- The date and software configuration of the last successful recovery with a new server process.

For this fixture, a retention decision that keeps F0 → after_rotation must keep access to both K0 and K1. Taking F1 adds a path that succeeded with K1 alone; it does not change the requirements of F0. Before retiring a key, identify every retained path that still depends on it rather than treating the newest full backup as evidence about older ones.

The evidence covers one database, one K0→K1 replacement across two file providers, a fixed WAL key and one software build. It does not establish behaviour for external KMS failures, repeated rotations, server-key rotation, standby failover, rotation during a backup, or cross-version recovery. The original three rounds and two clean-install runs demonstrate repeatability within that scope. They provide no latency, throughput or general production-reliability estimate.

The reproducibility package contains the exact scripts, per-run source snapshots, results, logs and checksums needed to examine these claims. Key files, repository passphrases, database directories and the backup repository are excluded.

