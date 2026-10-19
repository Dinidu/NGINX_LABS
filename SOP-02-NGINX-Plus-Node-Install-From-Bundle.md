# SOP-02: Install an NGINX Plus Node from the Offline Bundle

| | |
|---|---|
| Project | CLIENT NGINX Plus: additional nodes (2 → 6) |
| Phase | 2 of 3: install each new node from the bundle and join it to nginx-sync (Phase 1: SOP-01 builds the bundle. Phase 3: NIM and F5 pool membership) |
| Runs on | Each **new NGINX node** (no internet), plus a few commands on the **existing Node 1 / sync primary** |
| Input | `nginx-plus-<build>-el9-offline.tar.gz` + `.sha256` from SOP-01, and the node's `license.jwt` |
| Status | Validated in the Azure lab on 16 Oct 2026 (`lab-n2`: all steps passed). Repeat sections 1–6 for every new node |

---

## How to read this SOP

| Marker | Meaning |
|---|---|
| *(no marker)* | Do this step in **both** trial and production |
| 🧪 **TRIAL ONLY** | Practice runs in the Azure lab (`<LAB_RESOURCE_GROUP>`). **Skip in production** |
| 🏭 **PROD ONLY** | Production work on CLIENT nodes. **Skip in trial runs** |

Names used below:

| Role | 🧪 Trial | 🏭 Prod |
|---|---|---|
| Existing node / sync primary ("Node 1") | `lab-n1` (10.50.1.4) | CLIENT Node 1 (<NODE1_IP>) |
| New node being installed ("new node") | `lab-n2` (10.50.1.6) | CLIENT Nodes 3–6 (`<NEW_NODE_IP>`) |

Run every command as **root** (`sudo -i`). Paste commands **one line at a time**.

⚠️ This SOP switches between Node 1 and the new node. **Check the prompt (`[root@<hostname> ~]#`) before every step.** Running a Node 1 step on the new node creates the SSH key on the wrong server. Keep one terminal per server and label the terminal tabs.

---

## 1. Log in and check the new node

**1.1 Log in.**

🧪 **TRIAL ONLY.** Run from your Mac:

```bash
ssh -i ~/.ssh/id_lab azureuser@<LAB_N2_PUBLIC_IP>
```

🏭 **PROD ONLY.** SSH to the new node with the account CLIENT provided.

Then become root:

```bash
sudo -i
```

**1.2 Check the starting point:**

```bash
hostname
cat /etc/redhat-release
uname -m
rpm -q nginx-plus
ls /etc/yum.repos.d/ | grep -i nginx || echo "no nginx repo - good"
ping -c1 <NODE1_NAME_OR_IP>
```

✅ Expect:
- The correct hostname.
- The **same RHEL minor release as the bundle**. Compare with `BUILD-INFO.txt` from SOP-01 and with Node 1.
- `x86_64`.
- `package nginx-plus is not installed`.
- No nginx repository.
- Node 1 answers the ping.

If ping fails, do not assume the network is down. ICMP is often blocked by policy. Step 2.3 (SSH from Node 1) is the real test of the path nginx-sync needs.

**1.3 Check SELinux on BOTH Node 1 and the new node.** Run on each and record the results in the table in section 1.4:

```bash
getenforce
grep -E '^SELINUX=' /etc/selinux/config
getsebool httpd_can_network_connect
```

- `getenforce` shows the mode **now**.
- `/etc/selinux/config` shows the mode **after the next reboot**.
- If the two differ (for example `Permissive` now but `SELINUX=enforcing` in the config), someone switched it temporarily with `setenforce`. **The node will change behaviour at its next reboot.** Treat the **config value** as the real mode, and raise the difference with CLIENT.

🏭 **PROD ONLY.** On **Node 1**, also capture every local SELinux customisation, so the new nodes can be matched exactly:

```bash
semanage boolean -l -C
semanage port -l -C
semanage fcontext -l -C
semodule -lfull | grep -v '^100 '
```

`semanage` is in `policycoreutils-python-utils`. If it is not installed on Node 1, record "not available" and rely on `getsebool` above.

On **Node 1**, if it is `Permissive`, list what SELinux *would* have blocked for NGINX. This tells you what a new node in `Enforcing` mode will need:

