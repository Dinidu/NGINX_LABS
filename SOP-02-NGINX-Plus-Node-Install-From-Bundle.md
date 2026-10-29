# SOP-02: Install and Start an NGINX Plus Node from the Offline Bundle

| | |
|---|---|
| Project | CLIENT NGINX Plus: additional nodes (2 → 6) |
| Phase | 2 of 4: install each new node, give it Node 1's configuration and its own licence, and start it. (1: SOP-01 bundle. 3: SOP-03 nginx-sync and tests. 4: SOP-04 NIM) |
| Runs on | Each **new node** (Nodes 3–6, no internet), plus a few commands on **Node 1** (<NODE1_IP>) |
| Input | The offline bundle (`nginx-plus-<build>-el9-offline.tar.gz` + `.sha256`, SOP-01), and the new nodes' `license.jwt` from CLIENT |
| Result | NGINX Plus running on the new node with Node 1's exact configuration and the new subscription's licence |
| Status | Install, licence and start steps validated in the Azure lab (16 Oct 2026). The manual config copy (4.3–4.5) replaces the earlier copy over root SSH, because CLIENT does not allow root SSH between nodes |

---

## How you work on CLIENT nodes

SSH in as **`<ADMIN_USER>`**, then become root with `sudo su` (no password is asked). Run every command **as root** unless a step says **as `<ADMIN_USER>`**. Paste **one line at a time**, and check the prompt before every step:

| Prompt | You are |
|---|---|
| `[<ADMIN_USER>@<host> ~]$` | `<ADMIN_USER>` (the SSH login, or after `exit` from root) |
| `[root@<host> <ADMIN_USER>]#` | root, after `sudo su`. The folder stays `/home/<ADMIN_USER>`, so the prompt shows `<ADMIN_USER>` and not `~` |

⚠️ This SOP switches between Node 1 and the new node. **Check the host name in the prompt before every step.** Keep one terminal per server and label the tabs.

**Rules:**
- ❌ **Never** copy Node 1's `license.jwt` to a new node. Nodes 3–6 belong to a different subscription.
- ❌ **Never** run `setenforce 0`, or change the SELinux mode, without CLIENT's written approval.
- ❌ Don't add a new node to the F5 pools in this SOP (Phase 3).

---

## 1. Log in and check the new node

**1.1 Log in** to the new node as `<ADMIN_USER>`, then:

```bash
sudo su
```

**1.2 Check the starting point:**

```bash
hostname
cat /etc/redhat-release
uname -m
rpm -q nginx-plus
ls /etc/yum.repos.d/ | grep -i nginx || echo "no nginx repo - good"
df -h /root /usr /var
```

✅ Expect: the correct hostname, the **same RHEL minor release as Node 1 and the bundle** (`BUILD-INFO.txt` from SOP-01), `x86_64`, `package nginx-plus is not installed`, no nginx repository, and at least 500 MB free in `/root`, `/usr` and `/var`.

**1.3 SELinux.** CLIENT's Node 1 and new nodes all run **Enforcing**. Confirm it on **Node 1 and the new node**:

```bash
getenforce
grep -E '^SELINUX=' /etc/selinux/config
getsebool httpd_can_network_connect
```

✅ Expect on both: `Enforcing`, `SELINUX=enforcing`. Node 1 shows `httpd_can_network_connect --> on` (the new node is set in 4.1).

🛑 If a node shows anything other than `Enforcing` / `enforcing`, stop and raise it with CLIENT before going on.

On **Node 1**, record any local SELinux customisation, so the new nodes can match it:

```bash
semanage boolean -l -C
semanage port -l -C
semanage fcontext -l -C
```

Anything listed besides `httpd_can_network_connect` must be applied to the new node too (4.1). `semanage` is in `policycoreutils-python-utils`. If it is missing, record "not available".

---

## 2. Copy the bundle to the new node

