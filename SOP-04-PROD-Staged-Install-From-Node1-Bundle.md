# SOP-04: 🏭 PROD ONLY: Staged Install from the Existing Bundle on Node 1 (Licence Pending)

| | |
|---|---|
| Project | CLIENT NGINX Plus: additional nodes (2 → 6) |
| Use when | CLIENT's licence for the new nodes is **not ready yet**, and the new VMs must be prepared now |
| Runs on | The existing **Node 1** (<NODE1_IP>) and each **new node** (Nodes 3–6) |
| Input | The NGINX Plus RPMs (or offline bundle) left on Node 1 by its original install |
| Result | NGINX Plus and nginx-sync **installed but not started**, no licence on the node. Finish with SOP-02 when the licence arrives |
| Status | Section 1 and the key export in 3.2 checked on `lab-n1` (20 Oct 2026). The install commands match SOP-02 (validated on `lab-n2`, 16 Oct 2026). ⚠️ The end-to-end copy-and-install from Node 1's own folder was not run in the lab |

This SOP is **production only**. Run every command as **root** (`sudo -i`), **one line at a time**, and check the prompt (`[root@<hostname> ~]#`) before each step. Keep one terminal per server.

---

## 0. Rules

- ❌ **Never** copy Node 1's `license.jwt` (or any trial licence) to a new node.
- ❌ **Never** start NGINX on a new node in this SOP. R33+ will not run without a licence.
- ❌ Don't add a new node to nginx-sync `NODES` or to the F5 pools until its licence is in place.
- ✅ Tell CLIENT (email is enough) before installing, for example: *"NGINX Plus is being installed on the new nodes from the same RPMs used for Node 1, found on Node 1 at `<FOLDER>`; all package signatures verified with `rpm -K`. CLIENT's licence for the new nodes will be applied before go-live."*

---

## 1. On Node 1: find the existing RPMs

Steps 1.1–1.3 only read and change nothing.

**1.1 Record what Node 1 runs and how it was installed:**

```bash
hostname
rpm -q nginx-plus nginx-sync
nginx -v
cat /etc/redhat-release
dnf info --installed nginx-plus | grep -iE '^(version|release|from repo)'
dnf history info nginx-plus | grep -iE 'begin time|command line'
grep -rHiE '^\[|baseurl' /etc/yum.repos.d/
```

What it tells you:
- `From repo : @commandline`: installed from RPM files given on the command line. The `Command Line` from `dnf history` often shows the folder they were in.
- A repo name with a `baseurl=file:///...` line: installed from a local repo folder. Look there first.
- Any other name (for example `offline`) with `--repofrompath=<name>,<folder>` in the `Command Line`: installed from that folder. Look there first.
- `nginx-plus` (with a `pkgs.nginx.com` baseurl): installed online from F5. There may be no RPM files left on the server.

**1.2 Search the server for NGINX RPMs and bundles:**

```bash
find / \( -path /proc -o -path /sys -o -path /run -o -path /dev \) -prune -o \( -name 'nginx-plus*.rpm' -o -name 'nginx-sync*.rpm' -o -name '*nginx*.tar.gz' -o -name '*nginx*.tgz' \) -print 2>/dev/null | xargs -r ls -lh
```

- Ignore `.tgz` / `.tar.gz` files that are config backups (for example `nginx-etc-backup-*.tgz`, a few KB). A package bundle is tens of MB.
- 🔍 Found RPMs: note the **folder** that holds them, and continue with 1.3.
- 🔍 Found only a bundle archive: unpack a copy (`mkdir /tmp/node1-check && tar xzf <ARCHIVE> -C /tmp/node1-check`), find the folder with the RPMs inside (`find /tmp/node1-check -name 'nginx-plus*.rpm'`), and continue with 1.3.
- 🛑 Nothing found: **stop.** There is nothing on Node 1 to install from. Raise it with CLIENT and the team. A new bundle has to be built with SOP-01.

**1.3 Check the folder is complete and matches.** Set `D` to the folder from 1.2:

```bash
D=<FOLDER_WITH_THE_RPMS>
ls $D/*.rpm | wc -l
ls -d $D/repodata 2>/dev/null || echo "no repodata"
rpm -qp --qf '%{NAME}-%{EPOCH}:%{VERSION}-%{RELEASE}.%{ARCH}\n' $D/nginx-*.rpm
ls $D/initscripts-service-*.rpm 2>/dev/null || echo "no initscripts-service rpm"
rpm -K $D/*.rpm | grep -v 'digests signatures OK' || echo "ALL SIGNATURES OK"
```

✅ **Use it** only if all are true:
- An `nginx-plus-1:<build>` with the **same build** as `rpm -q nginx-plus` in 1.1. Other builds in the folder as well (for example R37) are acceptable, because steps 4.2–4.3 always name the exact build.
- An `nginx-sync` RPM is there.
- `ALL SIGNATURES OK`.

