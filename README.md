# SQL Restore-Verify

**A backup you have never restored is a hope, not a recovery plan.**

Backup jobs report success the instant the file is written. They never prove the backup actually restores, restores *clean*, or restores *inside your RTO*. Corrupt backups, broken chains, and RTO blowouts stay invisible until the one night you need to recover - which is the worst possible moment to find out.

This is a pure T-SQL harness that automatically proves your backups are restorable. For each database you register, it:

1. Finds the latest backup chain (full + latest diff + subsequent logs).
2. Restores that chain onto a scratch/DR instance under a temporary name.
3. Runs `DBCC CHECKDB` on the restored copy to prove it's structurally sound.
4. Times the restore against your RTO target and flags breaches.
5. Logs the outcome - success/fail, duration, CHECKDB result, RTO breach.
6. Drops the temp copy to reclaim space.


## Safety first - read before running

**This restores databases onto whatever instance you run it on.** It is designed to run on a *separate* restore/verification instance that can read the production
backup path - not on the production server itself. Restoring a full copy plus running `DBCC CHECKDB` is heavy work; you do not want it competing with live
workloads on production.

- **Run only on a dedicated restore/verification instance, never production.**
- A real restore requires an explicit confirmation flag (`@IConfirmThisIsARestoreInstance = 1`) so you can't kick off restores on the wrong box by accident. There's also a commented allow-list of approved restore hosts you can enable for a hard guard.
- It **never touches the source database** - it only reads backup history and restores copies from the backup files, under a temp name (`RVTEST_<db>_<date>`).
- It **refuses to overwrite** an existing database (temp-name collision aborts), so even if mispointed it can't clobber a production database.
- `@Execute` defaults to `0` (dry run) so you can inspect the generated `RESTORE` plan before anything happens.
- The restore instance's service account needs **read access to the backup path** (typically a UNC share the production server writes backups to).

**Can I run it on the same server the backups came from?** Technically yes - it restores under a temp name so it won't overwrite anything — but you shouldn't. The restore + CHECKDB load lands on that server, so on production it can hurt live performance. Point it at a scratch instance instead. That's also more honest as a DR test: recovering onto *different* hardware is closer to what a real disaster looks like.

## Quick start

```sql
-- 0. One-time: create a linked server to the production instance whose msdb holds the backup history (configure security per your standards):
--    EXEC sp_addlinkedserver @server=N'PRODSQL01', @srvproduct=N'',@provider=N'SQLNCLI', @datasrc=N'PRODSQL01';

-- 1. Register a database. SourceLinkedServer names the linked server above;paths point at THIS instance's scratch drives:
INSERT INTO dbo.RestoreVerifyConfig
    (DatabaseName, SourceServer, SourceLinkedServer, RTOTargetSeconds,
     RestoreDataPath, RestoreLogPath)
VALUES
    (N'Orders', N'PRODSQL01', N'PRODSQL01', 1800,
     N'D:\RestoreTest\', N'D:\RestoreTest\');

-- 2. Dry run - see the RESTORE plan without running it:
EXEC dbo.usp_VerifyRestore @DatabaseName = N'Orders', @Execute = 0;

-- 3. Run it for real (must confirm this is a restore instance, not prod):
EXEC dbo.usp_VerifyRestore @DatabaseName = N'Orders', @Execute = 1,
     @IConfirmThisIsARestoreInstance = 1;

-- 4. Verify every registered database (schedule on Agent, nightly):
EXEC dbo.usp_VerifyRestore_All @Execute = 1,
     @IConfirmThisIsARestoreInstance = 1;

-- 5. Review the history:
SELECT DatabaseName, TestDate, Succeeded, CheckDBPassed,
       RestoreSeconds, RTOTargetSeconds, RTOBreached, ErrorMessage
FROM dbo.RestoreVerifyLog
ORDER BY TestDate DESC;
```

## What you learn that backup jobs never tell you

- **This backup is actually restorable** - not just "the file was written."
- **The restored copy is structurally clean** - CHECKDB passed on the real bytes.
- **You can still hit your RTO** - or you can't, and now you know before the
  incident instead of during it.
- **Your chain is intact** - a broken chain fails the restore here, in a test,
  where it's a finding instead of a disaster.

## How it works: read history remotely, restore locally

There's a subtlety that trips up every restore-verification setup: **backup history
lives in the *source* instance's `msdb`.** A fresh restore box has no idea what
production backed up. So this harness splits the work:

- It **discovers the backup chain remotely** by querying the production instance's `msdb` over a **linked server** (full + latest diff + subsequent logs).
- It **restores locally** on the scratch instance, reading the backup files from a shared path, and runs CHECKDB on the local copy.

That keeps the heavy restore + CHECKDB load off production while still using production's authoritative backup history.

**Two prerequisites:**

1. A **linked server** on the restore instance pointing at the production instance, with rights to read its `msdb` backup history. You register its name per database in `RestoreVerifyConfig.SourceLinkedServer`.
2. The **backup files must be reachable from the restore instance.** SQL Server records whatever path the backup job wrote - and runs the `RESTORE` using that path from the restore box's own perspective. Two cases:
   - **Backups already go to a UNC share** (`\\SERVER\Backups\...`): nothing to do, the restore box reads the same file over the network.
   - **Backups go to a local path** (`E:\Backups\...`): that path is meaningless on the restore box. Add a **path-mapping** row telling the framework how that
     server's local folder translates to a reachable UNC share. It does *not* guess share names - you supply the mapping:

     ```sql
     INSERT INTO dbo.RestoreVerifyPathMap (SourceServer, LocalPrefix, NetworkPrefix)
     VALUES (N'PRODSQL01', N'E:\Backups\', N'\\PRODSQL01\Backups\');
     ```
   Longest `LocalPrefix` wins, so you can layer a general rule plus more specific
   ones. If a path still looks local after mapping, the harness fails early with a
   clear message instead of a cryptic "cannot find file."

## Pairs with Disaster-Recovery-Drift

The `dbo.RestoreVerifyLog` rows this produces are exactly the feed that
[Disaster-Recovery-Drift](https://github.com/deepeshd87/Disaster-Recovery-Drift) expects for its restore-test signals. Together they form a loop:

- **sql-restore-verify** *proves* your backups restore (the hard evidence).
- **Disaster-Recovery-Drift** *trends* that evidence over time, scoring readiness and catching drift - including restore-time creep and restore-test staleness - before it becomes an incident.

## Requirements

SQL Server 2016+ (uses `CREATE OR ALTER`). A linked server from the restore
instance to each production instance (to read backup history). Backup files on a
path the restore instance can read (UNC share). Enough scratch disk to hold the
largest database you verify.

---

Built from production DR practice across SQL Server databases in regulated,
compliance-driven environments - where "we have backups" is not an acceptable
answer to "can you prove you can recover?"
