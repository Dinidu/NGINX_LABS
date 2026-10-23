# SOP-02.01: 🏭 PROD ONLY: Install a New Node and Sync it with nginx-sync as `sysadmin`

| | |
|---|---|
| Project | CLIENT NGINX Plus: additional nodes (2 → 6) |
| Replaces | SOP-02 for production. Same flow, but nginx-sync connects as **`sysadmin` with sudo**, not as root. CLIENT does not allow root SSH between nodes, including for the existing Node 1 → Node 2 sync |
| Runs on | **All nodes:** Node 1 (<NODE1_IP>, sync primary), Node 2, and Nodes 3–6 |
| Input | NGINX Plus installed on the new nodes (SOP-04) or the offline bundle, the new nodes' `license.jwt`, CLIENT approvals (section 0) |
| Status | Install, licence, start and test steps are from SOP-02 (validated on `lab-n2`, 16 Oct 2026). The `sysadmin` sections (2, 6, 8, 9) are written from F5's `/usr/bin/nginx-sync.sh` (nginx-sync 1.1). ⚠️ **They have not yet been run in the lab.** Rehearse them on `lab-n1` / `lab-n2` before using them on CLIENT nodes |

Run commands as **root** (`sudo -i`) unless a step says **as `sysadmin`**. Paste **one line at a time**, and check the prompt before every step: `[root@<host> ~]#` or `[sysadmin@<host> ~]$`. Keep one terminal per server.

If SOP-04 and SOP-05 are already done on a new node, skip the steps they cover (marked ⏭️).

---

## 0. How nginx-sync works as `sysadmin`, and what CLIENT must approve

When `nginx-sync.sh` is started by a **non-root** user, it automatically switches to sudo mode (`/usr/bin/nginx-sync.sh`, `set_sudo_prefix`):

| What | Runs as | Needs |
|---|---|---|
| Remote commands (`rsync --version`, `nginx -t`, `test`, `mkdir`, `rm`, `service nginx reload`…) | `ssh -t sysadmin@<peer> "sudo <cmd>"` | `sysadmin` key login to the peer. `sysadmin`'s `known_hosts` on Node 1. Passwordless sudo on the peer |
| Config copy (`rsync --delete`) | `sudo rsync` **as root on Node 1**, `ssh -i /home/sysadmin/.ssh/id_rsa sysadmin@<peer>`, and `sudo rsync` on the peer | The key at exactly `/home/sysadmin/.ssh/id_rsa` (RSA). **Root's** `known_hosts` on Node 1 must know the peer. Passwordless sudo on both ends |
| Local checks and backups (`nginx -t`, `/var/lib/nginx-sync`) | `sudo` on Node 1 | Passwordless sudo on Node 1 |
| Lock file `/tmp/nginx-sync.lock`, log file (`-l`) | `sysadmin` **without** sudo | The log file writable by `sysadmin`. No stale root-owned lock file |

Run it **as `sysadmin` and without `sudo`** (`su - sysadmin`, then `nginx-sync.sh …`). `sudo nginx-sync.sh` runs as root and goes back to root SSH.

**For CLIENT's security team (be explicit):** F5's script itself says that running as non-root with sudo *"is no more secure than running as root"*. Passwordless `sudo rsync`, `rm` or `sed` can change any file. What CLIENT gains: **no root SSH login**, every action logged by sudo under a named account, and access revocable by disabling one account. What does not change: **`sysadmin` is root-equivalent on all six nodes**, and its private key on Node 1 must be protected as a root credential.

**Approvals needed before section 2 (record them):**
1. The `sysadmin` account on all six nodes (CLIENT may already have one: see 2.1).
2. The sudoers rule in 2.2 on all six nodes.
3. `sysadmin` key login from Node 1 to Nodes 2–6.
4. A change window for switching the **existing** Node 1 → Node 2 sync to `sysadmin` (section 2 on Node 2, then section 9).

**Never:**
- ❌ Copy Node 1's `license.jwt` to a new node.
- ❌ `setenforce 0`, or change any SELinux mode, without CLIENT's written approval.
- ❌ Remove the existing root path (section 9) before the `sysadmin` path has passed a real sync to every node.

---

## 1. Check each new node

