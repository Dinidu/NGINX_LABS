# SOP-03: NGINX Instance Manager (NIM): Usage Reporting for the New Nodes

| | |
|---|---|
| Project | CLIENT NGINX Plus: additional nodes (2 → 6) |
| Phase | 3 of 3: NIM and F5 pool membership (Phase 1: SOP-01 bundle. Phase 2: SOP-02 node install + nginx-sync) |
| 🏭 Production scope | CLIENT's **existing** NIM (`<NIM_IP>`, **version 2.21**, disconnected). **We do not install or reconfigure NIM.** We check it, confirm the new nodes report to it, and support CLIENT's usage-report cycle |
| 🧪 Trial scope | Build a NIM **2.21.1** in the lab (`lab-nim`) that matches CLIENT's, then practise everything above against it |
| Validated | Azure lab, 16 Oct 2026: NIM 2.21.1 (lightweight, disconnected) on RHEL 9.8. `lab-n1` + `lab-n2` reporting (`POST /api/nginx-usage` → 200). Initial report downloaded |

---

## How to read this SOP

| Marker | Meaning |
|---|---|
| *(no marker)* | Do this step in **both** trial and production |
| 🧪 **TRIAL ONLY** | Practice runs in the Azure lab. **Skip in production** |
| 🏭 **PROD ONLY** | Production work at CLIENT. **Skip in trial runs** |
| ⚠️ **NOT YET VALIDATED** | Taken from F5's documentation but not yet run in the lab. Practise it before relying on it onsite |
| 🤝 **WITH F5 SUPPORT ONSITE** | From F5's documentation. Deliberately **not** practised in the lab (it submits to F5's licensing service). Run it onsite **only together with F5 support** and CLIENT, using CLIENT's JWT |

Run every command as **root** (`sudo -i`). Paste commands **one line at a time**, and check the prompt's hostname before each step.

**Passwords:** never write a real password into this SOP, a ticket or a chat. In commands, `NIMPW` is read from the keyboard so it does not appear on screen or in shell history:

```bash
read -rsp 'NIM admin password: ' NIMPW; echo
```

---

## 0. Background: read once

**Two different ways NGINX Plus talks to NIM:**