Root SSH between nodes is not allowed, so files travel as `<ADMIN_USER>` into its home folder, and are moved into place as root.

**2.1 Get the bundle onto the new node** with CLIENT's approved transfer method (USB, or a jump host), into `/home/<ADMIN_USER>/`. If the bundle is on Node 1, run on **Node 1**:

```bash
install -m 0644 /root/<BUNDLE>.tar.gz /root/<BUNDLE>.tar.gz.sha256 /tmp/
```

```bash
su - <ADMIN_USER> -c 'scp /tmp/<BUNDLE>.tar.gz /tmp/<BUNDLE>.tar.gz.sha256 <ADMIN_USER>@<NEW_NODE>:~/'
```

```bash
rm -f /tmp/<BUNDLE>.tar.gz /tmp/<BUNDLE>.tar.gz.sha256
```

`scp` may ask for `<ADMIN_USER>`'s password on the new node. That is fine.

**2.2 On the new node: move it into place and verify it before unpacking:**

```bash
mv /home/<ADMIN_USER>/<BUNDLE>.tar.gz* /root/
cd /root
sha256sum -c <BUNDLE>.tar.gz.sha256
ls -lh /root/<BUNDLE>.tar.gz
```

✅ Expect: `<BUNDLE>.tar.gz: OK`. **If the check fails, do not unpack.** Copy the file again.

---

## 3. Unpack, install offline and place the licence

All on the **new node**.

**3.1 Unpack the bundle and check its contents:**

```bash
cd /root
tar xzf <BUNDLE>.tar.gz
cat /root/nginx-bundle/VERSIONS.txt
cat /root/nginx-bundle/BUILD-INFO.txt 2>/dev/null
ls /root/nginx-bundle/rpms/*.rpm | wc -l
```

✅ Expect: `VERSIONS.txt` lists exactly one `nginx-plus-1:<build>` (the same build as `rpm -q nginx-plus` on Node 1) plus `nginx-sync`. The RPM count matches the SOP-01 record sheet. `BUILD-INFO.txt` shows the same RHEL minor release as this node.

**3.2 Import F5's package signing key from the bundle, and check every package:**

```bash
rpm --import /root/nginx-bundle/nginx_signing.key
rpm -K /root/nginx-bundle/rpms/*.rpm | grep -v 'digests signatures OK' || echo "ALL SIGNATURES OK"
```

✅ Expect: `ALL SIGNATURES OK`. 🛑 Any other line: stop, do not install.

**3.3 Dry run.** Set the build from `VERSIONS.txt` first:

```bash
NGX_VER=1:<BUILD>
```

```bash
dnf --disablerepo='*' --repofrompath=offline,/root/nginx-bundle/rpms --enablerepo=offline --setopt=offline.gpgcheck=1 install --assumeno nginx-plus-$NGX_VER nginx-sync initscripts-service
```

✅ Expect: `Installing:` shows `nginx-plus <your build>` (**not** 37.x) and `nginx-sync`, plus possibly a few small dependencies, then `Operation aborted.`

🛑 **STOP if there is an `Upgrading:` section** listing core packages (glibc, openssl, systemd…). The bundle was built on a newer RHEL minor release than this node. Rebuild it with SOP-01 on a matching build server.

If dnf says `No match for argument: initscripts-service`, the bundle predates its addition to SOP-01. Check `ls /sbin/service`: if present, drop `initscripts-service` from the command. If absent, rebuild the bundle (nginx-sync needs `/sbin/service`).

**3.4 Install from the bundle only:**

```bash
dnf --disablerepo='*' --repofrompath=offline,/root/nginx-bundle/rpms --enablerepo=offline --setopt=offline.gpgcheck=1 install -y nginx-plus-$NGX_VER nginx-sync initscripts-service
```

```bash
rpm -q nginx-plus nginx-sync
nginx -v
ls -l /sbin/service
```

✅ Expect: the same `nginx-plus` build and `nginx -v` output as Node 1, and `/sbin/service` present.

