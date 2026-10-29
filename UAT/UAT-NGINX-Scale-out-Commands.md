# UAT Commands: NGINX Plus Scale-out (Nodes 3–6)

Companion to the NGINX Plus scale-out UAT workbook (13 test cases). The test IDs (TC_01…TC_13) match the workbook. Record each result in the workbook (columns F–K).

> Placeholders in `<ANGLE_BRACKETS>` must be replaced with the real values for the environment (see the table in section 0). Never write real tokens, passwords or licence files into this document.

**Current F5 state.** All six nodes (1–6) are enabled in the F5 pools and serve traffic active-active. Node 1 is also the nginx-sync primary.

**NTFY.** NGINX does not authenticate NTFY requests (the ntfy backend handles them), so this UAT has no NGINX JWT tests for NTFY: TC_03–TC_05 are not used.

**Where each test runs:**

| Marker | Where | Test cases |
|---|---|---|
| 💻 | **Any machine** on the client network with `bash` and `curl` (laptop, jump host, or a node). No SSH, no root | TC_06–TC_10, TC_12 (traffic), TC_13 (smoke tests) |
| 🖥️ | **On a node** over SSH, as `<ADMIN_USER>` then `sudo su` | TC_01, TC_02, TC_11, TC_12 (Filebeat), TC_13 (checksum) |

---

## 0. Before you start

**Hosts used below:**

