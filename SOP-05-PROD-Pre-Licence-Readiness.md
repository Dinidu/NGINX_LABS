# SOP-05: 🏭 PROD ONLY: Pre-Licence Readiness for the New Nodes

| | |
|---|---|
| Project | CLIENT NGINX Plus: additional nodes (2 → 6) |
| Use when | The bundle is on the new nodes, and CLIENT's licence for them has **not** arrived yet |
| Runs on | The existing **Node 1** (<NODE1_IP>) and each **new node** (Nodes 3–6) |
| Before this | SOP-04 sections 1–4: NGINX Plus installed from Node 1's RPMs, **not started**, no licence |
| Result | Every new node ready except for the licence. On licence day only SOP-02 from step 3.5 is left (about 30 minutes per node) |
| Status | The commands in 1.2, 1.3 and 5.2 were checked on `lab-n1` / `lab-n2` (22 Oct 2026). Sections 3, 4 and 6 repeat SOP-02 steps validated on `lab-n2` (16 Oct 2026) |

This SOP is **production only**. Run every command as **root** (`sudo -i`), **one line at a time**, and check the prompt (`[root@<hostname> ~]#`) before each step. Finish one node completely, then repeat sections 2–5 for the others.

---

## 0. Rules

Allowed today: install checks, OS settings, SSH, network tests, read-only checks on Node 1 and NIM, paperwork.

❌ **Not until the licence arrives:**
- Starting NGINX on a new node, or enabling it (`systemctl enable`).
- Putting any `license.jwt` on a new node. That includes the trial licence and Node 1's.
- Adding a new node to `NODES` in `/etc/nginx-sync.conf`, or running `nginx-sync.sh` without `-C`.
- Adding a new node to an F5 pool.

---

## 1. On Node 1: capture the reference values (read-only)

The new nodes must match Node 1. These steps only read. Write the answers in the Node 1 record sheet at the end.

**1.1 Versions, OS, SELinux, firewall, time:**

```bash
hostname; cat /etc/redhat-release
rpm -q nginx-plus nginx-sync; nginx -v
getenforce; grep -E '^SELINUX=' /etc/selinux/config; getsebool httpd_can_network_connect
firewall-cmd --state; firewall-cmd --list-services; firewall-cmd --list-ports
chronyc tracking | grep -E 'Reference ID|System time|Leap status'
```

**1.2 Every backend NGINX connects to, and the NIM usage-report endpoint:**

```bash
nginx -T 2>/dev/null | grep -vE '^\s*#' | grep -oE '(proxy_pass|server|endpoint=)\s*[^; ]+' | sed -E 's#^(proxy_pass|server|endpoint=)\s*##; s#^https?://##; s#/.*##' | grep -E ':[0-9]+$|\.' | sort -u
```

```bash
nginx -T 2>/dev/null | grep -A4 -E '^\s*mgmt\s*\{'
```

✅ Expect: a list of `host:port` targets (🧪 `lab-n1` showed the ntfy, internal API and Cohesity backends), and the `mgmt` block with `usage_report endpoint=<NIM>`. A target with no port (for example a plain host name) is an upstream name or uses the default port: check it in `nginx -T`. **Copy the target list.** Section 5 tests each one from the new nodes.

**1.3 Local name entries and service overrides that the new nodes may need too:**

```bash
grep -vE '^\s*(#|$)|localhost' /etc/hosts
ls /etc/systemd/system/nginx.service.d/ 2>/dev/null || echo "no nginx drop-ins"
cat /etc/logrotate.d/nginx | sha256sum
```

- If a backend name from 1.2 is in Node 1's `/etc/hosts` (not DNS), the new nodes need the same line (step 4.4).
- If Node 1 has nginx drop-ins (for example `LimitNOFILE`), the new nodes need the same files (step 4.5).

**1.4 nginx-sync on Node 1: confirm the current state (read-only):**

```bash
cat /etc/nginx-sync.conf
ls /root/.ssh/
nginx-sync.sh -C
```

✅ Expect:
- `EXCLUDE` contains `license.jwt`. If it does **not**, plan to add it on licence day (SOP-02 5.2) **before** any new node joins. Note what `CONFPATHS` is.
- An existing root SSH key (reused in section 3).
- `nginx-sync.sh -C` shows **no differences** for Node 2. If it does, someone changed Node 2 directly. Raise it with CLIENT now, because the licence-day sync would overwrite their change.

**1.5 Back up the nginx-sync config now**, so licence day starts from a known copy:

```bash
cp -p /etc/nginx-sync.conf /etc/nginx-sync.conf.bak.$(date +%F)
ls -l /etc/nginx-sync.conf*
```

---

## 2. On each new node: confirm the install state

```bash
hostname; cat /etc/redhat-release
rpm -q nginx-plus nginx-sync; nginx -v
ls -l /sbin/service
systemctl is-enabled nginx; systemctl is-active nginx
ls -l /etc/nginx/license.jwt 2>/dev/null || echo "no licence - correct for now"
```