**3.5 Put the new node's licence in place. Do not start NGINX yet.**

Copy the **new licence for the additional nodes** (from CLIENT) to `/root/license.jwt` on the new node, then:

```bash
install -m 0644 -o root -g root /root/license.jwt /etc/nginx/license.jwt
restorecon -v /etc/nginx/license.jwt
head -c 3 /etc/nginx/license.jwt; echo
python3 -c "import base64,json;p=open('/etc/nginx/license.jwt').read().split('.')[1];d=json.loads(base64.urlsafe_b64decode(p+'='*(-len(p)%4)));print({k:d.get(k) for k in ('sub','f5_order_type','f5_sat')})"
```

Run the same `python3` command on **Node 1** and compare.

✅ Expect: `eyJ`, then the licence's `sub`, type and end date (`f5_sat`, epoch seconds). **The `sub` must be different from Node 1's**: that proves the new node carries the new subscription. `f5_order_type` must not be `eval`.

---

## 4. Configure and start for the first time

The new node gets **Node 1's complete configuration** in one archive, then starts. CLIENT's configuration is split across files (`nginx.conf` uses variables defined in `conf.d/jwt-config.conf`, which loads files from `snippets/` and `jwt/`). Copying files one by one fails with `unknown "<name>" variable` or `open() "/etc/nginx/..." failed (2: No such file or directory)`, so copy the full set.

**4.1 On the new node: SELinux boolean and firewall:**

```bash
setsebool -P httpd_can_network_connect 1
getsebool httpd_can_network_connect
firewall-cmd --permanent --add-service=http
firewall-cmd --reload
firewall-cmd --list-services
```

✅ Expect: `httpd_can_network_connect --> on`, and `http` in the list. Apply any other customisation recorded on Node 1 in 1.3. If firewalld is not running on Node 1 either, match Node 1 and record it.

**4.2 On the new node: name resolution and the default site:**

```bash
getent hosts <COHESITY_HOST>
rm -f /etc/nginx/conf.d/default.conf
```

✅ Expect: `getent` returns the **same address as on Node 1**. If it returns nothing, raise it with CLIENT before going further: `nginx -t` fails if a backend name does not resolve.

**4.3 On Node 1: list every file the configuration uses.**

The folders nginx-sync keeps in step (these are what you will copy):

```bash
grep -E '^(CONFPATHS|EXCLUDE)' /etc/nginx-sync.conf
```

The config files NGINX actually loads:

```bash
nginx -T 2>/dev/null | sed -n 's/^# configuration file \(.*\):$/\1/p' | sort -u
```

Every other file named in the config (keys, certificates, scripts), including files chosen through a variable or `map`:

```bash
nginx -T 2>/dev/null | grep -vE '^\s*#' | grep -oE '/[A-Za-z0-9._/*-]+\.(jwk|jwt|pem|crt|key|conf|js|json|types|txt|html)\b' | sort -u
```

✅ Expect: every path from the last two commands is under one of the `CONFPATHS` folders, **or** is `/etc/nginx/mime.types` (installed by the package, identical on every node). Typical: `nginx.conf`, `conf.d/jwt-config.conf`, `snippets/errors.conf`, `jwt/es512.jwk`, `jwt/es512-admin.jwk`.

🛑 A path outside `CONFPATHS` (for example a certificate under `/etc/pki`): note it. Add it to the archive in 4.4 (`EXTRA` below), and raise with CLIENT whether nginx-sync's `CONFPATHS` should include it.

**4.4 On Node 1: pack the configuration and copy it to the new node.** The archive never contains `license.jwt` (each node keeps its own) or `default.conf`:

```bash
. /etc/nginx-sync.conf; EXTRA=""; echo "Packing: $CONFPATHS $EXTRA"
```

If 4.3 found files outside `CONFPATHS`, set them before packing, for example `EXTRA="/etc/pki/tls/certs/x.crt"`.