| Name | Value |
|---|---|
| Node 1 (existing, sync primary) | `<NODE1_IP>` |
| Node 2 (existing) | `<NODE2_IP>` |
| Nodes 3–6 (new) | `<NODE3_IP>` … `<NODE6_IP>` |
| NIM | `<NIM_IP>` |
| ntfy host name (exact `server_name`) | `<NTFY_HOST>` |
| Object storage host name (exact `server_name`) | `<OBJSTG_HOST>` |
| F5 VIP | `<VIP_IP>` (from the F5 team) |
| Client CA certificate (for the VIP's TLS) | `<CA_FILE>` |
| Admin login (🖥️ only) | `<ADMIN_USER>`, then `sudo su` |

The host names must be the **exact** `server_name` values in the NGINX config. To read them, on Node 1: `nginx -T 2>/dev/null | grep server_name`. A wrong or unreplaced host name sends the request to the default server, which checks **no** JWT and answers `404 Not Found` for every test.

### 0.1 💻 Test machine setup

Needs `bash` and `curl`. On macOS, type `bash` first. The machine needs network access to the VIP on port 443, and to port 80 on the nodes for TC_09 and per-node tests.

**Tokens.** Ask the client for two test tokens:

| Variable | Token |
|---|---|
| `USER_TOKEN` | Valid user token (has a `tid`), not expired |
| `ADMIN_TOKEN` | Valid admin token (no `tid`), not expired |

Load them into your shell. What you paste is hidden and is not saved in the shell history:

```bash
read -rsp "USER_TOKEN: " USER_TOKEN; echo; export USER_TOKEN
```

```bash
read -rsp "ADMIN_TOKEN: " ADMIN_TOKEN; echo; export ADMIN_TOKEN
```

Check that both are loaded (two numbers, neither `0`):

```bash
echo ${#USER_TOKEN} ${#ADMIN_TOKEN}
```

**Before running the commands:**
- Replace `<OBJSTG_HOST>` (and `<NTFY_HOST>`, used in TC_13) with the exact server names.
- The VIP commands assume `<OBJSTG_HOST>` resolves to the VIP from your machine. Check with `nslookup <OBJSTG_HOST>`: it should return `<VIP_IP>`. If not, add `--resolve <OBJSTG_HOST>:443:<VIP_IP>` to each command.
- If a command prints `000`, curl did not trust the certificate. Add `--cacert <CA_FILE>` (the client's CA certificate). Use `-k` only to troubleshoot, never for a recorded result.

Each test command prints only the HTTP status code.

### 0.2 💻 Sanity check

```bash
curl -s "https://<OBJSTG_HOST>/"; echo
```

✅ Expect the client's JSON error body (Unauthorized or Forbidden). An HTML `404 Not Found` page means the host name does not match the `server_name`: fix it before running any test.

### 0.3 🖥️ Node preconditions (from SOP-02, not separate test cases)

Before UAT, every new node must have passed SOP-02 section 4. On each new node:

```bash
systemctl is-active nginx; curl -s http://127.0.0.1/health; echo; nginx -T 2>/dev/null | sha256sum
```

✅ `active`, `{"status":"UP",...}`, and the same checksum as Node 1.

---

## Installation and licensing

### TC_01 🖥️ Licence for the new subscription

On **each new node** and on **Node 1**:

```bash
python3 -c "import base64,json;p=open('/etc/nginx/license.jwt').read().split('.')[1];d=json.loads(base64.urlsafe_b64decode(p+'='*(-len(p)%4)));print({k:d.get(k) for k in ('sub','f5_order_type','f5_sat')})"
```

On **each new node**:

```bash
curl -s http://127.0.0.1/api/9/license | jq '{eval, active_till, reporting}'
```

```bash
grep -iE 'license|usage|mgmt' /var/log/nginx/error.log | tail -5
```

✅ Pass:
- `sub` on Nodes 3–6 is **different** from Node 1's (new subscription).
- `f5_order_type` is not `eval`, and `f5_sat` (end date, epoch seconds) matches the subscription.
- The API shows `eval: false` and `reporting.healthy: true`. No licence errors in `error.log`.

---

## Configuration synchronisation

### TC_02 🖥️ nginx-sync covers all 6 nodes

⚠️ Run nginx-sync with the sync account agreed with the client (root SSH is not allowed). See SOP-03 section 1. On **Node 1**:

```bash
grep -E '^(NODES|CONFPATHS|EXCLUDE)' /etc/nginx-sync.conf
```

```bash
nginx-sync.sh -C
```

Tell the client before the next part. It touches every node (a comment-only file). As root on Node 1:

```bash
echo "# UAT sync test $(date -u +%FT%TZ)" > /etc/nginx/conf.d/zz-uat-sync-test.conf
```

As the sync account on Node 1:

```bash
nginx-sync.sh -l /var/log/nginx-sync.log | tail -6
```

On **each new node**:

```bash
cat /etc/nginx/conf.d/zz-uat-sync-test.conf
```

As root on Node 1, remove it, then sync again as the sync account:

```bash
rm -f /etc/nginx/conf.d/zz-uat-sync-test.conf
```

```bash
nginx-sync.sh -l /var/log/nginx-sync.log | tail -6; nginx-sync.sh -C
```

On **each new node**: `ls /etc/nginx/conf.d/`, then the licence `sub` command from TC_01.

✅ Pass:
- `NODES` lists Nodes 2–6, and `EXCLUDE` contains `license.jwt`.
- The first `-C` is clean.
- The test file is on every node after the first sync, and **gone** after the second.
- No `restoring previous config` or `Failed` in the log, and the final `-C` is clean.
- The licence `sub` on the new nodes is unchanged.

---

## JWT tests: object storage (TC_06–TC_08) 💻

Run the commands as written: they go through the F5 VIP, the real client path, and the F5 spreads them across all six nodes. To test one node directly, see the end of this section.

TC_03–TC_05 (NTFY) are not used: NGINX does not authenticate NTFY requests.

### TC_06 Cohesity JWT connection with valid user token

```bash
curl -s -o /dev/null -w "%{http_code}\n" "https://<OBJSTG_HOST>/?token=$USER_TOKEN"
```

✅ Pass: `200`.

### TC_07 Cohesity JWT connection with valid admin token

```bash
curl -s -o /dev/null -w "%{http_code}\n" "https://<OBJSTG_HOST>/?token=$ADMIN_TOKEN"
```

✅ Pass: `200` (NGINX selected the admin key because the token has no `tid`).

### TC_08 Cohesity request without token / non-GET

GET without a token (expect `401`):

```bash
curl -s -o /dev/null -w "%{http_code}\n" "https://<OBJSTG_HOST>/"
```

PUT without a token (expect `200`):

```bash
curl -s -o /dev/null -w "%{http_code}\n" -X PUT -d test "https://<OBJSTG_HOST>/bucket/obj"
```

✅ Pass: `401`, then `200` (the same behaviour as Node 1 and Node 2).

### Testing one node directly (optional)

The F5 chooses which node answers. To test a **specific** node, bypassing the F5, take any command above and make two changes:
1. Change `https://` to `http://` (the nodes listen on plain HTTP, port 80).
2. Add `--resolve <HOST>:80:<NODE_IP>`, using the same host name as in the URL.

For example, TC_06 against Node 3:

```bash
curl -s -o /dev/null -w "%{http_code}\n" --resolve <OBJSTG_HOST>:80:<NODE3_IP> "http://<OBJSTG_HOST>/?token=$USER_TOKEN"
```

Repeat for Nodes 4–6 as needed.

**If a result is unexpected,** run the same test against Node 1 (`<NODE1_IP>`). The new nodes must behave exactly like Node 1. If Node 1 gives the same result, the cause is the test input (host name or token) or the client's existing config, not the new nodes. If only the VIP result differs from the per-node result, the difference is in the F5 (TLS profile, WebSocket profile, header handling, or an unhealthy member): raise it with the F5 team, quoting the TC number and the code returned.

**Optional 🖥️ log check** on a node:

```bash
tail -5 /var/log/nginx/objstg.access.log; grep -i jwt /var/log/nginx/objstg.error.log | tail -5
```

The requests appear in `objstg.access.log`, not in `access.log`. The only JWT errors are from the TC_08 negative test.

---

## F5 load balancing and availability

### TC_09 💻 New nodes reachable over the network

```bash
for ip in <NODE3_IP> <NODE4_IP> <NODE5_IP> <NODE6_IP>; do echo -n "$ip health: "; curl -s --max-time 5 http://$ip/health; echo; done
```

✅ Pass: `{"status":"UP",...}` from every new node.

This proves the test machine's path to the nodes. The path from the F5 to the nodes is proven by the green F5 health monitors in TC_10.

### TC_10 💻 F5 pool membership: traffic to all six nodes

**F5 state** (confirm with the F5 team, on every F5 device including the new one):
- Nodes 1–6 are **enabled** and **green** in every pool. The monitor is `GET /health`, expecting `200` and `"status":"UP"`.

**Traffic through the VIP.** Send 40 requests carrying the marker `uatlb1` (use `uatlb2`, `uatlb3` … if you repeat the test):

```bash
for i in $(seq 1 40); do curl -s -o /dev/null -w "%{http_code} " "https://<OBJSTG_HOST>/?token=$USER_TOKEN&uatlb1=$i"; done; echo
```

✅ All `200`.

**Which nodes served them.** Either:
- ask the F5 team for the pool member statistics (requests or connections per member) for the test window, or
- 🖥️ on **every node** (all 6), count the marker:

```bash
grep -c "uatlb1" /var/log/nginx/objstg.access.log
```

✅ Pass:
- All six nodes each served some of the 40 requests. The counts don't need to be equal.
- The first field or `X-Forwarded-For` in the access log shows the F5 or client address.

---

## Licence management (NIM) and logging

### TC_11 🖥️ NIM usage reporting from all 6 nodes

On **every node**:

```bash
curl -s http://127.0.0.1/api/9/license | jq '{reporting}'
```

```bash
nginx -T 2>/dev/null | grep -A3 -E '^\s*mgmt\s*\{'
```

On the **NIM server** (client access), check that every node IP posts usage:

```bash
grep 'POST /api/nginx-usage' /var/log/nginx/access.log | awk '{print $1, $9}' | sort | uniq -c
```

Usage report download and the check that both subscriptions are in it: **SOP-04 4.1 and 4.1b**. The F5 submission (SOP-04 4.2–4.4) only with F5 support.

✅ Pass:
- `reporting.healthy: true` and `fails: 0` on every node.
- The `mgmt` block points to `<NIM_IP>`.
- NIM shows `POST /api/nginx-usage` with status `200` from all 6 node IPs.
- The usage report contains both subscriptions (SOP-04 4.1b).

The client does not use the NGINX Agent, so the nodes do not appear on NIM's **Instances** page. That is expected: usage reporting (above) is what licensing needs.

### TC_12 NGINX log integration with ELK

Precondition: Filebeat is set up on every new node as in SOP-05.

🖥️ On **each new node**:

```bash
systemctl is-active filebeat; filebeat test output | tail -3
```

💻 Send one marked request to each new node:

```bash
for NODE in <NODE3_IP> <NODE4_IP> <NODE5_IP> <NODE6_IP>; do echo -n "$NODE: "; curl -s -o /dev/null -w "%{http_code}\n" --max-time 5 --resolve <OBJSTG_HOST>:80:$NODE "http://<OBJSTG_HOST>/uat-elk-$NODE"; done
```

✅ Pass:
- Filebeat is `active`, and the output test is OK.
- In Kibana, each `/uat-elk-<node IP>` request appears under the NGINX service tag, with the matching host name. The status code does not matter here.

---

## Regression of existing nodes

### TC_13 Existing Node 1 and Node 2 unaffected

Nodes 1–2 serve traffic together with the new nodes (active-active). Node 1 is also the nginx-sync primary, and both report to NIM.

💻 Smoke tests direct to each existing node (health, NTFY reachability, TC_06):

```bash
for NODE in <NODE1_IP> <NODE2_IP>; do echo "$NODE health: $(curl -s http://$NODE/health)"; echo "$NODE ntfy: $(curl -s -o /dev/null -w '%{http_code}' --resolve <NTFY_HOST>:80:$NODE "http://<NTFY_HOST>/")"; echo "$NODE objstg: $(curl -s -o /dev/null -w '%{http_code}' --resolve <OBJSTG_HOST>:80:$NODE "http://<OBJSTG_HOST>/?token=$USER_TOKEN")"; done
```

🖥️ On **Node 1 and Node 2**:

```bash
systemctl is-active nginx; curl -s http://127.0.0.1/api/9/license | jq '{eval, reporting}'
```

🖥️ Configuration identical to the new nodes. Run on **all 6 nodes**:

```bash
nginx -T 2>/dev/null | sha256sum
```

On Node 1, as the sync account: `nginx-sync.sh -C`

✅ Pass:
- Both nodes `active`, `{"status":"UP",...}`, and licence reporting healthy.
- NTFY answers (not `000` or `5xx`), and the object storage test returns `200`.
- **The same checksum on all 6 nodes.**
- `-C` clean.

---

## After UAT

- 💻 On the test machine: `unset USER_TOKEN ADMIN_TOKEN`, then close the shell.
- 🖥️ Delete `/root/tokens.env` on any node where one was created.
- Remove any temporary files: `/home/<ADMIN_USER>/node1-nginx-conf.tgz*` on the new nodes, and `/tmp/node1-nginx-conf.tgz*` on Node 1.
- Fill in the summary and sign-off block in the workbook.