✅ Expect: the same RHEL release, `nginx-plus` build and `nginx -v` as Node 1 (1.1), `/sbin/service` present, `disabled`, `inactive`, `no licence - correct for now`. If NGINX is not installed yet, do SOP-04 sections 1–4 first.

---

## 3. Root SSH from Node 1 to the new node

nginx-sync pushes config as **root over SSH** from Node 1. This needs **CLIENT's approval**.

**3.1 On Node 1: print the existing public key.** Reuse the key nginx-sync already uses for Node 2. Do not create a second one:

```bash
ls /root/.ssh/
cat /root/.ssh/id_ed25519.pub 2>/dev/null || cat /root/.ssh/id_rsa.pub
```

**3.2 Authorise it on the new node.**

Option 1: root password login is allowed. On **Node 1**:

```bash
ssh-copy-id root@<NEW_NODE>
```

Option 2: password login is disabled. On the **new node**:

```bash
mkdir -p /root/.ssh && chmod 700 /root/.ssh
cp -n /root/.ssh/authorized_keys /root/.ssh/authorized_keys.bak 2>/dev/null
echo "<PASTE THE KEY LINE FROM 3.1>" >> /root/.ssh/authorized_keys
chmod 600 /root/.ssh/authorized_keys
restorecon -Rv /root/.ssh
tail -1 /root/.ssh/authorized_keys
```

To check which option applies, on the new node: `sshd -T | grep -iE '^(permitrootlogin|passwordauthentication)'`.

**3.3 On Node 1: test it.** Run the **first command on its own** and type `yes` at the host-key prompt:

```bash
getent hosts <NEW_NODE>
```

```bash
ssh root@<NEW_NODE> hostname
```

```bash
ssh root@<NEW_NODE> 'rpm -q nginx-plus; whoami; ls -l /sbin/service; rsync --version | head -1'
```

✅ Expect: Node 1 resolves the new node's name (nginx-sync uses names from `NODES`), then the hostname, the `nginx-plus` build, `root`, `/sbin/service` and an rsync version, **with no password prompt**. If `getent` returns nothing, use the IP in `NODES` on licence day, or ask CLIENT for a DNS entry.

