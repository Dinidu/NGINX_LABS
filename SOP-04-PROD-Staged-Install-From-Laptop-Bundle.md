# SOP-04: 🏭 PROD ONLY: Staged Install from the Laptop Bundle (Licence Pending)

| | |
|---|---|
| Project | CLIENT NGINX Plus: additional nodes (2 → 6) |
| Use when | CLIENT's licence for the new nodes is **not ready yet**, and the new VMs must be prepared now |
| Runs on | Your **laptop**, the existing **Node 1** (<NODE1_IP>), and each **new node** (Nodes 3–6) |
| Input | First choice: existing RPMs found on Node 1 (section P). Fallback: `~/<PROJECT_DIR>/bundles/nginx-plus-r36-9-el9-offline.tar.gz` + `.sha256` on your laptop |
| Result | NGINX Plus and nginx-sync **installed but not started**, no licence on the node. Finish with SOP-02 when the licence arrives |
| Status | Bundle validated in the lab (SOP-01, SOP-02 on `lab-n2`, 16 Oct 2026). This staged flow is a subset of SOP-02 |

This SOP is **production only**. There are no trial steps. Run every command on a server as **root** (`sudo -i`), **one line at a time**, and check the prompt (`[root@<hostname> ~]#`) before each step.

---

## P. Start here: look for existing NGINX Plus RPMs on Node 1

The original install of Node 1 may have left its RPMs (or an offline bundle) on the server. If a complete, matching set is there, use it instead of the laptop bundle. It came from CLIENT's own subscription and is already the exact build Node 1 runs.

Steps P.1–P.3 only read, and change nothing. They were checked on `lab-n1` (20 Oct 2026): they found the SOP-02 bundle folder, with 98 RPMs, `repodata`, one `nginx-plus-1:36-9.el9.ngx`, and all signatures OK. ⚠️ P.4–P.5 were **not run in the lab**.

**P.1 On Node 1: record how NGINX Plus was installed:**

```bash
rpm -q nginx-plus nginx-sync
dnf info --installed nginx-plus | grep -iE '^(version|release|from repo)'
dnf history info nginx-plus | grep -iE 'begin time|command line'
grep -rHiE '^\[|baseurl' /etc/yum.repos.d/
```

What it tells you:
- `From repo : @commandline`: installed from RPM files given on the command line. The `Command Line` from `dnf history` often shows the folder they were in.
- A repo name with a `baseurl=file:///...` line: installed from a local repo folder. That folder is the first place to look.
- Any other name (for example `offline`) with `--repofrompath=<name>,<folder>` in the `Command Line`: installed from that folder, the way SOP-02 does it. Look there first.
- `nginx-plus` (with a `pkgs.nginx.com` baseurl): installed online from F5. There may be no RPM files left on the server.

**P.2 On Node 1: search the server for NGINX RPMs and bundles:**

```bash
find / \( -path /proc -o -path /sys -o -path /run -o -path /dev \) -prune -o \( -name 'nginx-plus*.rpm' -o -name 'nginx-sync*.rpm' -o -name '*nginx*.tar.gz' -o -name '*nginx*.tgz' \) -print 2>/dev/null | xargs -r ls -lh
```

- Ignore `.tgz` / `.tar.gz` files that are config backups (for example `nginx-etc-backup-*.tgz`, a few KB). A package bundle is tens of MB.
- 🔍 Nothing found: **use the laptop bundle.** Go to section 0.
- 🔍 Found: note the **folder** that holds the RPMs (or the archive), and continue with P.3. A `.tar.gz` bundle: unpack a copy in `/tmp` first (`mkdir /tmp/node1-check && tar xzf <ARCHIVE> -C /tmp/node1-check`) and use the folder with the RPMs inside.

**P.3 On Node 1: check the folder is complete and matches.** Set `D` to the folder found in P.2:

```bash
D=<FOLDER_WITH_THE_RPMS>
ls $D/*.rpm | wc -l
ls -d $D/repodata 2>/dev/null || echo "no repodata"
rpm -qp --qf '%{NAME}-%{EPOCH}:%{VERSION}-%{RELEASE}.%{ARCH}\n' $D/nginx-*.rpm
rpm -K $D/*.rpm | grep -v 'digests signatures OK' || echo "ALL SIGNATURES OK"
```

✅ **Use it** only if all are true:
- Exactly one `nginx-plus-1:<build>`, the **same build** as `rpm -q nginx-plus` in P.1 (no R37 or other builds mixed in).
- An `nginx-sync` RPM is there.
- `ALL SIGNATURES OK`.

Otherwise (a lone `nginx-plus` RPM, a different build, failed signatures): **use the laptop bundle.** Go to section 0.