🛑 Otherwise (a different build only, no `nginx-sync`, failed signatures): **stop** and raise it, as in 1.2.

Write down `D`, the RPM count, `repodata` yes/no, the build, and whether `initscripts-service` is in the folder. Set the build for later steps (example for R36-p8):

```bash
NGX_VER=1:36-9.el9.ngx
```

---

## 2. On each new node: check it is ready

**2.1 Starting point:**

```bash
hostname
cat /etc/redhat-release
uname -m
rpm -q nginx-plus
ls /etc/yum.repos.d/ | grep -i nginx || echo "no nginx repo - good"
df -h /root /usr /var
```

✅ Expect: correct hostname, the **same RHEL minor release as Node 1** (1.1), `x86_64`, `package nginx-plus is not installed`, no nginx repo, and at least 500 MB free in `/root`, `/usr` and `/var`.

🛑 A **different** RHEL minor release from Node 1: raise it with CLIENT before going on. Node 1's RPMs were chosen for Node 1's release. The dry run in 4.2 will show whether it still installs cleanly.

**2.2 Check for `/sbin/service`** (nginx-sync needs it to reload this node):

```bash
ls -l /sbin/service
```

- ✅ Present: continue.
- ⚠️ Missing, but 1.3 found an `initscripts-service` RPM in Node 1's folder: it will be installed in step 4.
- ⚠️ Missing, and not in the folder: install it from CLIENT's own RHEL repository / Satellite (`dnf install initscripts-service`), if CLIENT allows. Without it, nginx-sync **silently skips the reload** on this node. Record the outcome.

**2.3 SELinux.** Run SOP-02 steps **1.3 and 1.4** on Node 1 and on this node, and record the decision-table row.