| | Usage reporting | NGINX Agent |
|---|---|---|
| What | NGINX Plus sends licence usage, set by `mgmt { usage_report endpoint=<NIM>; }` in `nginx.conf` | Separate `nginx-agent` service on each node, registered with NIM |
| NIM shows | Licensing / usage data. The nodes do **not** appear in the Instances list | The **Instances** inventory (version, status, config), metrics |
| Needed for | **Licensing compliance** (R33+ must report) | Inventory, monitoring. Optional |
| This SOP | Sections 3–5 | Section 6 (only if CLIENT's existing nodes run the agent) |

**NIM 2.21 facts that matter here** (F5 release notes):
- **Multiple subscriptions are supported from 2.20:** *"a single NGINX Instance Manager can receive usage from these instances and send a bulk usage report to F5."* Nodes 1–2 (existing subscription) and Nodes 3–6 (new subscription) can report to the same NIM.
- **Disconnected mode on 2.20/2.21 needs a manual cycle:** download the report from NIM, submit it to F5 from an internet-connected machine, download F5's acknowledgement, upload it back into NIM (section 5). From 2.22 the acknowledgement upload is no longer needed.
- **2.21.1 fixed** *"Usage reporting shows higher values than actual consumption (46867)."* On 2.21.0, reports can over-count, and going from 2 to 6 nodes makes this more visible.
- **Lightweight mode (no ClickHouse) exists from 2.20.** Instance management and usage reporting work, metrics do not.

---

## 1. Check CLIENT's NIM (🏭 PROD ONLY) / build the lab NIM (🧪 TRIAL ONLY)

### 1A. 🏭 PROD ONLY: check the existing NIM (read-only)

**1A.1 Exact version.** On the NIM server, with CLIENT's access:

```bash
rpm -q nms-instance-manager
```

Or in the NIM web UI, look at the version in the help/about area.

- `2.21.1-…`: nothing to raise.
- `2.21.0-…`: tell CLIENT about the over-counting fix in 2.21.1 (issue 46867). Upgrading is CLIENT's decision and is out of our scope.

**1A.2 Licence and health.** Run from Node 1, or any machine that can reach NIM, with CLIENT's NIM admin credentials:

```bash
read -rsp 'NIM admin password: ' NIMPW; echo
curl -sk -u admin:"$NIMPW" https://<NIM_IP>/api/platform/v1/health | jq .
curl -sk -u admin:"$NIMPW" https://<NIM_IP>/api/platform/v1/license | jq '{mode: .currentStatus.modeOfOperation, status: [.. | .status? // empty | strings] | unique}'
```

✅ Expect: `"status": "healthy"`, `mode` `DISCONNECTED`, and a licence status such as `CONFIG_REPORT_READY` or `INITIALIZE_ACTIVATION_COMPLETE`. `{"code":1409,"message":"No valid license..."}` means NIM's licence is missing or lapsed. Raise it with CLIENT.

**1A.3 Agent in use?** On Node 1:

```bash
rpm -q nginx-agent; systemctl is-active nginx-agent
```

Also look at **Instances** in CLIENT's NIM. If Nodes 1–2 are listed there, the agent is in use, and section 6 applies to the new nodes.

**1A.4 New subscription's JWT.** Do **not** upload the new subscription's `license.jwt` to NIM unless CLIENT or F5 confirm it is required. This is part of the licensing confirmation CLIENT is getting from F5 / the reseller.

Then go to **section 3**.

### 1B. 🧪 TRIAL ONLY: build the lab NIM 2.21.1

Validated on `lab-nim` (RHEL 9.8, 7.4 GB RAM, 40 GB `/var`, Enforcing).

**1B.1 Log in and check:**

```bash
ssh -i ~/.ssh/id_lab azureuser@<LAB_NIM_PUBLIC_IP>
```

```bash
sudo -i
hostname
cat /etc/redhat-release
free -h | head -2
getenforce
ss -ltn | grep -E ':(80|443) ' || echo "80/443 free - good"
```

**1B.2 Copy the repository certificate, key and licence from your Mac:**

```bash
cd ~/<PROJECT_DIR>/licenses
scp -i ~/.ssh/id_lab "nginx-one-eval (5).crt" azureuser@<LAB_NIM_PUBLIC_IP>:~/nginx-repo.crt
scp -i ~/.ssh/id_lab "nginx-one-eval (6).key" azureuser@<LAB_NIM_PUBLIC_IP>:~/nginx-repo.key
scp -i ~/.ssh/id_lab "nginx-one-eval (6).jwt" azureuser@<LAB_NIM_PUBLIC_IP>:~/license.jwt
```

Then, on `lab-nim`:

```bash
mkdir -p /etc/ssl/nginx && chmod 700 /etc/ssl/nginx
mv /home/azureuser/nginx-repo.crt /home/azureuser/nginx-repo.key /etc/ssl/nginx/
mv /home/azureuser/license.jwt /root/license.jwt
chown root:root /etc/ssl/nginx/* /root/license.jwt
chmod 644 /etc/ssl/nginx/nginx-repo.crt
chmod 600 /etc/ssl/nginx/nginx-repo.key /root/license.jwt
restorecon -Rv /etc/ssl/nginx
```

**1B.3 Add the NIM repository and install 2.21.1:**

```bash
curl -fsSo /etc/yum.repos.d/nms.repo https://cs.nginx.com/static/files/nms.repo
dnf config-manager --set-disabled adm
rpm --import https://nginx.org/keys/nginx_signing.key
dnf makecache --repo nms
dnf --showduplicates list nms-instance-manager --repo nms 2>/dev/null | grep '2\.21'
dnf install --assumeno nms-instance-manager-2.21.1~2354667175
dnf install -y nms-instance-manager-2.21.1~2354667175
```

✅ Expect: the dry run shows `nms-instance-manager 2.21.1~2354667175`, plus RHEL's **open-source `nginx` 1.20.1** (NIM's web server, so no NGINX Plus licence is used), and **no `clickhouse-server`**. To match a CLIENT NIM on 2.21.0, use `nms-instance-manager-2.21.0~2144243032` instead.

```bash
rpm -q nms-instance-manager nginx
ls /usr/share/selinux/packages/nms.pp
systemctl list-unit-files 'nms*'
```

**1B.4 Set disconnected mode (as at CLIENT) and lightweight mode (no ClickHouse):**

```bash
cp -p /etc/nms/nms.conf /etc/nms/nms.conf.bak
sed -i 's/^\(\s*mode_of_operation:\) connected/\1 disconnected/' /etc/nms/nms.conf
sed -i '/^clickhouse:/,/enable:/ s/enable: true/enable: false/' /etc/nms/nms.conf
diff /etc/nms/nms.conf.bak /etc/nms/nms.conf
```

✅ Expect: `diff` shows exactly two changed lines: `mode_of_operation: disconnected` and, under `clickhouse:`, `enable: false`.