```bash
tar czf /tmp/node1-nginx-conf.tgz --exclude=license.jwt --exclude=default.conf $CONFPATHS $EXTRA
```

```bash
tar tzvf /tmp/node1-nginx-conf.tgz
```

✅ Expect: `Removing leading '/' from member names` (normal), then the list of files, **without** `license.jwt` or `default.conf`.

The archive can contain backend tokens, so only `<ADMIN_USER>` may read it:

```bash
sha256sum /tmp/node1-nginx-conf.tgz > /tmp/node1-nginx-conf.tgz.sha256
chown <ADMIN_USER>:<ADMIN_USER> /tmp/node1-nginx-conf.tgz /tmp/node1-nginx-conf.tgz.sha256; chmod 600 /tmp/node1-nginx-conf.tgz /tmp/node1-nginx-conf.tgz.sha256
```

```bash
su - <ADMIN_USER> -c 'scp /tmp/node1-nginx-conf.tgz /tmp/node1-nginx-conf.tgz.sha256 <ADMIN_USER>@<NEW_NODE>:~/'
```

Keep the archive on Node 1 until all new nodes are done (repeat only the `scp` for the next node), then delete it: `rm -f /tmp/node1-nginx-conf.tgz /tmp/node1-nginx-conf.tgz.sha256`.

**4.5 On the new node: back up, unpack and label:**

```bash
cd /home/<ADMIN_USER>
sed -i 's#/tmp/##' node1-nginx-conf.tgz.sha256; sha256sum -c node1-nginx-conf.tgz.sha256
```

✅ Expect: `node1-nginx-conf.tgz: OK`. **If it fails, do not unpack.** Copy it again.

Back up whatever is in `/etc/nginx` now (including any files copied by hand earlier):

```bash
tar czf /root/etc-nginx-before-node1-conf-$(date +%F-%H%M).tgz /etc/nginx
```

Unpack Node 1's configuration over it, then fix labels:

```bash
tar xzf /home/<ADMIN_USER>/node1-nginx-conf.tgz -C /
restorecon -Rv /etc/nginx
rm -f /etc/nginx/conf.d/default.conf
ls -lZ /etc/nginx/ /etc/nginx/conf.d/ /etc/nginx/snippets/ /etc/nginx/jwt/
```

✅ Expect: the files owned by `root`, labelled `httpd_config_t`, and `license.jwt` still present (it was not in the archive).

If files were copied by hand **before** this step, check nothing extra remains: compare `ls /etc/nginx/conf.d/ /etc/nginx/snippets/ /etc/nginx/jwt/` with the same command on Node 1. Remove any file that Node 1 does not have (it is in the backup above).

Confirm the licence is still the new node's own:

```bash
python3 -c "import base64,json;p=open('/etc/nginx/license.jwt').read().split('.')[1];print(json.loads(base64.urlsafe_b64decode(p+'='*(-len(p)%4)))['sub'])"
```

