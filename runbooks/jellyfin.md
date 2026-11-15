# Runbook: Jellyfin

Jellyfin is the media server on the NAS, behind [Caddy](caddy.md) at `jellyfin.donny.lu`. This file and the `apps` dataset are sufficient to build it again. The reasons why it runs on the NAS, and what this CPU can transcode, are in [services.md](../docs/services.md). The events are in the [build log of phase 2](../build-log/phase-2.md).

The placeholder `<NAS_IPV4>` is the fixed IPv4 address of the NAS. Its real value is not in this repository.

## Host and identity

Jellyfin comes from the app catalogue of TrueNAS. No YAML is necessary.

| Item | Value |
| --- | --- |
| User ID | `568`, the `apps` user of the host |
| Group ID | `3002` (`media_access`). TrueNAS adds `apps` as a supplementary group itself |

## Storage

The storage is divided by this question: can the data be made again?

| Entry | Type | Location | In the backup |
| --- | --- | --- | --- |
| Config | Host Path | `NAS_Storage/apps/jellyfin/config` | Yes. It cannot be made again |
| Cache | ixVolume | `ix-apps` | No. It can be made again |
| Transcode | Temporary | The host disk | No |

The test for each new path: if this disk died now, what would the loss cost? If the answer is "nothing", use `ix-apps`. If the answer is "hours of work", use `apps`.

The media mount is read-only:

| Host path | Container path |
| --- | --- |
| `NAS_Storage/media/library` | `/media` |

Jellyfin only reads. Because the mount is read-only, deletions are done on the host through SFTP. Each feature that writes into the media folders must stay off (see "Library settings").

### The permissions of the config directory

A host path that a person makes by hand is `root:root 755`. A process that runs as `568:3002` is in the class *other* and cannot write. Jellyfin then stops with `Access to the path '/config/log' is denied`.

The fix: select **Enable ACL** on that host path, and add the entry **USER `568`, Modify**.

- Set the ACL on the directory of the service, never on the `apps` dataset. On the dataset, it would open the configuration of each service, credentials included.
- Use the checkbox, not `chown`. TrueNAS applies it at each deployment, and the middleware does not see an owner that was set by hand.
- Paths that the middleware makes itself need nothing.

## Network and resources

| Item | Value |
| --- | --- |
| WebUI port | Published (`30013`). Caddy reaches it at `http://<NAS_IPV4>:30013` |
| HTTPS port | Published, but not in use. TLS ends at Caddy. To close it is planned |
| Discovery port and host network | None, and off |
| CPUs and memory | 2, and 4096 MB |
| GPU passthrough | Quick Sync is passed through. It is not proven yet that Jellyfin uses it |

Quick Sync on Haswell does only H.264. HEVC decode, HDR tone-mapping and burned-in subtitles stay on the CPU, also when the passthrough works. Direct play needs none of them.

## Playback clients

Each reason that Jellyfin gives for a transcode is the client, which declines something. So the client decides the CPU load. The library is not limited to the weakest client ([services.md](../docs/services.md)).

| Client | Note |
| --- | --- |
| Browser | The weakest: no MKV, no HEVC and no HDR. Good for H.264 SDR |
| Jellyfin Media Player (desktop) | In use. It is based on mpv: MKV, HEVC Main 10, HDR10, PGS and 7.1 audio all play directly |
| Jellyfin for Android TV | The strongest. It is the only path on which HDR10 reaches an HDR panel |
| Samsung Tizen and LG webOS | Community builds. The Samsung build needs sideloading, and a new signature at intervals |

- The reasons have different costs. The container and the number of channels need a remux or a downmix, which is cheap. Bitmap subtitles (PGS or VOBSUB, burned in) and HDR to SDR each force a full new encode. Put the reasons in order before you make a theory. The [build log of phase 2](../build-log/phase-2.md) has an example.
- Text subtitles (SRT) cost nothing. If you select one in place of an embedded PGS track, that reason goes away.
- With direct play, the bandwidth is the limit. A stutter during Direct Play is a problem of the network or of the disk.

## Library settings

- **Content type.** *Movies* and *Shows* find their items in TheMovieDb and TVDB.
- **Folders.** Point each library at the deepest correct folder, because the scan is recursive.
- **Off, because the mount is read-only:** Metadata savers → Nfo, *Save artwork into media folders*, and *Save trickplay images next to media*. The metadata is in the config, on `apps`.
- **Trickplay and chapter images** are encode work on two cores. They are off. Turn on one at a time if you want them.
- ***Prefer embedded titles over filenames*.** Set it for each library, never globally. Some files have an embedded `title` tag that is not the title of the episode. With the setting on, the name in the library becomes that tag. The change comes only when something scans the file again, so the damage appears later, one item at a time.
- **Embedded metadata.** Jellyfin reads it again only when the modification time of a file changes. After a tool writes tags but keeps the mtime, run a metadata refresh, not a new scan.
- **Metadata downloaders** get titles and artwork, not media. Set a preferred download language.