⏭️ Done if SOP-04 section 2 and SOP-05 section 2 are done.

**1.1 Starting point** (on the new node):

```bash
hostname; cat /etc/redhat-release; uname -m
rpm -q nginx-plus nginx-sync
ls /etc/yum.repos.d/ | grep -i nginx || echo "no nginx repo - good"
ls -l /sbin/service
```

✅ Expect: the same RHEL minor release as Node 1, `x86_64`, and either `not installed` (install in section 4) or the same build as Node 1 (installed by SOP-04). `/sbin/service` must be present (nginx-sync uses it to reload).

**1.2 SELinux on Node 1 and on the new node:**

```bash
getenforce
grep -E '^SELINUX=' /etc/selinux/config
getsebool httpd_can_network_connect
```

If `getenforce` and the config differ, treat the **config value** as the real mode, and raise the difference with CLIENT. On **Node 1** only, also record the local customisations:

```bash
semanage boolean -l -C
semanage port -l -C
semanage fcontext -l -C
```

**1.3 Decide what to do with SELinux** (Node 1 mode, then new node mode):

| # | Node 1 | New node | Do | Don't |
|---|---|---|---|---|
| A | Enforcing | Enforcing | ✅ Proceed. Copy Node 1's settings (`setsebool -P httpd_can_network_connect 1` if Node 1 is `on`). `restorecon` after every copy | Skip `restorecon` |
| B | Enforcing | Permissive | ⚠️ Raise with CLIENT. Apply everything from A. Before handover, fix would-be denials (`ausearch -m avc -ts today`) | Switch modes without approval |
| C | Enforcing | Disabled | ⚠️ Raise with CLIENT (re-enabling needs a relabel and reboot). If told to proceed, skip `setsebool` / `restorecon` and record a deviation | Re-enable SELinux yourself |
| D | Permissive | Enforcing | ✅ Proceed. Apply A as a minimum. Test every UAT case on the new node directly, then check `ausearch -m avc -ts recent` | Switch the new node to Permissive |
| E | Permissive | Permissive | ✅ Proceed. Apply A anyway | Change modes now |
| F | Permissive | Disabled | ⚠️ As C | Re-enable SELinux yourself |
| G | Disabled | Enforcing | ✅ As D | Disable it to match Node 1 |
| H | Disabled | Permissive | ✅ As E | Change modes |
| I | Disabled | Disabled | ✅ Proceed. Skip `setsebool` / `restorecon` | Enable SELinux now |

A denial: `ausearch -m avc -ts recent | audit2why`. The usual fixes are `setsebool -P httpd_can_network_connect 1`, or `restorecon -Rv /etc/nginx` (wrong label after a copy). ❌ Never fix a denial by disabling SELinux.

---

## 2. Set up the `sysadmin` sync path

Do 2.1–2.2 on **every node** (Node 1, Node 2, and each new node). Do 2.3–2.8 on **Node 1**, once per peer: **Node 2 first** (so the existing sync is proven), then each new node. The root path keeps working throughout, so you can roll back.

**2.1 On every node: check whether `sysadmin` exists and what it may do:**

```bash
id sysadmin
sudo -l -U sysadmin
```

- `no such user`: create it, if CLIENT approves: `useradd -m -c "nginx-sync service account" sysadmin`.
- The user exists and `sudo -l` shows `(ALL) NOPASSWD: ALL`: CLIENT already grants full passwordless sudo. The rule in 2.2 is not needed. Record this.
- The user exists and sudo needs a password (no `NOPASSWD`): add the rule in 2.2. nginx-sync runs unattended and cannot answer a password prompt.

**2.2 On every node: install the sudoers rule (if 2.1 says it is needed).** Paste the whole block, including the closing `EOF` line:

```bash
cat > /tmp/nginx-sync.sudoers <<'EOF'
# nginx-sync: commands the sync user may run as root without a password.
# Both /bin and /usr/bin forms are listed because the script calls /bin/... paths.
Cmnd_Alias NGINX_SYNC = /usr/bin/rsync, /usr/sbin/nginx, \
    /sbin/service nginx *, /usr/sbin/service nginx *, \
    /usr/bin/test, /bin/test, /bin/mkdir, /usr/bin/mkdir, /bin/chown, /usr/bin/chown, \
    /bin/chmod, /usr/bin/chmod, /bin/rm, /usr/bin/rm, /bin/mv, /usr/bin/mv, \
    /bin/sed, /usr/bin/sed, /usr/bin/diff
sysadmin ALL=(root) NOPASSWD: NGINX_SYNC
EOF
```