✅ Expect: the **same `sub` as in 3.5** (different from Node 1's).

Delete the copy in `<ADMIN_USER>`'s home:

```bash
rm -f /home/<ADMIN_USER>/node1-nginx-conf.tgz /home/<ADMIN_USER>/node1-nginx-conf.tgz.sha256
```

**4.6 On the new node: test the configuration:**

```bash
nginx -t
```

✅ Expect: `syntax is ok` and `test is successful`.

If it fails, read the first error line:

| Error | Cause and fix |
|---|---|
| `unknown "<name>" variable` | A config file is missing. Recheck 4.3: a file is outside `CONFPATHS`, or the archive was incomplete (`tar tzf`) |
| `open() "/etc/nginx/..." failed (2: No such file or directory)` | Same: the named file is missing. Add it (4.3 `EXTRA`) and repeat 4.4–4.5 |
| `host not found in upstream "..."` | DNS on this node. Compare `getent hosts <name>` with Node 1 (4.2) |
| A `license` error | `/etc/nginx/license.jwt` missing or wrong (3.5) |
| `Permission denied` | Labels or ownership: `restorecon -Rv /etc/nginx`, then `ls -lZ` |

**4.7 On the new node: start NGINX and check it:**

```bash
systemctl enable --now nginx || { sleep 2; systemctl start nginx; }
systemctl is-active nginx
curl -s http://127.0.0.1/health; echo
```

✅ Expect: `active`, then `{"status":"UP","message":"Nginx is running"}`.

⚠️ **Known behaviour:** the **first** start after a fresh install fails with `open() "/run/nginx.pid" failed (13: Permission denied)` and `Failed with result 'protocol'`. A plain second start succeeds, and later starts and reboots are fine. The command above retries once automatically. **No SELinux change is needed.** If the second start also fails: `journalctl -u nginx -n 20 --no-pager` and `ausearch -m avc -ts recent | audit2why`. Root cause not yet confirmed: raise it with F5 support.

**4.8 On the new node: licence activation and usage reporting:**

```bash
sleep 10
grep -iE 'emerg|license|usage|mgmt' /var/log/nginx/error.log | tail -5
ls -l /var/lib/nginx/state/
curl -s http://127.0.0.1/api/9/license | jq '{eval, active_till, reporting}'
```

✅ Expect:
- No licence errors. The only `emerg` line may be the first-start pid error from 4.7.
- `/var/lib/nginx/state/nginx-mgmt-state` exists: usage reporting has started.
- `eval: false`, `active_till` = the new subscription's end date, and `reporting.healthy: true` with `fails: 0`. If `reporting` is not healthy yet, wait a minute and repeat. It reports to CLIENT's NIM (the `mgmt {}` block in Node 1's `nginx.conf`). Confirm on the NIM side with SOP-04.

**4.9 Same configuration as Node 1.** On **Node 1 and the new node**:

```bash
nginx -T 2>/dev/null | sha256sum
```

✅ Expect: **the same hash on both.** If they differ, compare the file lists (4.5) and the outputs of `nginx -T` to find the difference.

**4.10 SELinux check after start:**

```bash
ausearch -m avc -ts recent 2>/dev/null | grep -i nginx || echo "no nginx denials"
```

✅ Expect: `no nginx denials`.

The node is now running with Node 1's configuration and its own licence. Continue with **SOP-03** (join nginx-sync, full tests, F5 pools), then **SOP-04** (NIM).

---

## Rollback

| Situation | Action |
|---|---|
| `nginx -t` fails after 4.5, and you want the previous state back | `systemctl stop nginx`, then `tar xzf /root/etc-nginx-before-node1-conf-<date>.tgz -C /` and `restorecon -Rv /etc/nginx` |
| Remove the install from the node | `systemctl disable --now nginx`, `dnf history list` to find the install, then `dnf history undo <ID> -y` |

---

## Record sheet (one per node)

| Item | Value |
|---|---|
| Date / performed by | |
| New node hostname / IP | |
| RHEL release: new node / Node 1 / bundle (1.2) | |
| SELinux `Enforcing` on Node 1 and new node (1.3) | |
| Node 1 SELinux customisations besides the boolean (1.3) | |
| Bundle checksum OK on node (2.2) | |
| Signatures OK (3.2) / dry run: correct build, no upgrades (3.3) | |
| `nginx -v` matches Node 1 (3.4) | |
| Licence `sub`: new node / Node 1, different, not eval (3.5) | |
| Files outside `CONFPATHS` (4.3), or none | |
| Config archive checksum OK on node (4.5) | |
| Backup file of the previous `/etc/nginx` (4.5) | |
| `nginx -t` successful (4.6) | |
| First start needed retry (4.7) | |
| Licence: `eval false`, end date, reporting healthy (4.8) | |
| Config hash identical to Node 1 (4.9) | |
| No SELinux denials (4.10) | |