```bash
ausearch -m avc -c nginx -ts this-month 2>/dev/null | audit2why | head -40
```

**1.4 Decide what to do with SELinux.** Find the row that matches the two `getenforce` results (use the config values if they differ from the runtime values) and follow it.

### General rules (apply to every row)

- ❌ **Never** run `setenforce 0` to "fix" a problem.
- ❌ **Never** edit `/etc/selinux/config`, or change the mode of any node, **without written approval from CLIENT**. Changing SELinux mode changes the server's security posture, and it is the client's decision.
- ❌ **Never** disable SELinux to make NGINX work. Find the denial and fix that (see "Troubleshooting a denial" below).
- ✅ Configure the new node so that it **works in the mode it is in**, and would **also work if it were switched to Enforcing** later. In practice: always set the boolean and always run `restorecon` when SELinux is not Disabled.
- ✅ Record both modes and any deviation in the UAT evidence (TC_001 comments).

### Decision table

| # | Node 1 (existing) | New node | What to do | What NOT to do |
|---|---|---|---|---|
| **A** | `Enforcing` | `Enforcing` | ✅ **Ideal. Proceed.** Copy Node 1's SELinux settings: `setsebool -P httpd_can_network_connect 1` (if Node 1 shows `on`), plus any custom boolean, port or fcontext from 1.3. Run `restorecon -Rv` after every file copy. After start-up, check `ausearch -m avc -ts recent` is empty. | Don't skip `restorecon`. Files copied or moved from a home directory keep the wrong label and NGINX cannot read them. |
| **B** | `Enforcing` | `Permissive` | ⚠️ **Raise with CLIENT** before go-live: the new node is weaker than the existing nodes. Proceed with the install, applying everything from row A (boolean, labels, `restorecon`). Before handover, check `ausearch -m avc -ts today` for **would-be denials** and fix them, so the node is ready for CLIENT to switch to Enforcing. | Don't switch it to Enforcing yourself without approval. Don't treat "it works" as proof, because Permissive hides denials. |
| **C** | `Enforcing` | `Disabled` | ⚠️ **Raise with CLIENT.** Re-enabling SELinux needs a config change, a full filesystem relabel and a reboot: a server-build task for CLIENT, ideally done **before** NGINX is installed. If CLIENT says proceed as-is, install normally and skip the SELinux commands (`setsebool` and `restorecon` will fail or do nothing). Record it as a deviation. | Don't re-enable SELinux yourself. A wrong relabel can stop the server from booting. |
| **D** | `Permissive` | `Enforcing` | ✅ Proceed, but expect gaps. Node 1 never needed working SELinux settings, so nobody has proven what NGINX needs here. Use Node 1's would-be denials (1.3) as the checklist. At minimum, `setsebool -P httpd_can_network_connect 1`. Test **every** UAT case on the new node directly (not only via F5) and check `ausearch -m avc -ts recent` after the tests. | Don't switch the new node to Permissive to "match" Node 1 without approval. Keeping Enforcing is more secure, and the settings above make it work. |
| **E** | `Permissive` | `Permissive` | ✅ Proceed. Apply the row-A settings anyway (boolean, `restorecon`) so the node is ready for Enforcing. Note for CLIENT: all nodes run Permissive. Recommend moving to Enforcing as a separate, planned change. | Don't change the mode during this scale-out. |
| **F** | `Permissive` | `Disabled` | ⚠️ Raise with CLIENT, as in row C. If told to proceed, install normally and skip the SELinux commands. | Don't re-enable SELinux yourself. |
| **G** | `Disabled` | `Enforcing` | ✅ Proceed, as in row D. There are no audit logs on Node 1 to learn from, so rely on the minimum setting (`httpd_can_network_connect 1`), `restorecon`, and full UAT testing on the new node with `ausearch` checks. | Don't disable SELinux on the new node to match Node 1 without approval. |
| **H** | `Disabled` | `Permissive` | ✅ Proceed, as in row E: apply the row-A settings so the node is ready for Enforcing. | Don't change the mode. |
| **I** | `Disabled` | `Disabled` | ✅ Proceed. Skip `setsebool` and `restorecon` (they will report SELinux is disabled). Note for CLIENT that SELinux is off on all nodes. | Don't enable SELinux during this work. |