Record the folder, RPM count and result in the record sheet. Then do sections 0–2 as normal (rules, Node 1 go/no-go, new node checks), plus SOP-02 steps 2.1–2.3 (root SSH from Node 1), before P.4.

**P.4 On Node 1: package the folder and copy it to the new node:**

```bash
cd /root
tar czf node1-nginx-rpms.tar.gz -C $D .
sha256sum node1-nginx-rpms.tar.gz > node1-nginx-rpms.tar.gz.sha256
scp node1-nginx-rpms.tar.gz node1-nginx-rpms.tar.gz.sha256 root@<NEW_NODE>:/root/
```

If F5's signing key is not in the folder (`ls $D/*.key`), also copy it from the laptop bundle. On your **laptop**: `cd ~/<PROJECT_DIR>/bundles && tar xzf nginx-plus-r36-9-el9-offline.tar.gz nginx-bundle/nginx_signing.key`, then copy `nginx-bundle/nginx_signing.key` to `/root/` on the new node (same path as section 3.2).

**P.5 On the new node: verify, unpack and install:**

```bash
cd /root
sha256sum -c node1-nginx-rpms.tar.gz.sha256
mkdir -p /root/node1-rpms && tar xzf node1-nginx-rpms.tar.gz -C /root/node1-rpms
rpm --import /root/node1-rpms/*.key 2>/dev/null || rpm --import /root/nginx_signing.key
rpm -K /root/node1-rpms/*.rpm | grep -v 'digests signatures OK' || echo "ALL SIGNATURES OK"
```

✅ Expect: `OK`, then `ALL SIGNATURES OK`. Set the build from P.3 (for example `NGX_VER=1:36-9.el9.ngx`), then pick the case that matches P.3:

The folder **has `repodata`**: run the section 4.3 and 4.4 commands, replacing `/root/nginx-bundle/rpms` with `/root/node1-rpms`.

The folder has **no `repodata`**: install the two RPM files directly (dry run first):

```bash
dnf --disablerepo='*' install --assumeno /root/node1-rpms/nginx-plus-${NGX_VER#*:}.x86_64.rpm /root/node1-rpms/nginx-sync-*.rpm
```

```bash
dnf --disablerepo='*' install -y /root/node1-rpms/nginx-plus-${NGX_VER#*:}.x86_64.rpm /root/node1-rpms/nginx-sync-*.rpm
```

- 🛑 Dry run says `nothing provides ...` or `Problem: ... requires ...`: a dependency is missing from the folder. **Stop** and use the laptop bundle (section 3 onwards).
- 🛑 Dry run shows an `Upgrading:` section with core packages: stop, as in section 4.3.

When the install succeeds, run the checks at the end of **section 4.4** (`rpm -q`, `nginx -v`, `dnf history list`), then continue from **section 4.5**. Skip sections 3, 4.1–4.4 (they are for the laptop bundle).

---

## 0. The bundle and the rules

**The bundle on your laptop:**

| Item | Value |
|---|---|
| File | `nginx-plus-r36-9-el9-offline.tar.gz` (42 MB) |
| SHA256 | `3b7f0de8dec3bb315f203b872c69a9e6c73b7a2ecdc8d58e6741c46a05ce21a5` |
| NGINX Plus build | `nginx-plus-1:36-9.el9.ngx` (R36-p8, `nginx/1.29.3`) |
| nginx-sync | `nginx-sync-1.1-1.el9.ngx` |
| RPM count | 98 |
| Built on | RHEL 9.8, x86_64 (lab build server, 16 Oct 2026) |
| Not included | `BUILD-INFO.txt` and `initscripts-service` (the bundle predates both in SOP-01). See step 2.2 |

**Why it is allowed to use this bundle:** the RPMs are F5's standard signed packages. The trial repo cert/key were only used to **download** them and are not part of the packages. `rpm -K` proves every RPM is genuine (step 4.2). What licenses NGINX Plus to **run** is `license.jwt`, and that will be CLIENT's.