Check the syntax **before** installing it (a broken sudoers file can break sudo for everyone):

```bash
visudo -cf /tmp/nginx-sync.sudoers
```

```bash
install -m 0440 -o root -g root /tmp/nginx-sync.sudoers /etc/sudoers.d/nginx-sync
```

```bash
visudo -c; sudo -l -U sysadmin | tail -6
```

✅ Expect: `parsed OK` (twice), and the `NOPASSWD` command list. If CLIENT's own sudoers names `sysadmin` too, show them this file and agree the final rule with them.

**2.3 On Node 1: create `sysadmin`'s RSA key** (if it does not exist). The file name **must** be `id_rsa`, because the script hard-codes `$HOME/.ssh/id_rsa` for the config copy:

```bash
su - sysadmin -c 'ls ~/.ssh/id_rsa.pub 2>/dev/null || ssh-keygen -t rsa -b 4096 -N "" -C "nginx-sync@$(hostname)" -f ~/.ssh/id_rsa'
```

```bash
cat /home/sysadmin/.ssh/id_rsa.pub
```

If CLIENT's policy forbids keys without a passphrase, raise it. nginx-sync cannot answer a passphrase prompt.

**2.4 On the peer: authorise the key for `sysadmin`:**

```bash
install -d -m 700 -o sysadmin -g sysadmin /home/sysadmin/.ssh
cp -n /home/sysadmin/.ssh/authorized_keys /home/sysadmin/.ssh/authorized_keys.bak 2>/dev/null
echo "<PASTE THE ssh-rsa LINE FROM 2.3>" >> /home/sysadmin/.ssh/authorized_keys
chown sysadmin:sysadmin /home/sysadmin/.ssh/authorized_keys; chmod 600 /home/sysadmin/.ssh/authorized_keys
restorecon -Rv /home/sysadmin/.ssh
```

Check that the peer allows `sysadmin` to log in with a key: `sshd -T | grep -iE '^(pubkeyauthentication|allowusers|allowgroups|denyusers)'`. If `AllowUsers` / `AllowGroups` is set, `sysadmin` must be in it (CLIENT change).

**2.5 On Node 1: accept the peer's host key as `sysadmin` AND as root.** Two different `known_hosts` files are used (section 0). Run each command **on its own** and type `yes` at the prompt:

As `sysadmin`:

```bash
su - sysadmin -c 'ssh sysadmin@<PEER> hostname'
```

As root, with `sysadmin`'s key (this is the path the config copy uses):

```bash
ssh -i /home/sysadmin/.ssh/id_rsa sysadmin@<PEER> hostname
```

✅ Expect: the peer's hostname both times, **with no password prompt**. Use the **same name or IP** here as in `NODES` (section 6): `known_hosts` entries are per name.

**2.6 On Node 1: prove passwordless sudo on the peer.** `sudo -n` fails instead of prompting:

```bash
su - sysadmin -c 'ssh sysadmin@<PEER> "sudo -n /usr/bin/rsync --version | head -1; sudo -n /usr/sbin/nginx -v; sudo -n /usr/bin/test -d /etc/nginx && echo test-ok; sudo -n /sbin/service nginx status >/dev/null; echo service-rc=\$?"'
```

✅ Expect: an rsync version, the nginx version, `test-ok`, and `service-rc=0` (NGINX running) or `service-rc=3` (NGINX stopped, normal for a new node before section 5). 🛑 `sudo: a password is required`: the rule from 2.2 is missing or does not match on that peer.

**2.7 On Node 1: prove passwordless sudo locally:**

```bash
su - sysadmin -c 'sudo -n /usr/sbin/nginx -t && sudo -n /usr/bin/test -d /var/lib/nginx-sync && echo local-ok'
```

✅ Expect: `test is successful`, then `local-ok`.

**2.8 On Node 1 (once): log file and lock file.** The log is written by `sysadmin` without sudo:

