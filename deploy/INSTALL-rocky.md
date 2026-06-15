# Deploying the Blueberry repo on Rocky Linux 10

`blueberry-repo-sync` turns the PKGBUILD recipes in the Blueberry git repo into
a hosted package repository. It builds them inside an ephemeral Arch Linux
container (Rocky has no `makepkg`), so the only host dependencies are **git**,
**podman**, **openssl**, and **nginx**. The `bpm.index` is **signed on the host**
with an ECDSA P-256 key (`bpm` rejects an index not signed by the trusted key);
each package's sha256 in the index then anchors the package files.

## 1. Install prerequisites

```sh
sudo dnf install -y git podman openssl nginx
```

## 2. Install the script and config

```sh
sudo install -Dm755 bin/blueberry-repo-sync /usr/local/bin/blueberry-repo-sync
sudo install -Dm644 deploy/blueberry-repo-sync.conf.example /etc/blueberry-repo-sync.conf
sudoedit /etc/blueberry-repo-sync.conf      # adjust REPO_URL / OUT if needed
```

## 3. Configure nginx

```sh
sudo install -Dm644 deploy/nginx-repo.conf /etc/nginx/conf.d/blueberry-repo.conf
sudo mkdir -p /var/www/html/x86_64
# SELinux: let nginx read the web root and let podman write under it.
sudo setsebool -P httpd_read_user_content 1
sudo chcon -R -t httpd_sys_content_t /var/www/html
sudo systemctl enable --now nginx
sudo firewall-cmd --add-service=http --permanent && sudo firewall-cmd --reload
```

## 4. First build (run by hand once to watch it)

```sh
sudo blueberry-repo-sync
```

This clones the repo, pulls the Arch image, compiles all 9 packages, writes
`bpm.index`, and publishes to `/var/www/html/x86_64`. Check it:

```sh
curl -s http://localhost/x86_64/bpm.index | head
```

## 5. Automate with the timer

```sh
sudo install -Dm644 deploy/systemd/blueberry-repo-sync.service /etc/systemd/system/blueberry-repo-sync.service
sudo install -Dm644 deploy/systemd/blueberry-repo-sync.timer   /etc/systemd/system/blueberry-repo-sync.timer
sudo systemctl daemon-reload
sudo systemctl enable --now blueberry-repo-sync.timer
```

The repo now rebuilds hourly from git. Watch a run with:

```sh
journalctl -u blueberry-repo-sync.service -f
```

## 6. Point Blueberry clients at it

On the booted Blueberry system, in `/etc/bpm/repos.conf`:

```
blueberry http://<this-server-ip>/x86_64
```

Then `bpm update && bpm install vim`. To also resolve upstream library
dependencies (oniguruma, libevent, openssl, …) add the Arch repos as extra
sources — see the main README.

## Signing key

The host signs `bpm.index` so clients will trust it. Generate the key once and
keep it private (root-only):

```sh
sudo mkdir -p /etc/blueberry
sudo openssl ecparam -name prime256v1 -genkey -noout \
    -out /etc/blueberry/repo-signing-key.pem
sudo chmod 600 /etc/blueberry/repo-signing-key.pem
```

Then bake the **public** half into `bpm` so installed systems trust this repo:
on a machine with the Blueberry git checkout, copy the private key there and run
`tools/mkrepokey.sh /path/to/repo-signing-key.pem`, rebuild the image, and
reinstall. (The private key never needs to leave the server if you instead copy
just the public point; `mkrepokey.sh` only reads the public half.)

`blueberry-repo-sync` reads `SIGN_KEY` (default
`/etc/blueberry/repo-signing-key.pem`). To publish without signing during
bring-up, set `ALLOW_UNSIGNED=1` — clients then need `BPM_ALLOW_UNSIGNED=1`.

## Notes

- **Signed index.** `bpm` verifies the ECDSA signature on `bpm.index` against
  its baked-in key, then the per-package sha256 anchors the files. Still put the
  repo behind HTTPS so the package downloads themselves aren't tampered in flight.
- **Rootless podman** works too; if you run the timer as a non-root user, make
  sure that user owns `WORK` and can write `OUT`.
- A build failure in one recipe doesn't abort the rest, but it does fail the
  unit (non-zero exit) so the timer/journal flags it.