🧪 **TRIAL ONLY.** The lab is row **A**: both nodes are `Enforcing`, and `lab-n1` has `httpd_can_network_connect --> on`.

### Troubleshooting a denial (any row except C, F, I)

```bash
ausearch -m avc -ts recent
ausearch -m avc -ts recent | audit2why
```

| `audit2why` says | Typical fix |
|---|---|
| "boolean `httpd_can_network_connect` was set incorrectly" | `setsebool -P httpd_can_network_connect 1` |
| "missing type enforcement" on a file under `/etc/nginx` | `restorecon -Rv /etc/nginx` (wrong label after copy or move) |
| `name_connect` to an unusual port | Prefer the boolean above (matches non-prod). The stricter option is `semanage port -a -t http_port_t -p tcp <PORT>`, but only with CLIENT's agreement |

If a denial does not appear but something still fails, hidden rules may be suppressing the log. Run `semodule -DB`, reproduce the problem, read `ausearch -m avc -ts recent`, then **always** restore with `semodule -B`.

---

## 2. Set up root SSH from Node 1, then copy the bundle

nginx-sync runs on Node 1 and pushes config to every peer as **root over SSH**. The same access is used to copy the bundle.

🏭 **PROD ONLY.** Root SSH from Node 1 to each new node needs **CLIENT's approval** (Day 1 prerequisite).

**2.1 On Node 1: create root's SSH key (if it does not exist) and print the public half:**

```bash
ls /root/.ssh/id_ed25519.pub 2>/dev/null || ssh-keygen -t ed25519 -N "" -C "nginx-sync@$(hostname)" -f /root/.ssh/id_ed25519
cat /root/.ssh/id_ed25519.pub
```

🏭 **PROD ONLY.** Node 1 probably already has a key that nginx-sync uses for Node 2. Check `ls /root/.ssh/` first and **reuse the existing key**. Do not create a second one.

**2.2 On the new node: authorise Node 1's key for root.**

Option 1: password login is allowed. Run on **Node 1**:

```bash
ssh-copy-id root@<NEW_NODE>
```

Option 2: password login is disabled (🧪 the lab is like this). Run on the **new node**:

```bash
mkdir -p /root/.ssh && chmod 700 /root/.ssh
cp -n /root/.ssh/authorized_keys /root/.ssh/authorized_keys.bak 2>/dev/null
echo "<PASTE THE ssh-ed25519 LINE FROM 2.1>" >> /root/.ssh/authorized_keys
chmod 600 /root/.ssh/authorized_keys
restorecon -Rv /root/.ssh
tail -1 /root/.ssh/authorized_keys
```

🧪 **TRIAL ONLY.** `authorized_keys` already contains an Azure line starting `no-port-forwarding,...command="echo 'Please login as...`. Leave it in place and add Node 1's key as a new line.

To check whether password login is allowed, run on the new node: `sshd -T | grep -iE '^(permitrootlogin|passwordauthentication)'`. Root key login needs `permitrootlogin` to be `yes` or `without-password` / `prohibit-password`.

**2.3 On Node 1: test root SSH to the new node.** Run the **first command on its own** and type `yes` at the host-key prompt. If you paste both lines together, the second line is taken as the answer and you get `Host key verification failed`.

```bash
ssh root@<NEW_NODE> hostname
```

```bash
ssh root@<NEW_NODE> 'rpm -q nginx-plus; whoami'
```

