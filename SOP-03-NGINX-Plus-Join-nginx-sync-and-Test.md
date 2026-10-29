# SOP-03: Join the New Node to nginx-sync, Test, and Add it to the F5 Pools

| | |
|---|---|
| Project | CLIENT NGINX Plus: additional nodes (2 → 6) |
| Phase | 3 of 4: put the new node under nginx-sync, run the UAT tests, then add it to the F5 pools. (1: SOP-01 bundle. 2: SOP-02 install and start. 4: SOP-04 NIM) |
| Runs on | **Node 1** (<NODE1_IP>, the sync primary) and the **new node** |
| Before this | SOP-02 finished on the new node: NGINX running with Node 1's configuration (identical `nginx -T` hash) and its own licence |
| Status | Steps validated in the Azure lab (16 Oct 2026) with nginx-sync running as root. ⚠️ **The nginx-sync account is still to be confirmed.** CLIENT does not allow root SSH between nodes, so nginx-sync will run as `<ADMIN_USER>` (its built-in sudo mode). A separate nginx-sync SOP will cover that setup. Until then, the sync commands below show **what** to run, not the final account |

---

## How you work on CLIENT nodes

SSH in as **`<ADMIN_USER>`**, then `sudo su` for root. Check the host name in the prompt before every step, and paste **one line at a time**.

**Rules:**
- ❌ Never copy Node 1's `license.jwt` to a new node, and never let nginx-sync copy it (`EXCLUDE`, step 1.2).
- ❌ Never sync while `nginx-sync.sh -C` shows a difference nobody can explain. A sync makes every node an exact copy of Node 1 (`rsync --delete`).

---

## 1. Join nginx-sync

How nginx-sync works (checked against `/usr/bin/nginx-sync.sh`, nginx-sync 1.1):
- It tests Node 1's config, then for each node in `NODES`: backs up the node's config, `rsync --delete`s every path in `CONFPATHS`, runs `nginx -t` **on that node**, and reloads it with `service nginx reload`. If the test or the reload fails, it restores that node's previous config.
- `--delete` means files that exist only on a node are removed from the synced paths. `EXCLUDE` patterns are never copied **and never deleted**.
- The node keeps one copy of its previous config in `/var/lib/nginx-sync/`, replaced at the next sync. Node 1 keeps its own backups in `/var/lib/nginx-sync/0…9`.
- The lock file is `/tmp/nginx-sync.lock`. If a run was interrupted, delete the lock file before the next run.

**1.1 Prerequisite: the sync account can reach the new node.** ⚠️ To be set up with the separate nginx-sync SOP (SSH key from Node 1 to the new node for the sync account, and passwordless sudo). Check from Node 1, as the sync account:

```bash
ssh <SYNC_USER>@<NEW_NODE> 'hostname; nginx -v; rsync --version | head -1; ls -l /sbin/service'
```

✅ Expect: hostname, the same NGINX build as Node 1, an rsync version, and `/sbin/service`, **with no password prompt**. Without `/sbin/service`, nginx-sync **silently skips the reload** ("nginx is not running on …, skipping reload").

**1.2 On Node 1, as root: add the new node to `/etc/nginx-sync.conf`.** Node 1 **already has** this file (for Node 2). **Edit it. Do not overwrite it:**

```bash
cp -p /etc/nginx-sync.conf /etc/nginx-sync.conf.bak.$(date +%F-%H%M)
cat /etc/nginx-sync.conf
vi /etc/nginx-sync.conf
```

1. Append the new node to `NODES`, separated by a space (for example `NODES="<NODE2> <NODE3>"`). Add nodes one at a time.
2. Make sure `EXCLUDE` contains `license.jwt` (for example `EXCLUDE="default.conf license.jwt"`).
3. **Check `CONFPATHS`.** If it is `/etc/nginx`, every file under `/etc/nginx` is pushed with `--delete`, and the `license.jwt` exclusion is **essential**: otherwise every new node would get Node 1's licence. Don't change `CONFPATHS` without CLIENT.