**1B.5 Load NIM's SELinux policy** (F5's steps for Enforcing systems):

```bash
semodule -n -i /usr/share/selinux/packages/nms.pp
/usr/sbin/load_policy
restorecon -F -R /usr/bin/nms-core /usr/bin/nms-dpm /usr/bin/nms-ingestion /usr/bin/nms-integrations
restorecon -F -R /usr/lib/systemd/system/nms.service /usr/lib/systemd/system/nms-core.service /usr/lib/systemd/system/nms-dpm.service /usr/lib/systemd/system/nms-ingestion.service /usr/lib/systemd/system/nms-integrations.service
restorecon -F -R /var/lib/nms /var/log/nms
semanage port -a -t nms_port_t -p tcp 10000
semanage port -a -t nms_port_t -p tcp 11000
setsebool -P httpd_can_network_connect 1
semodule -l | grep -w nms
```

**1B.6 Set the admin password, open HTTPS and start:**

```bash
read -rsp 'New NIM admin password: ' NIMPW; echo
bash /etc/nms/scripts/basic_passwords.sh admin "$NIMPW"
firewall-cmd --permanent --add-service=https
firewall-cmd --reload
systemctl enable --now nms nms-core nms-dpm nms-ingestion nms-integrations
systemctl enable --now nginx
restorecon -F -R /var/run/nms
sleep 15
systemctl is-active nms nms-core nms-dpm nms-ingestion nms-integrations nginx
```

✅ Expect: `active` six times. `nms-sm` (security monitoring) stays disabled and is not needed.

**1B.7 Check:**

```bash
curl -sk -u admin:"$NIMPW" https://127.0.0.1/api/platform/v1/health; echo
```

✅ Expect at this point: `{"code":1409,"message":"No valid license. ..."}`. This is **correct** for a disconnected NIM before its licence is added. It also proves the password works.

---

## 2. Add the licence to NIM (🧪 TRIAL ONLY)

🏭 CLIENT's NIM already has its licence. Skip this section in production (see 1A.4).

**2.1 Upload the JWT through the API:**

```bash
jq -n --arg t "$(cat /root/license.jwt)" '{metadata:{name:"license"},desiredState:{content:$t}}' > /root/license-body.json
curl -sk -u admin:"$NIMPW" -H 'Content-Type: application/json' --data @/root/license-body.json 'https://127.0.0.1/api/platform/v1/license?telemetry=true' | jq .
rm -f /root/license-body.json
```

✅ Expect: `"modeOfOperation": "DISCONNECTED"` and `"content": "********"`.

Alternative: in the web UI, go to **Settings → Licenses → Get Started → Browse → (JWT) → Add**.

**2.2 Check the licence and health:**

```bash
curl -sk -u admin:"$NIMPW" https://127.0.0.1/api/platform/v1/license | jq '[.. | .status? // empty | strings] | unique'
curl -sk -u admin:"$NIMPW" https://127.0.0.1/api/platform/v1/health | jq .
```

✅ Expect: `CONFIG_REPORT_READY`, then `"status": "healthy"` with `core`, `dpm`, `ingestion`, `integrations` and `secmon` all healthy. In 2.20/2.21 disconnected mode, uploading the JWT enables all features for 90 days. The first usage report must reach F5 within that time (section 5).

**2.3 Web UI.** From your Mac, open `https://<LAB_NIM_PUBLIC_IP>` (the NSG allows only your public IP), accept the self-signed certificate warning, log in as `admin`, and go to **Settings → Licenses**.

---

## 3. Point the NGINX nodes' usage reporting at NIM

The `mgmt {}` block lives in `nginx.conf`, which nginx-sync distributes. **Change it once on Node 1 and sync.**

🏭 **PROD ONLY.** CLIENT's `nginx.conf` already points at their NIM. **Do not change it.** Just confirm on Node 1:

```bash
grep -n -A4 '^mgmt' /etc/nginx/nginx.conf
```

