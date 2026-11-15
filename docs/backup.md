# Backup

RAID survives a dead drive. It does not protect against deletion, corruption, ransomware or a bad `rm`. The target is 3-2-1: three copies, on two media, with one copy offsite.

The pool and the datasets are in [storage.md](storage.md).

## The copies

| Copy | What it is | State |
| --- | --- | --- |
| 1 | The pool | Built |
| 2 | Snapshots on the pool | Built |
| 3 | Replication to an external drive, which stays unplugged | Planned. The drive is not bought yet |
| Offsite | A Hetzner Storage Box | Planned |

**Copy 2: snapshots.** Two snapshot tasks run, both recursive from the root of the pool. The daily task has a retention of 2 weeks, and the weekly task has a retention of 3 months ([TrueNAS storage runbook](../runbooks/truenas-storage.md)). Snapshots are good against logical loss. They are of no use if the pool dies, so they are not a real second copy.

**Copy 3: the external drive.** It is not built yet.

## The accepted exposure

The NAS runs without copy 3. This is a deliberate choice.

Before the NAS, the data that cannot be replaced was on one desktop drive for years, with no checksums and no snapshots. Now it is on RAIDZ1 with checksums and snapshots, on drives that passed a burn-in. The added risk of the delay is small. These are not covered:

- Two failed drives
- The loss of the pool: a bad `zfs destroy`, or a fault of the controller or of the PSU
- Fire or theft
- Operator error

The personal files are also on the workstation. They were copied there during the migration of the workstation on 26 Aug. These are two manual copies, not a synced pair, and neither is offsite. Their planned offsite copy is a Hetzner Storage Box.

The rule for each migration: copy the data, verify it by file count and by bytes on the receiving side, and only then erase the source.

## The backup set

- The personal files and the personal media collection
- `apps`. This includes the repositories and the database of Forgejo, which are the one part of `apps` that is data and not configuration. The clones hold the history. The database is only here.
- `home`
- The config export of TrueNAS. With it, a new install becomes this NAS again in minutes. The export is made by hand and kept on the workstation. A scheduled export is planned. The export contains password hashes and the token of the Telegram bot. Do not put it in a file that git tracks.

`media` is not in the backup set, because it can be replaced.

## What snapshots cost

The snapshot tiers were decided together with the dataset layout.

- **The cost follows the churn, not the size.** A snapshot holds only the blocks that change or are deleted later. Deleted blocks stay allocated until each snapshot that holds them expires.
- **`media` is on the daily tier only.** This limits that delay to 2 weeks, where the weekly tier gives 3 months. The daily tier still gives a rollback if a tool renames the full library by mistake.
- **A bulk rewrite costs double until the snapshots expire.** A new tag, a remux or a new encode writes new blocks, and the snapshots keep the old blocks. A rewrite of 1 TB uses approximately 2 TB for up to three months.

## Test the backup

One trial restore is planned before Phase 2 of the build closes. Until a restore works, the backup is not proven.
