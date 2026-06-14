# blueberry-mirror

Repository infrastructure for [Blueberry Linux](https://github.com/zsigisti/blueberry).
Two tools, two roles:

- **`blueberry-repo-sync`** — the **origin**. Builds the package repo *from the
  git recipes* and hosts it. Run this on one server (e.g. your Rocky box).
- **`bpm-mirror-sync`** — an optional **mirror**. Copies an existing origin to a
  second server for redundancy. You only need this once you have more than one
  host serving packages.

```
        github.com/zsigisti/blueberry  (packages/*/PKGBUILD recipes)
                                 │
                 blueberry-repo-sync  (git pull → build in Arch container → bpm.index)
                                 │
                          ┌──────▼───────┐
                          │   ORIGIN     │  http://<rocky>/x86_64
                          │ bpm.index +  │
                          │ *.pkg.tar.zst│
                          └──────┬───────┘
                  bpm-mirror-sync│  (optional, checksum-verified)
                   ┌─────────────┼─────────────┐
              ┌────▼────┐   ┌────▼────┐    ┌────▼────┐
              │ mirror1 │   │ mirror2 │ …  │ mirrorN │
              └────┬────┘   └────┬────┘    └────┬────┘
                   └──────── bpm clients (failover) ──────┘
```

## Contents

| Path | What |
|------|------|
| `bin/blueberry-repo-sync` | **build the repo from the git recipes and publish it (the origin)** |
| `bin/bpm-mirror-sync` | copy an existing origin into a local mirror dir |
| `deploy/blueberry-repo-sync.conf.example` | config for the origin builder |
| `deploy/nginx-repo.conf` | nginx site for the origin |
| `deploy/nginx-mirror.conf` | nginx site for a mirror |
| `deploy/systemd/blueberry-repo-sync.*` | service + timer for the origin builder |
| `deploy/systemd/blueberry-mirror.*` | service + timer for a mirror |
| `deploy/INSTALL-rocky.md` | **step-by-step Rocky Linux 10 setup** |
| `mirrorlist` | canonical list of repo/mirror URLs |

## The origin: `blueberry-repo-sync`

This is the "whole repo solution". On the server it:

1. Clones/pulls the Blueberry git repo.
2. Builds every `packages/*/PKGBUILD` recipe. The recipes are Arch PKGBUILDs, so
   they need `makepkg` — which Rocky doesn't have — so the build runs inside an
   **ephemeral Arch Linux container** (`podman`). The host only needs `git`,
   `podman`, and `nginx`.
3. Generates `bpm.index` and **atomically swaps** the result into the web root.

Packages are **not signed** (for now); integrity is the sha256 carried in
`bpm.index` and re-verified by `bpm` on the client.

**Quick start (Rocky Linux 10):** see [`deploy/INSTALL-rocky.md`](deploy/INSTALL-rocky.md).
The short version:

```sh
sudo dnf install -y git podman nginx
sudo install -Dm755 bin/blueberry-repo-sync /usr/local/bin/blueberry-repo-sync
sudo install -Dm644 deploy/blueberry-repo-sync.conf.example /etc/blueberry-repo-sync.conf
sudo install -Dm644 deploy/nginx-repo.conf /etc/nginx/conf.d/blueberry-repo.conf
sudo mkdir -p /var/www/html/x86_64 && sudo systemctl enable --now nginx
sudo blueberry-repo-sync                     # first build (watch it)
sudo cp deploy/systemd/blueberry-repo-sync.* /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now blueberry-repo-sync.timer   # rebuild hourly from git
```

## The mirror: `bpm-mirror-sync` (optional)

Once an origin exists, a mirror is just a directory containing `bpm.index` plus
the package files, served over HTTP. `bpm-mirror-sync` keeps it in sync with the
origin (verifying checksums); clients list multiple URLs and `bpm` fails over.

```sh
sudo install -Dm755 bin/bpm-mirror-sync /usr/local/bin/bpm-mirror-sync
sudo mkdir -p /srv/blueberry-mirror
sudo bpm-mirror-sync http://<origin>/x86_64 /srv/blueberry-mirror --prune
sudo cp deploy/nginx-mirror.conf /etc/nginx/conf.d/blueberry-mirror.conf
sudo nginx -t && sudo systemctl reload nginx
sudo cp deploy/systemd/blueberry-mirror.* /etc/systemd/system/   # edit UPSTREAM/MIRROR
sudo systemctl enable --now blueberry-mirror.timer
```

`bpm-mirror-sync` is idempotent — packages whose checksum already matches are
skipped. `--prune` removes packages that fell out of the origin index.

## Point clients at it

On a Blueberry box, in `/etc/bpm/repos.conf` (order = failover order):

```
blueberry http://<origin>/x86_64 http://<mirror1>/x86_64
```

To also resolve upstream library dependencies (oniguruma, libevent, openssl, …)
that the Blueberry packages link against, add the Arch repos as extra sources —
`bpm` reads pacman `.db` files directly:

```
blueberry http://<origin>/x86_64
extra     https://geo.mirror.pkgbuild.com/extra/os/x86_64
core      https://geo.mirror.pkgbuild.com/core/os/x86_64
```

Then `bpm update && bpm install vim`. Base packages (glibc, bash, …) are skipped
via `/etc/bpm/provided`.

## Notes

- **No package signing.** Integrity is the sha256 in `bpm.index`, fetched over
  HTTP. Put it behind HTTPS or a trusted LAN for tamper resistance.
- Integrity is end-to-end: the origin records each sha256, mirrors verify on
  download, and `bpm` re-verifies on the client.
- Mirrors are stateless copies — no database, no server-side code. Losing one
  loses nothing; re-run the sync.