✅ Expect: `usage_report endpoint=<NIM_IP>;` (or NIM's FQDN), normally with `ssl_verify off;` because NIM uses a self-signed certificate. New nodes inherit this through SOP-02 step 4.3 (`nginx.conf` copied before the first start) and through nginx-sync.

🧪 **TRIAL ONLY.** Switch the lab nodes from reporting directly to F5 to reporting to `lab-nim`. On `lab-n1`:

```bash
cp -p /etc/nginx/nginx.conf /root/nginx.conf.bak-$(date +%F-%H%M)
sed -i 's/usage_report endpoint=product.connect.nginx.com;/usage_report endpoint=lab-nim;/' /etc/nginx/nginx.conf
grep -n -A4 '^mgmt' /etc/nginx/nginx.conf
nginx -t && systemctl reload nginx
nginx-sync.sh -l /var/log/nginx-sync.log
nginx-sync.sh -C
```

✅ Expect: `usage_report endpoint=lab-nim;` with `ssl_verify off;`, `lab-n2` tested and reloaded, and the compare showing no differences.

---

## 4. Verify every node reports to NIM

**4.1 On the NIM server: requests from each node's IP:**

```bash
grep -E '<NODE_IP_REGEX>' /var/log/nginx/access.log | grep nginx-usage | tail -12
```

- 🧪 `<NODE_IP_REGEX>` = `10\.50\.1\.(4|6)`
- 🏭 Use the six node IPs, for example `10\.80\.42\.(11|12|<n3>|<n4>|<n5>|<n6>)`. This needs shell access to the NIM server. If CLIENT cannot give it, ask them to run it, or use the NIM UI's usage/licensing pages.

✅ Expect, for **every** node: `"POST /api/nginx-usage HTTP/1.1" 200 … "nginx/1.29.3 (nginx-plus-r36-p8)"`. A node reports within seconds of a reload or start, then periodically.

🧪 Lab result (16 Oct 2026): `10.50.1.4` at 16:50:19 and `10.50.1.6` at 16:50:35, both `200`.

**4.2 On each node:**

```bash
curl -s http://127.0.0.1/api/9/license | jq '{eval, active_till, reporting}'
grep -iE 'usage|mgmt' /var/log/nginx/error.log | tail -3
```

✅ Expect: `reporting.healthy: true`, `fails: 0`, and no recent usage-report errors. Each node has its own `uuid`. 🏭 Nodes 3–6 show the **new** subscription's `active_till`, and `eval: false`.

If a node is missing from 4.1:
- Check its `mgmt` block (`grep -A4 '^mgmt' /etc/nginx/nginx.conf`). It must match Node 1. Run `nginx-sync.sh -C`.
- Check it can reach NIM on 443: `curl -sk -o /dev/null -w '%{http_code}\n' https://<NIM>/`.
- Check SELinux: `getsebool httpd_can_network_connect` should be `on` (SOP-02 decision table).

UAT: TC_005, TC_035.

---

## 5. Usage-report cycle for NIM 2.21 (disconnected)

🏭 **This cycle is CLIENT's responsibility** (with F5 / the reseller). We may demonstrate it, or support them, so the new nodes appear in the next report. Steps 5.2–5.4 run on a machine with internet access. The report file is carried across by CLIENT's approved transfer method.

**5.1 Download the report from NIM** (validated 🧪):

```bash
curl -sk -u admin:"$NIMPW" -o /root/report.zip 'https://<NIM>/api/platform/v1/report/download?format=zip&reportType=initial'
ls -l /root/report.zip
python3 -m zipfile -l /root/report.zip
```

- `reportType=initial` is for the **first** report after the licence is added (status `CONFIG_REPORT_READY`). For later periodic reports use `reportType=telemetry&telemetryAction=download`.
- ✅ Expect: a zip containing `report.json`, a list of entries, each with a `deviceId` and a signed `report`.
- 🧪 Lab finding: the **initial** report contains 2 entries, both `"reportType": "initialize"` / `"digitalAssetName": "nim"`. They describe **NIM's own activation**, not NGINX Plus instance usage. One carries the subscription ID in `cat`. Instance usage is expected in the later periodic (`telemetry`) reports. That has not yet been seen in the lab.

To read it (it contains instance and subscription identifiers: treat CLIENT's as client data):

```bash
python3 -c "import zipfile,json,base64;d=json.loads(zipfile.ZipFile('/root/report.zip').read('report.json'));[print(e.get('cat','-'),json.dumps(json.loads(base64.urlsafe_b64decode(e['report'].split('.')[1]+'='*(-len(e['report'].split('.')[1])%4))),indent=1)) for e in d]"
```

**5.2 🤝 WITH F5 SUPPORT ONSITE. Submit to F5** (internet-connected machine, using the subscription's JWT):

```bash
curl -s --location 'https://product.apis.f5.com/ee/v1/entitlements/telemetry/bulk' -H "Authorization: Bearer $(cat <PATH_TO_license.jwt>)" --form 'file=@"report.zip"' | tee submit.json | jq .
```

Note the report ID (UUID) in the reply.

**5.3 🤝 WITH F5 SUPPORT ONSITE. Check the status and download the acknowledgement:**

```bash
curl -s --location "https://product.apis.f5.com/ee/v1/entitlements/telemetry/bulk/status/<REPORT_ID>" -H "Authorization: Bearer $(cat <PATH_TO_license.jwt>)" | jq .
curl -s --location "https://product.apis.f5.com/ee/v1/entitlements/telemetry/bulk/download/<REPORT_ID>" -H "Authorization: Bearer $(cat <PATH_TO_license.jwt>)" --output acknowledgement.zip
```

**5.4 🤝 WITH F5 SUPPORT ONSITE. Upload the acknowledgement into NIM:**

```bash
curl -sk -u admin:"$NIMPW" --location 'https://<NIM>/api/platform/v1/report/upload' --form 'file=@"acknowledgement.zip"'
curl -sk -u admin:"$NIMPW" https://<NIM>/api/platform/v1/license | jq '[.. | .status? // empty | strings] | unique'
```

Questions to settle with F5 support before running 5.2–5.4:
- With **two subscriptions**, which JWT authenticates 5.2/5.3 (one submission per subscription, or either JWT)?
- Is a fresh `telemetry` report needed after the 4 new nodes start reporting, so they appear in F5's records?
- Who at CLIENT owns the cycle going forward, and how often does it run?

---

## 6. NGINX Agent: Instances inventory (only if CLIENT uses it)

⚠️ **NOT YET VALIDATED.** Do this section only if 1A.3 showed `nginx-agent` on CLIENT's existing nodes.

- The agent installer is served **by NIM itself**, so it works without internet and matches NIM 2.21. On each new node:

  ```bash
  curl -k https://<NIM>/install/nginx-agent | sh
  systemctl enable --now nginx-agent
  systemctl is-active nginx-agent
  ```

- ✅ Expect: the node appears under **Instances** in NIM, next to Nodes 1–2.
- ❌ **Do not publish configuration from NIM** to these nodes. nginx-sync remains the only tool that pushes config, otherwise the two overwrite each other.
- SELinux, the agent's version and its config (`/etc/nginx-agent/nginx-agent.conf`) should match Node 1's. Compare them before going live.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `{"code":1409,"message":"No valid license..."}` | Disconnected NIM with no licence, or an expired one | 🧪 Section 2. 🏭 Raise with CLIENT |
| Services not `active` after 1B.6 | Usually SELinux or config | `journalctl -u <service> -n 20 --no-pager`. `ausearch -m avc -ts recent`. `diff /etc/nms/nms.conf.bak /etc/nms/nms.conf` |
| Nodes not listed under **Instances** | Usage reporting does not register instances | Expected without the agent. See section 0 and section 6 |
| No `POST /api/nginx-usage` from a node | Wrong `mgmt` endpoint, no route to NIM on 443, or SELinux boolean off | Section 4 checks |
| `ssl_verify` errors in a node's error log | NIM's self-signed certificate | `ssl_verify off;` in `mgmt {}` (as in non-prod) |
| Report over-counts usage | NIM 2.21.0 bug 46867 | Fixed in 2.21.1. Raise with CLIENT |
| `adm` repo errors during `dnf` (🧪) | App Delivery Manager repo enabled by `nms.repo` | `dnf config-manager --set-disabled adm` |

---

## Record sheet

| Item | 🧪 Trial reference (16 Oct 2026) | 🏭 Value at CLIENT |
|---|---|---|
| NIM host / version (`rpm -q`) | lab-nim / 2.21.1-2354667175 | <NIM_IP> / 2.21.___ |
| Mode / ClickHouse | disconnected / disabled (lightweight) | disconnected / ___ |
| NIM health (1A.2 / 2.2) | healthy | |
| Licence status | CONFIG_REPORT_READY | |
| Agent in use on existing nodes (1A.3) | no | |
| `mgmt` endpoint on Node 1 (3) | lab-nim, ssl_verify off | |
| Nodes reporting `200` (4.1) | 10.50.1.4, 10.50.1.6 | Node 1 ☐ 2 ☐ 3 ☐ 4 ☐ 5 ☐ 6 ☐ |
| Node-side reporting healthy (4.2) | lab-n1 ✅ lab-n2 ✅ | |
| Initial report downloaded (5.1) | ✅ 2 entries, both NIM activation (`initialize`) | |
| Report submitted + acknowledgement uploaded (5.2–5.4) | not practised (by decision; onsite with F5 support) | |
| New subscription JWT uploaded to NIM? (1A.4) | n/a | Only if CLIENT/F5 confirm |