```bash
ls -l /var/log/nginx-sync.log /tmp/nginx-sync.lock 2>&1
```

```bash
touch /var/log/nginx-sync.log; chown sysadmin:sysadmin /var/log/nginx-sync.log; chmod 0640 /var/log/nginx-sync.log; restorecon -v /var/log/nginx-sync.log
```

If `/tmp/nginx-sync.lock` exists and no sync is running (`pgrep -f nginx-sync.sh` shows nothing), remove it: `rm -f /tmp/nginx-sync.lock`. A lock file left by root blocks `sysadmin`.

**2.9 Existing Node 2 only: prove the `sysadmin` path before anything else.** On Node 1, as `sysadmin`:

```bash
su - sysadmin
```

```bash
nginx-sync.sh -C
```

✅ Expect: the `Comparing configs on master node and sysadmin@<NODE2>:` header with **no differences** and no `Permission denied`. 🛑 Differences: someone changed Node 2 directly. Stop and investigate before any sync. Type `exit` to return to root.

---

## 3. Copy the bundle to the new node

⏭️ Skip if NGINX Plus is already installed by SOP-04.

`sysadmin` cannot write to `/root/` on the peer, so copy to `sysadmin`'s home, then move it as root. On **Node 1**:

```bash
su - sysadmin -c 'scp /tmp/<BUNDLE>.tar.gz /tmp/<BUNDLE>.tar.gz.sha256 sysadmin@<NEW_NODE>:~/'
```

(Place the files in Node 1's `/tmp/` first, readable by `sysadmin`, or use CLIENT's approved transfer method.) Then on the **new node**:

```bash
mv /home/sysadmin/<BUNDLE>.tar.gz* /root/
cd /root
sha256sum -c <BUNDLE>.tar.gz.sha256
```

✅ Expect: `OK`. **If it fails, do not unpack.**

---

## 4. Install offline and place the licence

On the **new node**.

**4.1 to 4.3: install.** ⏭️ Skip if SOP-04 installed it. Otherwise:

```bash
cd /root && tar xzf <BUNDLE>.tar.gz
cat /root/nginx-bundle/VERSIONS.txt
rpm --import /root/nginx-bundle/nginx_signing.key
rpm -K /root/nginx-bundle/rpms/*.rpm | grep -v 'digests signatures OK' || echo "ALL SIGNATURES OK"
```

```bash
NGX_VER=1:<BUILD_FROM_NODE1>
dnf --disablerepo='*' --repofrompath=offline,/root/nginx-bundle/rpms --enablerepo=offline --setopt=offline.gpgcheck=1 install --assumeno nginx-plus-$NGX_VER nginx-sync
```

🛑 Stop if there is an `Upgrading:` section with core packages, or a missing dependency. Otherwise run the same command with `-y` in place of `--assumeno`, then:

```bash
rpm -q nginx-plus nginx-sync; nginx -v; ls -l /sbin/service
```

✅ Expect: the same build and `nginx -v` as Node 1, and `/sbin/service` present.

**4.4 Place the new node's licence.** Use the **new licence for the additional nodes**, from CLIENT, into `/root/license.jwt`. ❌ **Never copy Node 1's.**

```bash
install -m 0644 -o root -g root /root/license.jwt /etc/nginx/license.jwt
restorecon -v /etc/nginx/license.jwt
python3 -c "import base64,json;p=open('/etc/nginx/license.jwt').read().split('.')[1];d=json.loads(base64.urlsafe_b64decode(p+'='*(-len(p)%4)));print({k:d.get(k) for k in ('sub','f5_order_type','f5_sat')})"
```

Run the same `python3` command on **Node 1**. ✅ The `sub` **must be different** from Node 1's (it proves the new subscription). `f5_order_type` must not be `eval`.

---

## 5. Configure and start for the first time

On the **new node** unless stated.

**5.1 SELinux and firewall.** ⏭️ Done if SOP-05 section 4 is done. For rows A, B, D, E, G, H:

```bash
setsebool -P httpd_can_network_connect 1
firewall-cmd --permanent --add-service=http; firewall-cmd --reload; firewall-cmd --list-services
```

**5.2 Name resolution and the default site.** ⏭️ Done if SOP-05 is done:

