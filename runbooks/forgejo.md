# Runbook: Forgejo

Forgejo is self-hosted git on the NAS, behind [Caddy](caddy.md) at `forgejo.donny.lu`. It is the remote for everything that is not public. This file and the `apps` dataset are sufficient to build it again. The reason is in [services.md](../docs/services.md).

## Structure

- The TrueNAS catalogue app *Forgejo*, image `16.0.5-rootless`, with a bundled Postgres 18 container. Two jobs (`permissions` and `postgres_upgrade`) run one time at each deployment and stop. `forgejo` and `postgres` stay in operation.
- Plain HTTP behind Caddy. It is reachable only from the LAN and the tailnet.

## Storage

| Entry | Type | Path | Access | Mounted at |
| --- | --- | --- | --- | --- |
| Forgejo Data | Host Path | `NAS_Storage/apps/forgejo/data` | ACL: USER `568`, Modify | `/var/lib/gitea` |
| Forgejo Postgres Data | Host Path | `NAS_Storage/apps/forgejo/postgres` | Owner `999:999` | The data directory of Postgres |

- The default of both entries is ixVolume, and the second entry is lower on the form. An ixVolume has no snapshots and no replication. If you miss the database, a restore gives repositories with no users, no keys and no issues.
- These two paths are the full state. On 16.x, `app.ini` (with the generated secrets) is at `/var/lib/gitea/custom/conf/app.ini`, in Data.
- Both paths are in one dataset, so one snapshot holds the repositories and the database together. This is data, not configuration ([backup.md](../docs/backup.md)).

## Permissions

Both folders were made by hand as `root:root 755`.

- **Data.** Forgejo runs as `568:568` and cannot write. The result is a restart loop with `mkdir: can't create directory '/var/lib/gitea/git': Permission denied`. Select **Enable ACL**, and add the entry USER `568`, Modify. The checkbox alone gives no permission.
- An app update fails with `path contains existing data and force was not specified`. Select *Force Flag* on the ACL of the Data path.
- **Postgres.** It runs as `999:999`, and it does not start unless it owns its directory. Here, `chown 999:999` before the first deployment did that. As an alternative for a new build, *Automatic Permissions* below the Postgres entry can set the owner. It is not tested here. `ls -ln` shows the numbers. `ls -l` shows the names that the NAS has for those numbers, which have no relation to Postgres.

## Configuration

| Setting | Value |
| --- | --- |
| Postgres image | Postgres 18. An upgrade goes only forward, so make a snapshot of `apps` before you change it |
| Database password | Set at the first deployment, then fixed in the database. A restore must use the same password, which is also in `app.ini`. Never put it in a file that git tracks |
| Root URL | `https://forgejo.donny.lu`. Forgejo prints it in clone URLs and links. It must include the scheme |
| WebUI and SSH ports | `30142` and `30143` (the defaults). External SSH Port is empty |
| Run as | `568:568` |
| Self-registration | Disabled |

## Repositories and clients

A new repository is public to the LAN by default. Set each repository that must be private to **Private**.

A repository can have a copy on GitHub and a copy on Forgejo, and one push can go to both:

- Give `origin` a second `url`: `git remote set-url --add origin https://forgejo.donny.lu/<user>/<repo>.git`. With more than one `url` and no `pushurl`, `git push` goes to all of them, and `git fetch` uses the first.
- Do not use `set-url --add --push`. When a `pushurl` exists, git no longer pushes to the `url` entries. Then the first remote gets nothing, with no error.
- This setting is in `.git/config` of each clone. A new clone needs the second `url` again.
- To remove a duplicate URL, edit `.git/config`. `set-url --delete` removes each match.
- To verify a push, compare the latest hash on the Forgejo page with `git log -1`.

Git over SSH on port `30143` is not tested yet.