**1.3 Compare before syncing.** This changes nothing:

```bash
nginx-sync.sh -C
```

✅ Expect: the `Comparing configs on master node and <node>:` header for each node, with **no differences**. The new node already has Node 1's configuration (SOP-02 4.5), so it should be clean too.

🛑 **Any difference: understand it before syncing.** See 1.4.

**1.4 If `-C` shows a difference.** `local:` = Node 1, `remote:` = the other node:

| Line | Meaning | What a sync will do to that node |
|---|---|---|
| `Files local:X and remote:X differ` | The file is different on the two nodes | **Overwrite** it with Node 1's |
| `Only in local:<dir>: <file>` | The file exists only on Node 1 | **Create** it |
| `Only in remote:<dir>: <file>` | The file exists only on that node | **Delete** it (unless it matches `EXCLUDE`) |

See the exact lines for one node (`-d` = full diff, `-c` = one node):

```bash
nginx-sync.sh -d -c <NODE>
```

Lines starting with `-` exist only on Node 1 (the sync **adds** them). Lines starting with `+` exist only on the other node (the sync **removes** them).

Before deciding, save that node's current config. From Node 1:

```bash
ssh <SYNC_USER>@<NODE> 'sudo tar czf - /etc/nginx 2>/dev/null' > ~/node-<NODE>-etc-nginx-$(date +%F-%H%M).tgz
```

Then decide with CLIENT, and record the decision:
- A **wanted** change made directly on that node: copy it into Node 1 first (as root, then `nginx -t`), until `-C` no longer shows it.
- An **unwanted** or stale change: get CLIENT's written OK to discard it, keep the `.tgz`, then sync.
- An unexpected change that exists only on Node 1: someone edited Node 1 without syncing, and the change is not live on the other nodes. Ask CLIENT whether it should go live **now**.

**1.5 Sync, keeping a log:**

```bash
nginx-sync.sh -l /var/log/nginx-sync.log
```

✅ Expect, for each node:
- `Testing nginx config on <node>` followed by `test is successful`.
- `Redirecting to /bin/systemctl reload nginx.service`.
- **None** of these: `skipping reload`, `restoring previous config`, `Failed to`, `Permission denied`.

**1.6 Compare again:**

```bash
nginx-sync.sh -C
```

✅ Expect: only the `Comparing configs on master node and <node>:` header for each node. No differences and no rsync errors.

**1.7 On the new node: check the licence was not touched, and NGINX is healthy:**

```bash
python3 -c "import base64,json;p=open('/etc/nginx/license.jwt').read().split('.')[1];print(json.loads(base64.urlsafe_b64decode(p+'='*(-len(p)%4)))['sub'])"
systemctl is-active nginx
curl -s http://127.0.0.1/health; echo
```

✅ Expect: the licence `sub` **unchanged** from SOP-02 3.5, `active`, and `{"status":"UP","message":"Nginx is running"}`.

---

## 2. Test and verify

UAT references are to `UAT_DOCS/CLIENT_NGINX_Plus_UAT Document.xlsx`.

**2.1 Test tokens.** Use the test tokens supplied by the CLIENT application team (valid user, admin, expired, revoked tid). Put them in `/root/tokens.env` on the new node with these variable names: `USER_TOKEN=…`, `ADMIN_TOKEN=…`, `EXPIRED_TOKEN=…`, `REVOKED_TOKEN=…`. Delete the file after testing.

**2.2 Identical configuration (TC_008).** Run on **Node 1 and the new node**:

```bash
nginx -T 2>/dev/null | sha256sum
```

✅ Expect: **the same hash on both.**

**2.3 JWT tests directly on the new node (TC_016–TC_028):**

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

WebSocket upgrade through the new node:

```bash
curl -si -N --max-time 3 $N -H "Connection: Upgrade" -H "Upgrade: websocket" -H "Sec-WebSocket-Version: 13" -H "Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==" "$U/?token=$USER_TOKEN" | head -1
```