If 3.2 was run on the wrong server, undo it there: `\cp /root/.ssh/authorized_keys.bak /root/.ssh/authorized_keys` (the `\` bypasses RHEL's `cp -i` alias).

---

## 4. On each new node: OS settings to match Node 1

**4.1 SELinux.** Follow the decision table in SOP-02 1.3–1.4 with the values from 1.1. If SELinux is not `Disabled`, and Node 1 shows `on`:

```bash
setsebool -P httpd_can_network_connect 1
getsebool httpd_can_network_connect
```

❌ Never change the SELinux **mode** without CLIENT's written approval.

**4.2 Firewall.** Match Node 1 (1.1). If firewalld is running:

```bash
firewall-cmd --permanent --add-service=http
firewall-cmd --reload
firewall-cmd --list-services
```

✅ Expect: `http` in the list. Add any other service or port Node 1 has (for example `https`) only if it is needed and CLIENT agrees.

**4.3 Remove the default site** (it clashes with `default_server` in the synced config, and nginx-sync never touches it):

```bash
rm -f /etc/nginx/conf.d/default.conf
ls /etc/nginx/conf.d/
```

**4.4 Local name entries (only if 1.3 showed backend names in Node 1's `/etc/hosts`).** With CLIENT's agreement, add the **same** line:

```bash
cp -p /etc/hosts /etc/hosts.bak.$(date +%F)
grep -q '<BACKEND_NAME>' /etc/hosts || echo "<LINE FROM NODE 1>" >> /etc/hosts
getent hosts <BACKEND_NAME>
```

**4.5 Service overrides (only if 1.3 showed nginx drop-ins on Node 1).** On **Node 1**:

```bash
ssh root@<NEW_NODE> 'mkdir -p /etc/systemd/system/nginx.service.d'
scp /etc/systemd/system/nginx.service.d/*.conf root@<NEW_NODE>:/etc/systemd/system/nginx.service.d/
ssh root@<NEW_NODE> 'restorecon -Rv /etc/systemd/system/nginx.service.d; systemctl daemon-reload; systemctl is-enabled nginx'
```

✅ Expect: `disabled` (copying a drop-in must not enable NGINX).

**4.6 Pre-stage Node 1's `nginx.conf` (optional, saves time on licence day).** This is SOP-02 step 4.3 done early. It is safe because NGINX is not running. On **Node 1**:

```bash
scp /etc/nginx/nginx.conf root@<NEW_NODE>:/etc/nginx/nginx.conf
ssh root@<NEW_NODE> 'chown root:root /etc/nginx/nginx.conf; restorecon -v /etc/nginx/nginx.conf; grep -A2 "mgmt" /etc/nginx/nginx.conf'
```

✅ Expect: the `mgmt` block pointing to CLIENT's NIM. Do **not** run `nginx -t` to judge this: without a licence it fails anyway.

---

## 5. On each new node: network, DNS and time checks

These catch most go-live problems without NGINX running.

**5.1 Time.** JWT expiry checks depend on correct time:

```bash
chronyc tracking | grep -E 'Reference ID|System time|Leap status'
timedatectl | grep -E 'Time zone|synchronized'
```

✅ Expect: `Leap status : Normal`, `System clock synchronized: yes`, the time offset within a fraction of a second, and the same time zone as Node 1.

**5.2 Every backend and NIM, from the new node.** Put the targets from 1.2 (backends **and** the NIM endpoint, with port 443 if the endpoint has none) in the list:

```bash
for t in <TARGET1:PORT> <TARGET2:PORT> <NIM_IP>:443; do h=${t%:*}; p=${t##*:}; if [[ ! $h =~ ^[0-9.]+$ ]] && ! getent hosts $h >/dev/null; then echo "$t  NO-DNS"; elif timeout 5 bash -c "</dev/tcp/$h/$p" 2>/dev/null; then echo "$t  OPEN"; else echo "$t  BLOCKED/CLOSED"; fi; done
```

✅ Expect: `OPEN` for every target (🧪 lab: ntfy, internal APIs, Cohesity all `OPEN`).
- `NO-DNS`: the name does not resolve on this node. Compare with Node 1 (`getent hosts <name>` there). Fix with DNS (CLIENT) or step 4.4.
- `BLOCKED/CLOSED`: a firewall between this node and the backend. Run the same loop on Node 1: if it is `OPEN` there, ask CLIENT's network team to open the same rule for the new node's IP.

**5.3 Names resolve to the same addresses as on Node 1.** For every name in the target list:

```bash
getent hosts <BACKEND_NAME>
```

✅ Expect: the same address as Node 1 shows for that name.

**5.4 F5 to the new node.** NGINX is not running, so this cannot be tested end to end today. Ask CLIENT's network team to confirm that **port 80 from the F5 self-IPs** to each new node is allowed, the same as for Node 1 and Node 2. The node firewall was opened in 4.2.

---

## 6. With CLIENT's teams (today)

| Who | What to get or confirm |
|---|---|
| Network / F5 | Port 80 from F5 self-IPs to Nodes 3–6. Pool names. The change window for adding the new nodes as **disabled** pool members |
| Application team | UAT test tokens: valid user, admin, expired, and a revoked tid. Delivered securely, not in chat |
| Licensing | The licence for Nodes 3–6. Confirm it is a **separate subscription**, and whether its JWT must be uploaded to NIM (SOP-03 1A.4). Expected delivery date |
| NIM owner (read-only) | NIM patch level, 2.21.0 or 2.21.1 (SOP-03 1A.1). Licence mode and health (SOP-03 1A.2). Whether Node 1 runs the NGINX Agent (SOP-03 1A.3) |
| CLIENT approvals | Root SSH from Node 1 (section 3). Any `/etc/hosts` change (4.4). The install from Node 1's RPMs (SOP-04) |

---

## 7. Paperwork

- Fill in the record sheets below (Node 1 once, then one per new node).
- UAT sheet: fill in the test cases that can already pass (OS and version checks, install, SELinux, firewall, network reachability). Leave the rest `Pending`.

---

## 8. Licence day: what is left

For each new node, in order:
1. **SOP-02 3.5:** place the new `license.jwt`, and check its `sub` is **different** from Node 1's.
2. **SOP-02 4.3–4.5:** copy `nginx.conf` (skip if pre-staged in 4.6), start NGINX (the first-start retry is expected), check licence and usage reporting.
3. **SOP-02 5:** add the node to `NODES` (back-up made in 1.5), compare, sync.
4. **SOP-02 6:** tests, then add to the F5 pools **disabled**, test, enable.

---

## Record sheet: Node 1 (once)

| Item | Value |
|---|---|
| Date / performed by | |
| RHEL release / `nginx-plus` / `nginx -v` (1.1) | |
| SELinux runtime / config / boolean (1.1) | |
| Firewall state / services / ports (1.1) | |
| Time source and offset (1.1) | |
| Backend targets (1.2) | |
| NIM `usage_report endpoint` (1.2) | |
| Backend names in `/etc/hosts` (1.3) | |
| nginx drop-ins / logrotate hash (1.3) | |
| nginx-sync `NODES` / `CONFPATHS` / `EXCLUDE` has `license.jwt` (1.4) | |
| `nginx-sync.sh -C`: Node 2 clean (1.4) | |
| nginx-sync config backup file (1.5) | |

## Record sheet: each new node

| Item | Node 3 | Node 4 | Node 5 | Node 6 |
|---|---|---|---|---|
| Hostname / IP | | | | |
| Install state OK, disabled, inactive, no licence (2) | | | | |
| Root SSH from Node 1, name resolves (3.3) | | | | |
| SELinux row / boolean (4.1) | | | | |
| Firewall `http` (4.2) | | | | |
| `default.conf` removed (4.3) | | | | |
| `/etc/hosts` / drop-ins matched, or n/a (4.4–4.5) | | | | |
| `nginx.conf` pre-staged (4.6), yes/no | | | | |
| Time synchronised (5.1) | | | | |
| All targets `OPEN` (5.2) | | | | |
| Names match Node 1 (5.3) | | | | |
| F5 port 80 confirmed by CLIENT (5.4) | | | | |
