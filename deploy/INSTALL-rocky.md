# Deploying the Blueberry repo on Rocky Linux 10

`blueberry-repo-sync` turns the PKGBUILD recipes in the Blueberry git repo into
a hosted package repository. It builds them inside an ephemeral Arch Linux
container (Rocky has no `makepkg`), so the only host dependencies are **git**,
**podman**, and **nginx**. Packages are **not signed** — `bpm` verifies the
sha256 from `bpm.index`.

## 1. Install prerequisites

```sh
sudo dnf install -y git podman nginx
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

## Notes

- **No signing.** Integrity is the sha256 in `bpm.index`, fetched over HTTP.
  Put it behind HTTPS (or a trusted LAN) if you care about tamper resistance.
- **Rootless podman** works too; if you run the timer as a non-root user, make
  sure that user owns `WORK` and can write `OUT`.
- A build failure in one recipe doesn't abort the rest, but it does fail the
  unit (non-zero exit) so the timer/journal flags it.