**Rules:**
- ❌ **Never** put the trial `license.jwt` (or Node 1's) on a CLIENT node.
- ❌ **Never** start NGINX on a new node in this SOP. R33+ will not run without a licence.
- ❌ Don't add a new node to nginx-sync `NODES` or to the F5 pools until its licence is in place.
- ✅ Get **written approval from CLIENT** (email is enough) before installing, for example: *"NGINX Plus R36-p8 is installed on the new nodes from an offline bundle built from F5's official repository; all package signatures verified with `rpm -K`; bundle SHA256 3b7f0de8…21a5. CLIENT's licence will be applied before go-live."*

---

## 1. Check the bundle matches Node 1 (go / no-go)

**1.1 On Node 1:**

```bash
hostname
rpm -q nginx-plus nginx-sync
nginx -v
cat /etc/redhat-release
```

✅ **GO** only if all three match:
- `nginx-plus-36-9.el9.ngx.x86_64`
- `nginx version: nginx/1.29.3 (nginx-plus-r36-p8)`
- `Red Hat Enterprise Linux release 9.8`

🛑 **NO-GO**, if any differ. Don't use this bundle:
- Different `nginx-plus` build: rebuild the bundle with SOP-01, setting `NGX_VER=1:<build from rpm -q>`.
- Older RHEL minor release (for example 9.4): the dependencies in this bundle are from 9.8 and would upgrade core packages. Rebuild with SOP-01 on a build server with the **same** minor release as the nodes.
- A rebuild needs repo cert/key. Use CLIENT's own if available, or the trial cert/key from your laptop's `licenses/` folder (download only, never copied to CLIENT servers).

---

## 2. Check each new node

**2.1 On the new node:**

```bash
hostname
cat /etc/redhat-release
uname -m
rpm -q nginx-plus
ls /etc/yum.repos.d/ | grep -i nginx || echo "no nginx repo - good"
df -h /root /usr /var
```

✅ Expect: correct hostname, **RHEL 9.8** (same as Node 1), `x86_64`, `package nginx-plus is not installed`, no nginx repo, and at least 500 MB free in `/root`, `/usr` and `/var`.

**2.2 Check for `/sbin/service`** (nginx-sync needs it to reload this node, and it is **not** in this bundle):

```bash
ls -l /sbin/service
rpm -q initscripts-service
```

- ✅ Present: continue.
- ⚠️ Missing: install it from CLIENT's own RHEL repository / Satellite (`dnf install initscripts-service`), if CLIENT allows. Otherwise rebuild the bundle with SOP-01, which now includes it. Without it, nginx-sync **silently skips the reload** on this node. Record the outcome.

**2.3 SELinux.** Run SOP-02 steps **1.3 and 1.4** on Node 1 and on this node, and record the decision-table row.

---

## 3. Copy the bundle from the laptop to the new node

**3.1 On your laptop: verify the bundle before it leaves:**

```bash
cd ~/<PROJECT_DIR>/bundles
shasum -a 256 -c nginx-plus-r36-9-el9-offline.tar.gz.sha256
```

✅ Expect: `nginx-plus-r36-9-el9-offline.tar.gz: OK`.

**3.2 Copy it**, using whichever path CLIENT allows:

Option 1: the laptop can SSH to the new node directly:

```bash
scp nginx-plus-r36-9-el9-offline.tar.gz nginx-plus-r36-9-el9-offline.tar.gz.sha256 <CLIENT_USER>@<NEW_NODE_IP>:/tmp/
```

Option 2: the laptop can only reach Node 1 (or a jump host). Copy to Node 1's `/tmp/` the same way, then on **Node 1** as root (after SOP-02 section 2, root SSH):

```bash
scp /tmp/nginx-plus-r36-9-el9-offline.tar.gz /tmp/nginx-plus-r36-9-el9-offline.tar.gz.sha256 root@<NEW_NODE>:/root/
```

Option 3: CLIENT's approved USB / file-transfer method, into `/root/` on the new node.

**3.3 On the new node: move it into place and verify again:**

```bash
mv /tmp/nginx-plus-r36-9-el9-offline.tar.gz* /root/ 2>/dev/null
cd /root
sha256sum -c nginx-plus-r36-9-el9-offline.tar.gz.sha256
```

✅ Expect: `OK`. **If it fails, do not unpack.** Copy again.

---

## 4. Unpack, verify signatures, install (do not start)

All on the **new node**.

**4.1 Unpack and check contents:**

```bash
cd /root
tar xzf nginx-plus-r36-9-el9-offline.tar.gz
cat /root/nginx-bundle/VERSIONS.txt
ls /root/nginx-bundle/rpms/*.rpm | wc -l
```

✅ Expect: `nginx-plus-1:36-9.el9.ngx` and `nginx-sync-(none):1.1-1.el9.ngx`, and `98`.

**4.2 Import F5's signing key and verify every RPM:**

```bash
rpm --import /root/nginx-bundle/nginx_signing.key
rpm -K /root/nginx-bundle/rpms/*.rpm | grep -v 'digests signatures OK' || echo "ALL SIGNATURES OK"
```

✅ Expect: `ALL SIGNATURES OK`. RHEL packages are checked against Red Hat's key (already on the server), NGINX packages against F5's key. 🛑 Any other line: **stop** and do not install.

**4.3 Dry run:**

```bash
NGX_VER=1:36-9.el9.ngx
dnf --disablerepo='*' --repofrompath=offline,/root/nginx-bundle/rpms --enablerepo=offline --setopt=offline.gpgcheck=1 install --assumeno nginx-plus-$NGX_VER nginx-sync
```

✅ Expect: `Installing:` shows `nginx-plus 1:36-9.el9.ngx` (**not** 37.x) and `nginx-sync`, perhaps with a few small dependencies, then `Operation aborted.`

🛑 **STOP if there is an `Upgrading:` section** listing core packages (glibc, openssl, systemd…). The node's RHEL release does not match the bundle. Go back to step 1 (NO-GO).

**4.4 Install:**

```bash
dnf --disablerepo='*' --repofrompath=offline,/root/nginx-bundle/rpms --enablerepo=offline --setopt=offline.gpgcheck=1 install -y nginx-plus-$NGX_VER nginx-sync
```

```bash
rpm -q nginx-plus nginx-sync
nginx -v
dnf history list | head -3
```

✅ Expect: `nginx-plus-36-9.el9.ngx.x86_64`, `nginx-sync-1.1-1.el9.ngx.noarch`, `nginx/1.29.3 (nginx-plus-r36-p8)`. Record the **dnf history ID** of the install (top line) for rollback.

**4.5 Make sure NGINX stays stopped:**

```bash
systemctl is-enabled nginx
systemctl is-active nginx
ls -l /etc/nginx/license.jwt 2>/dev/null || echo "no licence - correct for now"
```

✅ Expect: `disabled`, `inactive`, `no licence - correct for now`. Do **not** run `systemctl enable` or `start`. If NGINX were enabled, every reboot would try to start it and log licence errors.

**4.6 Optional: record what NGINX says without a licence:**

```bash
nginx -t
```

Record the output in the record sheet. A licence error is expected. It confirms the nginx-sync push (which runs `nginx -t` on each peer) must wait for the licence.

---

## 5. Prepare everything that does not need the licence

Do these now so only the licence steps are left. All are in SOP-02:

| SOP-02 step | What | Where |
|---|---|---|
| 2.1–2.3 | Root SSH from Node 1 to the new node (CLIENT approval) | Node 1 + new node |
| 4.1 | SELinux boolean and firewall (`http`) | New node |
| 4.2 | `getent hosts` for the object-storage name, and remove `default.conf` | New node |

Then clean up the bundle copy (keep the archive until go-live in case you need to reinstall):

```bash
rm -rf /root/nginx-bundle
ls -lh /root/nginx-plus-r36-9-el9-offline.tar.gz*
```

---

## 6. When CLIENT's licence arrives

Continue with **SOP-02 from step 3.5** for each node:

1. **3.5** Place the new `license.jwt`, and check its `sub` is **different** from Node 1's.
2. **4.3–4.5** Copy Node 1's `nginx.conf`, start NGINX (first-start retry is expected), check licence and usage reporting.
3. **5** Add the node to nginx-sync `NODES` and sync.
4. **6** Run the tests, then Phase 3 (F5 pool, disabled first).

---

## Rollback

To remove the staged install from a new node, using the history ID from 4.4:

```bash
dnf history info <ID>
dnf history undo <ID> -y
rpm -q nginx-plus nginx-sync
```

✅ Expect: both `not installed`. This also removes the dependencies that the install added, and nothing else.

---

## Record sheet (one per node)

| Item | Value |
|---|---|
| Date / performed by | |
| Node 1 install source: `From repo` / baseurl (P.1) | |
| Existing RPMs found on Node 1: folder / RPM count / signatures (P.2–P.3) | |
| Source used: Node 1 RPMs (P.4–P.5) or laptop bundle (3–4) | |
| CLIENT written approval to install from bundle (ref / date) | |
| Node 1: `rpm -q nginx-plus` / RHEL release (1.1) | |
| GO / NO-GO (1.1) | |
| New node hostname / IP | |
| New node RHEL release / free space (2.1) | |
| `/sbin/service` present, or how resolved (2.2) | |
| SELinux decision-table row (2.3) | |
| Copy method (3.2) / checksum OK on node (3.3) | |
| `rpm -K` all OK (4.2) | |
| Dry run: correct build, no upgrades (4.3) | |
| dnf history ID (4.4) | |
| nginx `disabled` / `inactive`, no licence (4.5) | |
| `nginx -t` output without licence (4.6) | |
| SOP-02 2.1–2.3, 4.1, 4.2 done (5) | |
| Licence received → SOP-02 from 3.5 (date) | |