```bash
getent hosts <COHESITY_HOST>
rm -f /etc/nginx/conf.d/default.conf; ls /etc/nginx/conf.d/
```

✅ Expect: the same address as on Node 1, and an empty `conf.d`.

**5.3 Give the new node Node 1's `nginx.conf`** (with the `mgmt {}` block, so the first usage report goes to CLIENT's NIM). ⏭️ Done if SOP-05 4.6 is done. On **Node 1**:

```bash
su - sysadmin -c 'scp /etc/nginx/nginx.conf sysadmin@<NEW_NODE>:~/nginx.conf'
```

On the **new node**:

```bash
install -m 0644 -o root -g root /home/sysadmin/nginx.conf /etc/nginx/nginx.conf && rm -f /home/sysadmin/nginx.conf
```

**5.4 Test and start:**

```bash
restorecon -Rv /etc/nginx
nginx -t
systemctl enable --now nginx || { sleep 2; systemctl start nginx; }
systemctl is-active nginx
```

✅ Expect: `test is successful`, then `active`. ⚠️ Known behaviour: the **first** start after a fresh install fails with `open() "/run/nginx.pid" failed (13: Permission denied)`. The retry in the command handles it. No SELinux change is needed. If the second start also fails: `journalctl -u nginx -n 20 --no-pager` and `ausearch -m avc -ts recent`.

**5.5 Licence and usage reporting:**

```bash
sleep 10
grep -iE 'emerg|license|usage|mgmt' /var/log/nginx/error.log | tail -5
ls -l /var/lib/nginx/state/
```

✅ Expect: no licence errors (only the first-start pid line), and `nginx-mgmt-state` present.

---

## 6. Add the new node to nginx-sync and sync, as `sysadmin`

**6.1 On Node 1, as root: back up and edit `/etc/nginx-sync.conf`.** Do not overwrite it:

```bash
cp -p /etc/nginx-sync.conf /etc/nginx-sync.conf.bak.$(date +%F-%H%M)
cat /etc/nginx-sync.conf
vi /etc/nginx-sync.conf
```

1. Append the new node to `NODES`, using the **same name or IP as in 2.5** (for example `NODES="<NODE2> <NODE3>"`). Add nodes one at a time. Do **not** write `sysadmin@` in `NODES`: the script adds it.
2. `EXCLUDE` must contain `license.jwt` (for example `EXCLUDE="default.conf license.jwt"`).
3. If `CONFPATHS` is `/etc/nginx`, the `license.jwt` exclusion is essential. Otherwise every node gets Node 1's licence. Don't change `CONFPATHS` without CLIENT.

**6.2 On Node 1, as `sysadmin`: compare, sync, compare:**

```bash
su - sysadmin
```

```bash
nginx-sync.sh -C
```

✅ Expect: `Only in local:` lines for the files the new node does not have yet, and **no** differences for any node already in sync.

```bash
nginx-sync.sh -l /var/log/nginx-sync.log
```

✅ Expect, for each node: `Testing nginx config on sysadmin@<node>`, `test is successful`, then `Redirecting to /bin/systemctl reload nginx.service`. **None** of: `skipping reload`, `restoring previous config`, `Failed to`, `Permission denied`, `password is required`.

```bash
nginx-sync.sh -C
```

✅ Expect: only the comparing headers, with no differences. Type `exit` to return to root.

`Pseudo-terminal will not be allocated` warnings are harmless (the script uses `ssh -t`).

**6.3 On the new node: check what arrived:**

```bash
ls -RZ /etc/nginx/conf.d /etc/nginx/snippets /etc/nginx/jwt
ls -l /etc/nginx/conf.d/
python3 -c "import base64,json;p=open('/etc/nginx/license.jwt').read().split('.')[1];print(json.loads(base64.urlsafe_b64decode(p+'='*(-len(p)%4)))['sub'])"
curl -s http://127.0.0.1/health; echo
```

✅ Expect: the synced files present, **owned by root** and labelled `httpd_config_t`. The licence `sub` **unchanged** from 4.4. `{"status":"UP",...}`.

---

## 7. Test and verify