✅ Expect: every code matches its bracket, and `HTTP/1.1 101 Switching Protocols`. The real backends do not echo what they received, so confirm the backend behaviour (path, injected backend token, `Authorization` removed) with the backend teams' logs. TC_020 (bad signature) needs a specially crafted token. Run it only if the application team supplies one.

**2.4 Reach the new node over the network, as the F5 will (TC_010).** Run on **Node 1**, after copying the same tokens file there:

```bash
curl -s http://<NEW_NODE_IP>/health; echo
curl -s -o /dev/null -w "%{http_code}\n" -H "Host: <OBJSTG_HOST>" -H "X-Forwarded-For: 203.0.113.50" "http://<NEW_NODE_IP>/?token=$USER_TOKEN"
```

✅ Expect: `/health` UP through the new node's firewall, and `200`.

**2.5 Licence (TC_005, TC_035):**

```bash
curl -s http://127.0.0.1/api/9/license | jq '{eval, active_till, reporting}'
```

✅ Expect: `eval: false`, `active_till` = the new subscription's end date, `reporting.healthy: true` and `fails: 0`. The `uuid` differs from every other node, because each instance reports separately.

**2.6 Change propagation, including deletion (TC_009).** Tell CLIENT first: this touches every node in `NODES` (a harmless comment-only file). On **Node 1**, as root:

```bash
echo "# sync test $(date -u +%FT%TZ)" > /etc/nginx/conf.d/zz-sync-test.conf
```

Sync (as the sync account), and check the new node:

```bash
nginx-sync.sh -l /var/log/nginx-sync.log | tail -4
ssh <SYNC_USER>@<NEW_NODE> 'cat /etc/nginx/conf.d/zz-sync-test.conf'
```

Remove it on Node 1 (as root), sync again, and check:

```bash
rm -f /etc/nginx/conf.d/zz-sync-test.conf
```

```bash
nginx-sync.sh -l /var/log/nginx-sync.log | tail -4
ssh <SYNC_USER>@<NEW_NODE> 'ls /etc/nginx/conf.d/'
```

✅ Expect: the test line appears on the new node after the first sync, and is **gone** after the second.

**2.7 Reboot test (TC_032).** On the new node:

```bash
reboot
```

Log in again after about a minute (`<ADMIN_USER>`, then `sudo su`):

```bash
systemctl is-active nginx; curl -s http://127.0.0.1/health; echo
```

✅ Expect: `active` and UP, with no pid error. The first-start issue in SOP-02 4.7 does not recur after a reboot.

---

## 3. Add the new node to the F5 pools

CLIENT's network / F5 team makes this change. Give them the node's IP and port 80, and ask for:
1. The node added to each pool **disabled** (or with ratio 0), with the same health monitor as Node 1 and Node 2 (`GET /health`, expecting `200` and `"status":"UP"`).
2. The monitor showing the member **up**.
3. A test through the VIP while only that member is enabled, if CLIENT allows. Otherwise enable it alongside the others and watch `tail -f /var/log/nginx/access.log` on the new node for real traffic with `200`s.
4. Then enabled for normal traffic.

Repeat SOP-02 and this SOP for each new node, then check usage reporting for all nodes with **SOP-04**.

---

## Record sheet (one per node)

| Item | Value |
|---|---|
| Date / performed by | |
| New node hostname / IP | |
| Sync account reaches node without password (1.1) | |
| `nginx-sync.conf` backup file name (1.2) | |
| `EXCLUDE` contains `license.jwt` / `CONFPATHS` (1.2) | |
| `-C` before sync clean, or differences + decision + approved by (1.3–1.4) | |
| Sync OK for every node (1.5) / `-C` clean after (1.6) | |
| Licence `sub` unchanged (1.7) | |
| Config hash identical to Node 1 (2.2) | |
| JWT tests TC_016–028 / WebSocket 101 (2.3) | |
| Network test (2.4) | |
| Licence: not eval, reporting healthy, uuid (2.5) | |
| Change + delete propagation (2.6) | |
| Reboot test (2.7) | |
| F5: added disabled, monitor up, enabled, date (3) | |
