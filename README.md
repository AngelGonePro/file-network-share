https://raw.githubusercontent.com/AngelGonePro/file-network-share/refs/heads/main/network-share.zip


```
rm -rf ~/network-share && \
mkdir -p ~/network-share && \
wget -q -O /tmp/network-share.zip https://raw.githubusercontent.com/AngelGonePro/file-network-share/refs/heads/main/network-share.zip && \
unzip -q /tmp/network-share.zip -d ~ && \
rm /tmp/network-share.zip && \
cd ~/network-share && \
ls -la
```

```
cp .env.example .env
nano .env      # set NC_TRUSTED_PROXIES to your proxy VM's IP, set NC_PORT
```

```
docker compose up -d --build
```

# Network Share (Samba) — Deployment Runbook

A performant SMB share, built for feeding video to Jellyfin over the network and
for being mounted into Nextcloud as External Storage. Everything configurable
lives in `.env`. This doc exists because the first deployment hit four real,
non-obvious bugs — read "Why force user/force group" before you skip it.

---

## 0. Before you deploy — the one thing that will bite you if you skip it

This share's permission model depends on a shared Linux group (e.g. `mediashare`)
that already exists on your host, owns the storage folder, and is NOT the same
GID as any individual user's personal group. If you don't already have that
group set up, create it first:

```bash
groupadd mediashare
usermod -aG mediashare <your-user>
getent group mediashare   # note the GID in the 3rd field — you need this exact number
```

**Do not assume this GID equals your user's own GID.** On Debian, personal user
GIDs and group GIDs are both assigned sequentially starting around 1000, so
`angel`'s personal GID might be `1000` while `mediashare` is `1001` — two
different numbers that look interchangeable but aren't. Mixing them up is what
caused hours of "auth works, every write still gets denied" last time.

---

## 1. Host folder setup

```bash
mkdir -p /root/data/NetworkShare
chown <user>:mediashare /root/data/NetworkShare
chmod 2775 /root/data/NetworkShare
```

```bash
apt install -y acl
setfacl -R -d -m g:mediashare:rwx /root/data/NetworkShare
setfacl -R -m g:mediashare:rwx /root/data/NetworkShare
```

The `2` in `2775` is the setgid bit (new files/folders inherit the `mediashare`
group). The default ACL is what forces group-*write* on every new file too —
setgid alone only handles group *ownership*, not write access.

**Known gap:** Samba's recycle-bin module (enabled below) creates its `.deleted`
folder with its own permissions, which do NOT reliably inherit this default ACL.
After first deploying, re-run the two `setfacl` lines above once more to catch it.

---

## 2. Deploy

```bash
git clone <your-repo-url> && cd network-share
cp .env.example .env
nano .env
```

Fill in, at minimum:
- `NETWORK_SHARE_PASSWORD`
- `NETWORK_SHARE_UID` / `NETWORK_SHARE_GID` — your login account's own UID/GID (`id <user>`)
- `NETWORK_SHARE_FORCE_GID` — the `mediashare` GID from step 0 (**not** the same as above — see Section 0)

```bash
docker compose up -d
docker compose logs -f network-share
```

---

## 3. Verify SMB works, and that writes actually work

Auth alone succeeding is not enough — test a real write, since that's the part
that silently fails if `FORCE_GID` is wrong:

```bash
apt install -y smbclient
smbclient //127.0.0.1/NetworkShare -U <your-smb-username>
```
At the `smb: \>` prompt:
```
put /etc/hostname test-upload.txt
del test-upload.txt
exit
```
If `put` returns `NT_STATUS_ACCESS_DENIED`, see **Troubleshooting → Writes fail
despite correct auth** below — don't start debugging Docker/CIFS/Jellyfin yet,
fix this first since everything downstream depends on it.

---

## 4. Mount it on any other host that needs it (Jellyfin, qBittorrent, etc.)

```bash
apt install -y cifs-utils
```
```bash
cat > /root/.network-share-credentials << 'EOF'
username=<your-smb-username>
password=<same password as .env>
EOF
chmod 600 /root/.network-share-credentials
```
```bash
mkdir -p /mnt/network-share
mount -t cifs //<SHARE-HOST-IP>/NetworkShare /mnt/network-share \
  -o credentials=/root/.network-share-credentials,iocharset=utf8,vers=3.0,cache=loose,rsize=1048576,wsize=1048576,actimeo=30,noperm
```
```bash
echo "//<SHARE-HOST-IP>/NetworkShare /mnt/network-share cifs credentials=/root/.network-share-credentials,iocharset=utf8,vers=3.0,cache=loose,rsize=1048576,wsize=1048576,actimeo=30,noperm,nofail,x-systemd.mount-timeout=30,_netdev 0 0" >> /etc/fstab
```
```bash
systemctl daemon-reload
```

`nofail` + `x-systemd.mount-timeout=30` keep the host booting normally if the
share machine is down or slow, instead of hanging or dropping to emergency mode.

`rsize`/`wsize` at 1MB + `cache=loose` are what actually prevent stutter on
large sequential video reads — not raw bandwidth tuning.

**`noperm` is required, not optional.** Without it, the CIFS client fabricates
its own local permission bits from `uid=`/`gid=`/`file_mode=` instead of
deferring to the server — and since this share uses `force user`/`force group`
server-side (see below), those fabricated client-side bits are actively wrong
and will block writes the server would otherwise allow.

