# blueberry-mirror

Mirror solution for the [Blueberry Linux](https://github.com/zsigisti/blueberry)
package repository (the `.pkg.tar.zst` + `bpm.index` produced by Blueberry's
`tools/mkrepo.sh` from OBS builds).

A mirror is just a directory containing `bpm.index` and the package files, served
over HTTP. `bpm-mirror-sync` keeps that directory in sync with an upstream repo
(verifying checksums); clients list multiple mirror URLs in `/etc/bpm/repos.conf`
and `bpm` fails over between them.

```
                       OBS build + tools/mkrepo.sh
                                 │
                          ┌──────▼───────┐
                          │   origin     │  https://repo.blueberry.lan/x86_64
                          │ bpm.index +  │
                          │ *.pkg.tar.zst│
                          └──────┬───────┘
                bpm-mirror-sync  │  (periodic, checksum-verified)
                   ┌─────────────┼─────────────┐
              ┌────▼────┐   ┌────▼────┐    ┌────▼────┐
              │ mirror1 │   │ mirror2 │ …  │ mirrorN │   (each: nginx + timer)
              └────┬────┘   └────┬────┘    └────┬────┘
                   └──────── bpm clients (failover) ──────┘
```

## Contents

| Path | What |
|------|------|
| `bin/bpm-mirror-sync` | sync an upstream bpm repo into a local mirror dir |
| `deploy/nginx-mirror.conf` | nginx site to serve the mirror |
| `deploy/systemd/*` | service + timer for periodic syncs |
| `mirrorlist` | canonical list of mirror URLs |

## Run a mirror

On a server (Rocky/Arch/Debian — anything with `wget`, `sha256sum`, `nginx`):

```sh
# 1. Install the sync tool
sudo install -Dm755 bin/bpm-mirror-sync /usr/local/bin/bpm-mirror-sync

# 2. First sync (origin -> local dir)
sudo mkdir -p /srv/blueberry-mirror
sudo bpm-mirror-sync https://repo.blueberry.lan/x86_64 /srv/blueberry-mirror --prune

# 3. Serve it
sudo cp deploy/nginx-mirror.conf /etc/nginx/conf.d/blueberry-mirror.conf
#   (edit server_name; on Rocky/SELinux: restorecon -Rv /srv/blueberry-mirror)
sudo nginx -t && sudo systemctl reload nginx

# 4. Keep it fresh (edit UPSTREAM/MIRROR in the unit first)
sudo cp deploy/systemd/blueberry-mirror.* /etc/systemd/system/
sudo install -Dm755 bin/bpm-mirror-sync /usr/local/bin/bpm-mirror-sync
sudo systemctl enable --now blueberry-mirror.timer
```

`bpm-mirror-sync` is idempotent — packages whose checksum already matches are
skipped, so the timer run is cheap. `--prune` removes local packages that have
fallen out of the upstream index.

## Point clients at the mirrors

On a Blueberry box, list the origin plus mirrors on one line in
`/etc/bpm/repos.conf` (order = failover order):

```
core https://repo.blueberry.lan/x86_64 http://mirror1.blueberry.lan/x86_64 http://mirror2.blueberry.lan/x86_64
```

Then `bpm update && bpm install <pkg>` — bpm tries each mirror in turn and moves
on when one is unreachable. The canonical URL list lives in `mirrorlist`.

## Notes

- Integrity is end-to-end: the origin `bpm.index` carries each package's sha256,
  `bpm-mirror-sync` verifies it on download, and `bpm` re-verifies on the client.
  A malicious or broken mirror can't serve a tampered package.
- Mirrors are stateless copies — no database, no server-side code. Losing a
  mirror loses nothing; re-run the sync.
- To mirror over the WAN cheaply, run the timer less often or front nginx with a
  CDN; package filenames are version-stamped and immutable, so they cache well.