**7.1 Test tokens.** Use the tokens supplied by CLIENT's application team. Put them in `/root/tokens.env` on the new node (`USER_TOKEN=…`, `ADMIN_TOKEN=…`, `EXPIRED_TOKEN=…`, `REVOKED_TOKEN=…`). Delete the file after testing.

**7.2 Identical configuration (TC_008).** On **Node 1 and the new node**:

```bash
nginx -T 2>/dev/null | sha256sum
```

✅ Expect: the same hash on both.

**7.3 JWT tests on the new node (TC_016–TC_028):**

```bash
source /root/tokens.env
N="-H Host:<NTFY_HOST>"; O="-H Host:<OBJSTG_HOST>"; U=http://127.0.0.1
c(){ curl -s -o /dev/null -w "%{http_code}\n" "$@"; }
```

```bash
echo -n "TC_016 ntfy no token (401): ";        c $N $U/
echo -n "TC_017 ntfy token in query (200): ";  c $N "$U/?token=$USER_TOKEN"
echo -n "TC_018 ntfy token in header (200): "; c $N -H "Authorization: Bearer $USER_TOKEN" $U/
echo -n "TC_019 ntfy expired (401): ";         c $N "$U/?token=$EXPIRED_TOKEN"
echo -n "TC_021 ntfy revoked tid (403): ";     c $N "$U/?token=$REVOKED_TOKEN"
echo -n "TC_022 ntfy POST, no token (200): ";  c $N -X POST -d test $U/
echo -n "TC_024 objstg no token (401): ";      c $O $U/
echo -n "TC_025 objstg user token (200): ";    c $O "$U/?token=$USER_TOKEN"
echo -n "TC_026 objstg admin token (200): ";   c $O "$U/?token=$ADMIN_TOKEN"
echo -n "TC_027 objstg expired (401): ";       c $O "$U/?token=$EXPIRED_TOKEN"
echo -n "TC_028 objstg PUT, no token (200): "; c $O -X PUT -d test $U/bucket/obj
```

```bash
curl -si -N --max-time 3 $N -H "Connection: Upgrade" -H "Upgrade: websocket" -H "Sec-WebSocket-Version: 13" -H "Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==" "$U/?token=$USER_TOKEN" | head -1
```

✅ Expect: every code matches its bracket, and `HTTP/1.1 101 Switching Protocols`. TC_020 (bad signature) is lab-only. Confirm backend behaviour with the backend teams' logs.

**7.4 Over the network, as the F5 will (TC_010).** On **Node 1**:

```bash
curl -s http://<NEW_NODE_IP>/health; echo
curl -s -o /dev/null -w "%{http_code}\n" -H "Host: <OBJSTG_HOST>" "http://<NEW_NODE_IP>/?token=$USER_TOKEN"
```

✅ Expect: UP, and `200`.

**7.5 Licence (TC_005, TC_035):**

```bash
curl -s http://127.0.0.1/api/9/license | jq '{eval, active_till, reporting}'
```

✅ Expect: `eval: false`, `active_till` = the new subscription's end date, `reporting.healthy: true`, `fails: 0`.

**7.6 Change propagation, including deletion (TC_009).** Tell CLIENT first (it touches every node in `NODES`). On **Node 1**, as root:

```bash
echo "# sync test $(date -u +%FT%TZ)" > /etc/nginx/conf.d/zz-sync-test.conf
```

As `sysadmin` (`su - sysadmin`):

```bash
nginx-sync.sh -l /var/log/nginx-sync.log | tail -4
ssh sysadmin@<NEW_NODE> 'cat /etc/nginx/conf.d/zz-sync-test.conf'
```

Back as root (`exit`), remove it, then sync again as `sysadmin`:

```bash
rm -f /etc/nginx/conf.d/zz-sync-test.conf
```

```bash
su - sysadmin -c 'nginx-sync.sh -l /var/log/nginx-sync.log | tail -4; ssh sysadmin@<NEW_NODE> "ls /etc/nginx/conf.d/"'
```

✅ Expect: the test line on the new node after the first sync, and **gone** after the second.

**7.7 Reboot test (TC_032).** On the new node: `reboot`. After about a minute: `systemctl is-active nginx; curl -s http://127.0.0.1/health; echo`. ✅ `active` and UP.