### Adding the mount into a container
Any container reading/writing this mount needs the volume added **and the
container recreated**, not just restarted — `docker restart` keeps the
container's existing mount-namespace reference and won't pick up a mount that
changed on the host afterward:
```bash
docker compose up -d --force-recreate <service-name>
```

Add `:rslave` to the end of the volume line so a mount that appears or is
re-done on the host later (slow boot, share machine restarted) shows up inside
the container instead of leaving it stuck on an empty folder:
```
- /mnt/network-share:/data/network-share:rslave
```

### Surviving reboots (run on the host that mounts the share)
Without this, Docker can start containers before the CIFS mount is up (very
likely if both machines restart together after a power cut). Containers then
bind an empty folder, and Jellyfin can read that as "all media deleted" on its
next scan. Make Docker start only after the network mounts have been attempted:
```bash
mkdir -p /etc/systemd/system/docker.service.d
cat > /etc/systemd/system/docker.service.d/wait-for-share.conf << 'EOF'
[Unit]
After=remote-fs.target
EOF
systemctl daemon-reload
```
Verify with a real test, not by inspection: reboot the share machine, wait for
it, reboot the mounting host, then check:
```bash
mount | grep network-share
docker exec <container> ls <container-mount-path>
```
Also test the bad order: boot the mounting host while the share machine is off,
bring the share machine up, run `mount -a`, and confirm the container sees the
files without being recreated (this is what `:rslave` is for).

Share machine side needs nothing extra: the Samba container uses
`restart: unless-stopped`, and the folder ownership, ACLs and fstab bind mounts
are all on disk. Just confirm `systemctl is-enabled docker` prints `enabled`.

If the target container runs with `read_only: true`, Docker can't create a new
mountpoint directory inside its read-only rootfs. Mount under a path that's
already an existing volume in that container (e.g. `/data/network-share`
instead of inventing `/media/network-share`), not a brand-new top-level path.

---

## 5. Nextcloud — External Storage (UI, not CLI)
- Settings → Apps → enable **External storage support**
- Settings → Administration → External Storage → Add storage → **SMB / CIFS**
- Host: `<SHARE-HOST-IP>` · Share: `NetworkShare` · Auth: Username/password

## 6. Jellyfin
Add the mounted path (e.g. `/data/network-share`) as a library folder in
Dashboard → Libraries → Add Media Library.

## 7. FileZilla / SFTP access to the real files
```bash
mkdir -p /srv/network-share
mount --bind /root/data/NetworkShare /srv/network-share
echo "/root/data/NetworkShare /srv/network-share none bind 0 0" >> /etc/fstab
```
Bind mount, not a symlink — a symlink pointing outside a user's home directory
can silently fail to resolve depending on SFTP client/session behavior; a bind
mount is part of the real filesystem tree and always works.

For a shortcut on first login:
```bash
mkdir -p /home/<user>/NetworkShare
mount --bind /srv/network-share /home/<user>/NetworkShare
echo "/srv/network-share /home/<user>/NetworkShare none bind 0 0" >> /etc/fstab
```

### Recovering/permanently deleting files via FileZilla
This share's recycle-bin module means normal "delete" doesn't free space — it
moves files into a hidden `.deleted` folder (mirroring your directory
structure, no size limit, nothing auto-expires). To reach it: enable **Server →
Force showing hidden files** in FileZilla, then delete from inside
`NetworkShare/.deleted` to actually free space. If that folder looks
inaccessible, re-run the `setfacl` commands from Section 1 — see the known gap
noted there.

---

## Troubleshooting

### Writes fail despite correct auth (`NT_STATUS_ACCESS_DENIED` on `put`)
Check what's actually forced:
```bash
docker exec network-share grep -E "force user|force group" /etc/samba/smb.conf
```
If `force group` doesn't match your real shared group's GID (`getent group
<name>`), that's it — fix `NETWORK_SHARE_FORCE_GID` in `.env` and
`docker compose up -d --force-recreate`.

### Why `force user`/`force group` exist at all
The base image's default `smb.conf` ships with `force user = smbuser` /
`force group = smb` baked in — meaning by default, every SMB write happens on
disk as that built-in identity regardless of who authenticated, not as your
real account. Your real user can own the folder, authenticate successfully,
and still get denied, because Samba is writing as a completely different
identity than the one that just logged in. This compose overrides both to
point at your real account and real shared group instead.

### A container can read the mount but can't write to it
Check the mount's actual live options, not what you think you set:
```bash
docker exec <container> cat /proc/mounts | grep network-share
```
If you see `uid=0,gid=0,file_mode=0755` even after adding `noperm` to your
`mount` command, the container is looking at a stale mount from before you
remounted — force-recreate it (Section 4).

### FileZilla isn't showing a file/folder that's confirmed present server-side
Before assuming it's permissions: test with a completely different client
first (native `sftp <user>@<host>` or Windows' built-in `sftp` command) to
rule out FileZilla-side caching/filtering. If the native client sees it and
FileZilla doesn't, check FileZilla's filter icon (funnel, in the toolbar) isn't
silently excluding it, and if that's not it, delete the saved Site Manager
entry entirely and reconnect via a fresh Quickconnect rather than reusing the
existing session/tab.

### Jellyfin container fails to start after adding the mount: `read-only file system`
The container has `read_only: true` set. See Section 4 — mount under an
existing volume path, don't add a brand-new top-level directory.