**2.4 Root SSH from Node 1.** Run SOP-02 steps **2.1–2.3** (needs CLIENT's approval). Step 3 copies the RPMs over this connection.

---

## 3. On Node 1: package the RPMs and copy them to the new node

**3.1 Package the folder.** `D` is from 1.3 (set it again if this is a new shell):

```bash
cd /root
tar czf node1-nginx-rpms.tar.gz -C $D .
sha256sum node1-nginx-rpms.tar.gz > node1-nginx-rpms.tar.gz.sha256
ls -lh node1-nginx-rpms.tar.gz; cat node1-nginx-rpms.tar.gz.sha256
```

**3.2 Export F5's signing keys** from Node 1's RPM database, so the new node can check the NGINX packages:

```bash
for k in $(rpm -q gpg-pubkey --qf '%{NAME}-%{VERSION}-%{RELEASE}\t%{SUMMARY}\n' | grep -i nginx | cut -f1); do rpm -q $k --qf '%{DESCRIPTION}\n'; done > /root/nginx_signing.key
grep -c 'BEGIN PGP PUBLIC KEY BLOCK' /root/nginx_signing.key
```

✅ Expect: a number of 1 or more (🧪 `lab-n1` had 3: `signing-key`, `signing-key-2`, `signing-key-3@nginx.com`). `0` means Node 1 never imported F5's key. Then use a `.key` file from the folder (`ls $D/*.key`) if there is one, or raise it.

**3.3 Copy to the new node:**

```bash
scp /root/node1-nginx-rpms.tar.gz /root/node1-nginx-rpms.tar.gz.sha256 /root/nginx_signing.key root@<NEW_NODE>:/root/
```

Repeat 3.3 for every new node. The archive only needs to be built once.

---

## 4. On the new node: verify and install (do not start)

**4.1 Verify, unpack and check signatures:**

```bash
cd /root
sha256sum -c node1-nginx-rpms.tar.gz.sha256
mkdir -p /root/node1-rpms && tar xzf node1-nginx-rpms.tar.gz -C /root/node1-rpms
ls /root/node1-rpms/*.rpm | wc -l
rpm --import /root/nginx_signing.key
rpm -K /root/node1-rpms/*.rpm | grep -v 'digests signatures OK' || echo "ALL SIGNATURES OK"
```

✅ Expect: `OK`, the same RPM count as 1.3, then `ALL SIGNATURES OK`. RHEL packages are checked against Red Hat's key (already on the server), NGINX packages against F5's key. 🛑 **If the checksum fails, do not unpack**: copy again. 🛑 Any line other than `ALL SIGNATURES OK`: **stop**, do not install.

**4.2 Dry run.** Set the build first (same as 1.3):

```bash
NGX_VER=1:36-9.el9.ngx
```

If the folder **has `repodata`** (1.3):

```bash
dnf --disablerepo='*' --repofrompath=offline,/root/node1-rpms --enablerepo=offline --setopt=offline.gpgcheck=1 install --assumeno nginx-plus-$NGX_VER nginx-sync
```

If it has **no `repodata`**:

```bash
dnf --disablerepo='*' install --assumeno /root/node1-rpms/nginx-plus-${NGX_VER#*:}.x86_64.rpm /root/node1-rpms/nginx-sync-*.rpm
```

If 2.2 found `/sbin/service` missing and the folder has `initscripts-service`, add it: `initscripts-service` at the end of the `repodata` command, or `/root/node1-rpms/initscripts-service-*.rpm` at the end of the no-`repodata` command. Use the same command in 4.3.

✅ Expect: `Installing:` shows `nginx-plus` with the build from 1.3 (**not** 37.x) and `nginx-sync`, perhaps with a few small dependencies, then `Operation aborted.`

- 🛑 `Upgrading:` section listing core packages (glibc, openssl, systemd…): **stop.** This node's RHEL release does not match the RPMs. Raise it (see 2.1).
- 🛑 `nothing provides ...` or `Problem: ... requires ...`: a dependency is missing from Node 1's folder. **Stop** and raise it.

**4.3 Install.** Run the same command as the dry run, with `-y` in place of `--assumeno`.

With `repodata`:

```bash
dnf --disablerepo='*' --repofrompath=offline,/root/node1-rpms --enablerepo=offline --setopt=offline.gpgcheck=1 install -y nginx-plus-$NGX_VER nginx-sync
```

Without `repodata`:

```bash
dnf --disablerepo='*' install -y /root/node1-rpms/nginx-plus-${NGX_VER#*:}.x86_64.rpm /root/node1-rpms/nginx-sync-*.rpm
```

Then check:

```bash
rpm -q nginx-plus nginx-sync
nginx -v
ls -l /sbin/service
dnf history list | head -3
```

✅ Expect: the same `rpm -q nginx-plus` and `nginx -v` output as Node 1 (1.1), an `nginx-sync` package, and `/sbin/service` present (or the outcome recorded in 2.2). Record the **dnf history ID** of the install (top line) for rollback.

**4.4 Make sure NGINX stays stopped:**

```bash
systemctl is-enabled nginx
systemctl is-active nginx
ls -l /etc/nginx/license.jwt 2>/dev/null || echo "no licence - correct for now"
```

✅ Expect: `disabled`, `inactive`, `no licence - correct for now`. Do **not** run `systemctl enable` or `start`. If NGINX were enabled, every reboot would try to start it and log licence errors.

**4.5 Optional: record what NGINX says without a licence:**

```bash
nginx -t
```

Record the output. A licence error is expected. It confirms the nginx-sync push (which runs `nginx -t` on each peer) must wait for the licence.

---

## 5. Prepare everything else that does not need the licence

All are in SOP-02, on the **new node**:

| SOP-02 step | What |
|---|---|
| 4.1 | SELinux boolean and firewall (`http`) |
| 4.2 | `getent hosts` for the object-storage name, and remove `default.conf` |

Then clean up the unpacked copy (keep the archive until go-live in case you need to reinstall):

```bash
rm -rf /root/node1-rpms
ls -lh /root/node1-nginx-rpms.tar.gz*
```

On **Node 1**, when every new node is installed, remove the temporary files from 1.2 and 3.1 (`rm -rf /tmp/node1-check`; keep or remove `/root/node1-nginx-rpms.tar.gz*` as CLIENT prefers).

---

## 6. When CLIENT's licence arrives

Continue with **SOP-02 from step 3.5** for each node:

1. **3.5** Place the new `license.jwt`, and check its `sub` is **different** from Node 1's.
2. **4.3–4.5** Copy Node 1's `nginx.conf`, start NGINX (first-start retry is expected), check licence and usage reporting.
3. **5** Add the node to nginx-sync `NODES` and sync.
4. **6** Run the tests, then Phase 3 (F5 pool, disabled first).

---

## Rollback

To remove the staged install from a new node, using the history ID from 4.3:

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
| CLIENT informed before install (ref / date) | |
| Node 1: `rpm -q nginx-plus` / `nginx -v` / RHEL release (1.1) | |
| Node 1 install source: `From repo` / baseurl / command line (1.1) | |
| Folder found on Node 1 (1.2) | |
| RPM count / `repodata` yes-no / `initscripts-service` in folder (1.3) | |
| Signatures OK on Node 1 (1.3) | |
| New node hostname / IP | |
| New node RHEL release / free space (2.1) | |
| `/sbin/service` present, or how resolved (2.2) | |
| SELinux decision-table row (2.3) | |
| Root SSH from Node 1 works (2.4) | |
| Archive SHA256 (3.1) / keys exported (3.2) | |
| Checksum and signatures OK on new node (4.1) | |
| Dry run: correct build, no upgrades, no missing deps (4.2) | |
| dnf history ID (4.3) | |
| nginx `disabled` / `inactive`, no licence (4.4) | |
| `nginx -t` output without licence (4.5) | |
| SOP-02 4.1, 4.2 done (5) | |
| Licence received → SOP-02 from 3.5 (date) | |