When section 7 passes: **Phase 3**. Add the node to the F5 pools **disabled**, test, then enable. Repeat sections 1–7 for each new node (2.1–2.2 and 2.4–2.6 per node).

---

## 8. Running nginx-sync from now on

Always: log in to Node 1, `su - sysadmin`, then `nginx-sync.sh -C` before and after `nginx-sync.sh -l /var/log/nginx-sync.log`. Config changes are still made as root on Node 1. Only the sync runs as `sysadmin`.

---

## 9. Retire the root SSH path (after all nodes sync as `sysadmin`)

Only after section 6 has synced **every** node in `NODES` as `sysadmin`, and with CLIENT's approval.

**9.1 On each peer (Node 2–6): remove Node 1's root key from root's `authorized_keys`:**

```bash
cp -p /root/.ssh/authorized_keys /root/.ssh/authorized_keys.bak.$(date +%F)
grep -n 'nginx-sync@' /root/.ssh/authorized_keys
```

Delete only the line(s) for Node 1's root key (comment `nginx-sync@<node1>` or matching `cat /root/.ssh/id_ed25519.pub` / `id_rsa.pub` on Node 1), with `vi /root/.ssh/authorized_keys`.

**9.2 On Node 1: confirm root no longer gets in, and `sysadmin` still does:**

```bash
ssh -o BatchMode=yes root@<PEER> true; echo "root-rc=$?"
su - sysadmin -c 'nginx-sync.sh -C'
```

✅ Expect: `root-rc=255` (refused), and a clean compare as `sysadmin`.

Whether to disable root SSH entirely (`PermitRootLogin no`) or delete root's key pair on Node 1 is CLIENT's own hardening decision. It is not part of this change.

---

## Rollback

| Situation | Action |
|---|---|
| `sysadmin` sync fails, root path still in place | On Node 1, as root: `nginx-sync.sh -C` / `nginx-sync.sh -l /var/log/nginx-sync.log`. It uses root SSH as before. Fix the `sysadmin` path at leisure |
| Root path already retired (section 9) | On the peer: `\cp /root/.ssh/authorized_keys.bak.<date> /root/.ssh/authorized_keys` (the `\` bypasses RHEL's `cp -i` alias), then sync as root |
| Remove the `sysadmin` setup | On each node: `rm -f /etc/sudoers.d/nginx-sync && visudo -c`, and remove the key line from `/home/sysadmin/.ssh/authorized_keys` |
| A sync broke a peer | nginx-sync restores the peer's previous config automatically when its `nginx -t` or reload fails. Older copies are in `/var/lib/nginx-sync/0…9` on Node 1 |

---

## Record sheet

| Item | Node 2 | Node 3 | Node 4 | Node 5 | Node 6 |
|---|---|---|---|---|---|
| CLIENT approvals (section 0), ref / date | | | | | |
| `sysadmin` existed / created; sudo as found (2.1) | | | | | |
| Sudoers rule installed, `parsed OK` (2.2) | | | | | |
| Key authorised (2.4) | | | | | |
| Host key accepted as `sysadmin` + root (2.5) | | | | | |
| `sudo -n` checks OK on peer (2.6) | | | | | |
| `nginx-sync.sh -C` as `sysadmin` clean (2.9 / 6.2) | | | | | |
| RHEL / SELinux row (1.1–1.3) | | | | | |
| Build / `nginx -v` match Node 1 (4) | | | | | |
| Licence `sub` differs from Node 1, not eval (4.4) | | | | | |
| First start needed retry (5.4) | | | | | |
| Sync as `sysadmin` OK, no errors (6.2) | | | | | |
| Config hash = Node 1 (7.2) | | | | | |
| JWT tests TC_016–028 (7.3) / 101 | | | | | |
| Network test (7.4) / licence healthy (7.5) | | | | | |
| Change + delete propagation (7.6) | | | | | |
| Reboot (7.7) | | | | | |
| Root key removed, `root-rc=255` (9) | | | | | |

Node 1 (once): `sysadmin` key created (2.3) ☐ · local `sudo -n` OK (2.7) ☐ · log file owned by `sysadmin`, no stale lock (2.8) ☐ · `nginx-sync.conf` backup name (6.1): ________