If steps 2.1–2.2 were run on the wrong server, undo them on that server: `\cp /root/.ssh/authorized_keys.bak /root/.ssh/authorized_keys` (the leading `\` bypasses RHEL's root alias `cp -i`, which otherwise asks "overwrite?" and silently skips the copy unless you answer `y`), then `rm -f /root/.ssh/id_ed25519 /root/.ssh/id_ed25519.pub`. Then redo 2.1 on Node 1.

✅ Expect: the new node's hostname, `package nginx-plus is not installed`, and `root`, **with no password prompt**.

**2.4 Copy the bundle to the new node.**

🧪 **TRIAL ONLY.** Run on `lab-n1`:

```bash
scp /root/nginx-plus-r36-9-el9-offline.tar.gz /root/nginx-plus-r36-9-el9-offline.tar.gz.sha256 root@lab-n2:/root/
```

🏭 **PROD ONLY.** Copy the archive and its `.sha256` from Node 1 with `scp` as above (if the bundle is on Node 1), or from the client USB, into `/root/` on the new node.

**2.5 On the new node: verify the bundle before unpacking it:**

```bash
cd /root
sha256sum -c nginx-plus-*-offline.tar.gz.sha256
ls -lh /root/nginx-plus-*-offline.tar.gz
```

✅ Expect: `...-offline.tar.gz: OK` (trial: 42 MB). **If the check fails, do not unpack.** Copy the file again.

---

## 3. Unpack, add the licence and install offline

Run all of this on the **new node** unless a step says otherwise.

**3.1 Unpack the bundle and check its contents:**

```bash
cd /root
tar xzf nginx-plus-*-offline.tar.gz
cat /root/nginx-bundle/VERSIONS.txt
cat /root/nginx-bundle/BUILD-INFO.txt 2>/dev/null
ls /root/nginx-bundle/rpms/*.rpm | wc -l
```

✅ Expect: `VERSIONS.txt` lists exactly one `nginx-plus-1:<build>` plus `nginx-sync`. The RPM count matches the SOP-01 record sheet. `BUILD-INFO.txt` shows the same RHEL minor release as this node (🧪 the first lab bundle predates this file, so it may be missing).

**3.2 Import F5's package signing key from the bundle:**

```bash
rpm --import /root/nginx-bundle/nginx_signing.key
rpm -q gpg-pubkey --qf '%{SUMMARY}\n' | grep -i nginx
```

**3.3 Dry run on this node.** Set the build from `VERSIONS.txt` first:

```bash
NGX_VER=1:36-9.el9.ngx
dnf --disablerepo='*' --repofrompath=offline,/root/nginx-bundle/rpms --enablerepo=offline --setopt=offline.gpgcheck=1 install --assumeno nginx-plus-$NGX_VER nginx-sync initscripts-service
```

✅ Expect: `Installing:` shows `nginx-plus <your build>` (**not** 37.x) and `nginx-sync`, plus possibly a few small dependencies, then `Operation aborted.` (🧪 lab-n2: 2 packages.)

🛑 **STOP if there is an `Upgrading:` section** listing core packages (glibc, openssl, systemd…). The bundle was built on a newer RHEL minor release than this node. Rebuild it per SOP-01 on a matching build server.

If dnf says `No match for argument: initscripts-service`, the bundle predates its addition to SOP-01. Check `ls /sbin/service`: if it is present, drop `initscripts-service` from the command. If it is absent, rebuild the bundle.

**3.4 Install from the bundle only:**

```bash
dnf --disablerepo='*' --repofrompath=offline,/root/nginx-bundle/rpms --enablerepo=offline --setopt=offline.gpgcheck=1 install -y nginx-plus-$NGX_VER nginx-sync initscripts-service
```

```bash
rpm -q nginx-plus nginx-sync
nginx -v
ls -l /sbin/service
```

✅ Expect: the same `nginx-plus` build and `nginx -v` output as Node 1 (🧪 `nginx/1.29.3 (nginx-plus-r36-p8)`), and `/sbin/service` present (nginx-sync needs it to reload this node).

**3.5 Put the licence in place. Do not start NGINX yet.**

🧪 **TRIAL ONLY.** There is one trial licence. Copy Node 1's (run on **Node 1**):

```bash
scp /etc/nginx/license.jwt root@<NEW_NODE>:/root/license.jwt
```

🏭 **PROD ONLY.** Use the **new licence for the additional nodes**, from the client USB, into `/root/license.jwt` on the new node. ❌ **Never copy Node 1's `license.jwt` to a new node.** Nodes 3–6 belong to a different subscription.

Then, on the **new node**:

```bash
install -m 0644 -o root -g root /root/license.jwt /etc/nginx/license.jwt
restorecon -v /etc/nginx/license.jwt
head -c 3 /etc/nginx/license.jwt; echo
python3 -c "import base64,json;p=open('/etc/nginx/license.jwt').read().split('.')[1];d=json.loads(base64.urlsafe_b64decode(p+'='*(-len(p)%4)));print({k:d.get(k) for k in ('sub','f5_order_type','f5_sat')})"
```

✅ Expect: `eyJ`, then the licence's `sub`, type and end date (`f5_sat`, epoch seconds).
- 🧪 Trial: the same `sub` as Node 1, type `eval`.
- 🏭 Prod: run the same `python3` command on **Node 1** and compare. **The `sub` must be different.** That proves the new node carries the new subscription.

---

## 4. Configure and start for the first time

**4.1 On the new node: SELinux and firewall.** Apply the decision-table row from 1.4. For row A (and B, D, E, G, H):

```bash
setsebool -P httpd_can_network_connect 1
getsebool httpd_can_network_connect
firewall-cmd --permanent --add-service=http
firewall-cmd --reload
firewall-cmd --list-services
```

✅ Expect: `httpd_can_network_connect --> on`, and `http` in the list. Rows C, F, I: skip `setsebool`. If firewalld is not running on Node 1 either, match Node 1 and record it.

**4.2 On the new node: name resolution and the default site:**

🧪 **TRIAL ONLY.** The lab has no DNS for Cohesity. Add a hosts entry first:

```bash
grep -q objstorage /etc/hosts || echo "10.50.1.5 objstorage.client.int" >> /etc/hosts
```

Then, in both trial and production:

```bash
getent hosts objstorage.client.int
rm -f /etc/nginx/conf.d/default.conf
ls /etc/nginx/conf.d/
```

✅ Expect: `getent` returns Cohesity's address, and `conf.d` is empty.
- 🏭 **PROD ONLY.** `getent` must return the same address from DNS as on Node 1. If it returns nothing, raise it with CLIENT before going further: `nginx -t` will fail once the configuration arrives.
- `default.conf` must be removed by hand. It clashes with `default_server` in the synced config, and nginx-sync never touches it (`EXCLUDE`).

**4.3 On Node 1: give the new node Node 1's `nginx.conf` before its first start.** This includes the `mgmt {}` usage-reporting block:

```bash
scp /etc/nginx/nginx.conf root@<NEW_NODE>:/etc/nginx/nginx.conf
```

🏭 This step matters in production. Started with the stock `nginx.conf`, a node tries to report usage to F5 over the internet, fails, and never sends its initial report to NIM.

**4.4 On the new node: fix labels, test and start:**

```bash
chown root:root /etc/nginx/nginx.conf
restorecon -Rv /etc/nginx
nginx -t
systemctl enable --now nginx || { sleep 2; systemctl start nginx; }
systemctl is-active nginx
```

✅ Expect: `nginx -t` successful, then `active`.

⚠️ **Known behaviour (seen on both lab nodes).** The **first** start after a fresh install fails with `open() "/run/nginx.pid" failed (13: Permission denied)` and `Failed with result 'protocol'`. A plain second start succeeds, and later starts and reboots are fine. The command above retries once automatically. **No SELinux change is needed.** If the **second** start also fails, troubleshoot (`journalctl -u nginx -n 20 --no-pager`, `ausearch -m avc -ts recent`, decision table 1.4). Root cause not yet confirmed: raise it with F5 support if it appears in production.

**4.5 On the new node: check the licence and usage reporting:**

```bash
sleep 10
grep -iE 'emerg|license|usage|mgmt' /var/log/nginx/error.log | tail -5
ls -l /var/lib/nginx/state/
```

✅ Expect:
- No licence errors. The only `emerg` line may be the first-start pid error from 4.4.
- `/var/lib/nginx/state/nginx-mgmt-state` exists, which means usage reporting has started.

Nothing listens on port 80 yet. That is expected: the site configuration arrives in step 5.

---

## 5. Join nginx-sync and receive the configuration

Run on **Node 1** (the sync primary) unless a step says otherwise.

How nginx-sync works (checked against `/usr/bin/nginx-sync.sh`):
- It tests Node 1's config, then for each peer: backs up the peer's config, `rsync --delete`s every path in `CONFPATHS`, runs `nginx -t` **on the peer**, and reloads it with `service nginx reload`. If the test or the reload fails, it restores the peer's previous config.
- `--delete` means files that exist only on a peer are removed from the synced paths.
- `EXCLUDE` patterns are never copied **and never deleted**.
- It runs as root using root's default SSH key (`~/.ssh/id_ed25519` or `id_rsa`).
- Local backups go to `/var/lib/nginx-sync/0…9` on Node 1. The lock file is `/tmp/nginx-sync.lock`. If a run was interrupted, delete the lock file before the next run.

**5.1 Check the peer has what nginx-sync needs:**

```bash
ssh root@<NEW_NODE> 'hostname; nginx -v; rsync --version | head -1; ls -l /sbin/service'
```

✅ Expect: hostname, the same NGINX build as Node 1, an rsync version, and `/sbin/service`. Without `/sbin/service`, nginx-sync **silently skips the reload** ("nginx is not running on …, skipping reload").

**5.2 Add the new node to `/etc/nginx-sync.conf`.**

🧪 **TRIAL ONLY.** Create the file. Paste the whole block, including the closing `EOF` line:

```bash
cat > /etc/nginx-sync.conf <<'EOF'
# nginx-sync: run on the sync primary only. Pushes config to NODES.
# Never sync license.jwt: each node keeps its own licence.
NODES="lab-n2"
CONFPATHS="/etc/nginx/nginx.conf /etc/nginx/conf.d /etc/nginx/snippets /etc/nginx/jwt"
EXCLUDE="default.conf license.jwt"
EOF
cat /etc/nginx-sync.conf
```

🏭 **PROD ONLY.** Node 1 **already has** this file (for Node 2). **Edit it. Do not overwrite it:**

```bash
cp -p /etc/nginx-sync.conf /etc/nginx-sync.conf.bak.$(date +%F)
cat /etc/nginx-sync.conf
vi /etc/nginx-sync.conf
```

1. Append the new node to `NODES`, separated by a space or a new line (for example `NODES="<NODE2> <NODE3> <NODE4> <NODE5> <NODE6>"`). Add nodes one at a time as each finishes section 4.
2. Make sure `EXCLUDE` contains `license.jwt` (for example `EXCLUDE="default.conf license.jwt"`).
3. **Check `CONFPATHS`.** If it is `/etc/nginx` (as in F5's example), every file under `/etc/nginx` is pushed with `--delete`. The `license.jwt` exclusion is then **essential**, or every new node would be overwritten with Node 1's licence. Do not change `CONFPATHS` itself without agreeing it with CLIENT.

**5.3 Compare before syncing.** This changes nothing:

```bash
nginx-sync.sh -C
```

✅ Expect: `Only in local:` lines for the files the new node does not have yet (🧪 `conf.d/jwt-config.conf`, `jwt`, `snippets`). `rsync: link_stat ... No such file or directory` messages for those folders are normal at this point. `nginx.conf` should **not** be listed (copied in 4.3).

🏭 **PROD ONLY.** The compare covers **every** node in `NODES`. Node 2 must show **no** differences. If it does, someone changed Node 2 directly. Stop and investigate before syncing, because the sync would overwrite their change.

**5.4 Sync, keeping a log:**

```bash
nginx-sync.sh -l /var/log/nginx-sync.log
```

✅ Expect, for each node:
- `Testing nginx config on <node>` followed by `test is successful`.
- `Redirecting to /bin/systemctl reload nginx.service`.
- **None** of these: `skipping reload`, `restoring previous config`, `Failed to`.

**5.5 Compare again:**

```bash
nginx-sync.sh -C
```

✅ Expect: only the `Comparing configs on master node and <node>:` header for each node. No differences and no rsync errors.

**5.6 On the new node: check what arrived:**

```bash
ls -RZ /etc/nginx/conf.d /etc/nginx/snippets /etc/nginx/jwt
python3 -c "import base64,json;p=open('/etc/nginx/license.jwt').read().split('.')[1];print(json.loads(base64.urlsafe_b64decode(p+'='*(-len(p)%4)))['sub'])"
systemctl is-active nginx
curl -s http://127.0.0.1/health; echo
```

✅ Expect:
- `jwt-config.conf`, `errors.conf`, `es512.jwk` and `es512-admin.jwk` present, all labelled `httpd_config_t`.
- The licence `sub` **unchanged** from 3.5.
- `active`.
- `{"status":"UP","message":"Nginx is running"}`.

If `/health` fails, the reload was skipped. Run `systemctl reload nginx` on the new node and check 5.1.

---

## 6. Test and verify

UAT references are to `UAT_DOCS/CLIENT_NGINX_Plus_UAT Document.xlsx`.

**6.1 Get test tokens.**

🧪 **TRIAL ONLY.** Tokens are valid for 1 hour. Run on your Mac:

```bash
cd ~/<PROJECT_DIR>/prod-build/lab
for k in user admin expired badsig revoked admin_with_userkey user_with_adminkey; do echo "$(echo $k | tr a-z A-Z)_TOKEN=$(python3 mint_token.py $k)"; done > tokens.env
scp -i ~/.ssh/id_lab tokens.env azureuser@<NEW_NODE_PUBLIC_IP>:~/
```

🏭 **PROD ONLY.** Use the test tokens supplied by the CLIENT application team (valid user, admin, expired, revoked tid). Put them in `/root/tokens.env` on the new node with the same variable names (`USER_TOKEN=…`). The `badsig` and key-mismatch tokens are lab-only. Delete the file after testing.

**6.2 Identical configuration (TC_008).** Run on **Node 1 and the new node**:

```bash
nginx -T 2>/dev/null | sha256sum
```

✅ Expect: **the same hash on both.** `nginx -T` prints the complete running config, including every included file.

**6.3 JWT tests directly on the new node (TC_016–TC_028).**

Set the tokens file and host names first:
- 🧪 Trial: `/home/azureuser/tokens.env`, hosts `ntfy.lab.local` / `objstg.lab.local`.
- 🏭 Prod: `/root/tokens.env`, hosts `<NTFY_HOST>` / `<OBJSTG_HOST>`.

```bash
source <TOKENS_FILE>
N="-H Host:<NTFY_HOST>"; O="-H Host:<OBJSTG_HOST>"; U=http://127.0.0.1
c(){ curl -s -o /dev/null -w "%{http_code}\n" "$@"; }
```

```bash
echo -n "TC_016 ntfy no token (401): ";        c $N $U/
echo -n "TC_017 ntfy token in query (200): ";  c $N "$U/?token=$USER_TOKEN"
echo -n "TC_018 ntfy token in header (200): "; c $N -H "Authorization: Bearer $USER_TOKEN" $U/
echo -n "TC_019 ntfy expired (401): ";         c $N "$U/?token=$EXPIRED_TOKEN"
echo -n "TC_020 ntfy bad signature (401): ";   c $N "$U/?token=$BADSIG_TOKEN"
echo -n "TC_021 ntfy revoked tid (403): ";     c $N "$U/?token=$REVOKED_TOKEN"
echo -n "TC_022 ntfy POST, no token (200): ";  c $N -X POST -d test $U/
echo -n "TC_024 objstg no token (401): ";      c $O $U/
echo -n "TC_025 objstg user token (200): ";    c $O "$U/?token=$USER_TOKEN"
echo -n "TC_026 objstg admin token (200): ";   c $O "$U/?token=$ADMIN_TOKEN"
echo -n "TC_027 objstg expired (401): ";       c $O "$U/?token=$EXPIRED_TOKEN"
echo -n "TC_028 objstg PUT, no token (200): "; c $O -X PUT -d test $U/bucket/obj
```

🧪 **TRIAL ONLY.** Key-selection proofs (both must be 401):

```bash
echo -n "objstg admin claims + user key (401): "; c $O "$U/?token=$ADMIN_WITH_USERKEY_TOKEN"
echo -n "objstg user claims + admin key (401): "; c $O "$U/?token=$USER_WITH_ADMINKEY_TOKEN"
```

What the backends received:

```bash
curl -s $N "$U/?token=$USER_TOKEN" | jq '{path, authorization}'
curl -si -N --max-time 3 $N -H "Connection: Upgrade" -H "Upgrade: websocket" -H "Sec-WebSocket-Version: 13" -H "Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==" "$U/?token=$USER_TOKEN" | head -1
```

✅ Expect:
- Every code matches the bracket.
- ntfy `path` is `/<userId><deviceId>/ws` (🧪 `/user42dev7/ws`) and `authorization` is the backend token.
- The WebSocket returns `HTTP/1.1 101 Switching Protocols`.
- 🏭 In production the real backends do not echo JSON, so the `jq` line shows nothing useful. Rely on the status codes, the `101`, and the backend teams' logs.

**6.4 Reach the new node over the network, as the F5 will (TC_010).** Run on **Node 1**, after sourcing the same tokens file:

```bash
curl -s http://<NEW_NODE_IP>/health; echo
curl -s -o /dev/null -w "%{http_code}\n" -H "Host: <OBJSTG_HOST>" -H "X-Forwarded-For: 203.0.113.50" "http://<NEW_NODE_IP>/?token=$USER_TOKEN"
```

✅ Expect: `/health` UP through the new node's firewall, and `200`. 🧪 In the lab, replace `-o /dev/null -w "%{http_code}\n"` with `| jq '{x_forwarded_for, authorization_stripped}'` to see `203.0.113.50` and `true` at the mock.

**6.5 Licence (TC_005, TC_035):**

```bash
curl -s http://127.0.0.1/api/9/license | jq '{eval, active_till, reporting}'
```

✅ Expect: `reporting.healthy: true` and `fails: 0`. The `uuid` differs from every other node, because each instance reports separately. 🏭 `eval` is `false`, and `active_till` matches the new subscription's end date.

**6.6 Change propagation, including deletion (TC_009).** Run on **Node 1**:

```bash
echo "# sync test $(date -u +%FT%TZ)" > /etc/nginx/conf.d/zz-sync-test.conf
nginx-sync.sh -l /var/log/nginx-sync.log | tail -4
ssh root@<NEW_NODE> 'cat /etc/nginx/conf.d/zz-sync-test.conf'
rm -f /etc/nginx/conf.d/zz-sync-test.conf
nginx-sync.sh -l /var/log/nginx-sync.log | tail -4
ssh root@<NEW_NODE> 'ls /etc/nginx/conf.d/'
```

✅ Expect: the test line appears on the new node after the first sync and is **gone** after the second, leaving only `jwt-config.conf`. 🏭 This also touches every other node in `NODES`. That is harmless (a comment-only file), but tell CLIENT before running it.

**6.7 Reboot test (TC_032):**

```bash
reboot
```

Log in again after about a minute:

```bash
systemctl is-active nginx; curl -s http://127.0.0.1/health; echo
```

✅ Expect: `active` and UP, with no pid error. The first-start issue in 4.4 does not recur after a reboot.

When section 6 passes, the node is ready for **Phase 3**: added to the F5 pools **disabled**, tested, then enabled.

---

## Record sheet

| Item | 🧪 Trial reference (lab-n2, 16 Oct 2026) | Value for this node |
|---|---|---|
| Date / performed by | 16 Oct 2026 / Dinidu | |
| New node hostname / IP | lab-n2 / 10.50.1.6 | |
| RHEL release: new node / Node 1 / bundle | 9.8 / 9.8 / 9.8 | |
| SELinux `getenforce`: Node 1 / new node | Enforcing / Enforcing | |
| SELinux config value: Node 1 / new node | enforcing / enforcing | |
| `httpd_can_network_connect`: Node 1 / new node | on / on | |
| Decision-table row (1.4) | A | |
| Deviations raised with CLIENT | none | |
| Root SSH from Node 1 works (2.3) | ✅ | |
| Bundle checksum OK on node (2.5) | ✅ | |
| Dry run: correct build, no upgrades (3.3) | ✅ 2 packages | |
| `nginx -v` matches Node 1 (3.4) | ✅ nginx-plus-r36-p8 | |
| Licence `sub`: new node / Node 1 (3.5) | FNI-xxxxxxxx… / same (trial) | |
| First start needed retry (4.4) | yes (known behaviour) | |
| nginx-sync compare clean after sync (5.5) | ✅ | |
| Config hash identical to Node 1 (6.2) | ✅ 201442f1…a9d1 | |
| JWT tests TC_016–028 (6.3) | ✅ 12/12 + 2 key-selection proofs | |
| Licence reporting healthy / uuid (6.5) | ✅ / xxxxxxxxxxxx… | |
| Change + delete propagation (6.6) | ✅ | |
| Reboot test (6.7) | ✅ active + UP after reboot, no pid error | |
