# SOP-04: NGINX Instance Manager (NIM): Usage Reporting for the New Nodes

| | |
|---|---|
| Project | CLIENT NGINX Plus: additional nodes (2 → 6) |
| Phase | 4 of 4: NIM usage reporting for the new nodes. (1: SOP-01 bundle. 2: SOP-02 install and start. 3: SOP-03 nginx-sync, tests and F5 pools) |
| Scope | CLIENT's **existing** NIM (`<NIM_IP>`, **version 2.21**, disconnected). **We do not install or reconfigure NIM.** We check it, confirm all six nodes report to it, and support CLIENT's usage-report cycle |
| Status | Sections 1–4 (checks, reporting, report download) validated against a NIM 2.21.1 in the Azure lab (16 Oct 2026). Section 4.2–4.4 (submission to F5) is from F5's documentation and is run **only with F5 support**. |

---

## How you work

SSH to the nodes as **`<ADMIN_USER>`**, then `sudo su` for root. Check the host name in the prompt before every step, and paste **one line at a time**. NIM server access, and the NIM admin credentials, come from CLIENT.

**Passwords:** never write a real password into this SOP, a ticket or a chat. `NIMPW` is read from the keyboard, so it does not appear on screen or in shell history:

```bash
read -rsp 'NIM admin password: ' NIMPW; echo
```

**Markers:**

| Marker | Meaning |
|---|---|
| 🤝 **WITH F5 SUPPORT** | Submits data to F5's licensing service. Run only together with F5 support and CLIENT, using CLIENT's licence JWT |
| ⚠️ **NOT VALIDATED** | From F5's documentation, not yet run in the lab |

---

## 0. Background

**Two different ways NGINX Plus talks to NIM:**

| | Usage reporting | NGINX Agent |
|---|---|---|
| What | NGINX Plus sends licence usage, set by `mgmt { usage_report endpoint=<NIM>; }` in `nginx.conf` | A separate `nginx-agent` service on each node, registered with NIM |
| NIM shows | Licensing / usage data. The nodes do **not** appear in the Instances list | The **Instances** inventory (version, status, config) and metrics |
| Needed for | **Licensing compliance** (R33 and later must report) | Inventory and monitoring |
| This SOP | Sections 2–4 | Not used at CLIENT (confirmed by CLIENT). Checked in 1.3 only |

**NIM 2.21 facts that matter here** (F5 release notes):
- **Multiple subscriptions are supported from 2.20:** *"a single NGINX Instance Manager can receive usage from these instances and send a bulk usage report to F5."* Nodes 1–2 (existing subscription) and Nodes 3–6 (new subscription) report to the same NIM.
- **Disconnected mode on 2.20/2.21 needs a manual cycle:** download the report from NIM, submit it to F5 from a machine with internet access, download F5's acknowledgement, upload it back into NIM (section 4). From 2.22 the acknowledgement upload is no longer needed.
- **2.21.1 fixed** *"Usage reporting shows higher values than actual consumption (46867)."* On 2.21.0, reports can over-count, and going from 2 to 6 nodes makes this more visible.

---

## 1. Check CLIENT's NIM (read-only)

**1.1 Exact version.** On the NIM server:

```bash
rpm -q nms-instance-manager
```

Or in the NIM web UI, look at the version in the help / about area.

- `2.21.1-…`: nothing to raise.
- `2.21.0-…`: tell CLIENT about the over-counting fix in 2.21.1 (issue 46867). Upgrading is CLIENT's decision and is out of scope.

**1.2 Licence and health.** From Node 1, or any machine that can reach NIM:

```bash
read -rsp 'NIM admin password: ' NIMPW; echo
```

```bash
curl -sk -u admin:"$NIMPW" https://<NIM_IP>/api/platform/v1/health | jq .
```

```bash
curl -sk -u admin:"$NIMPW" https://<NIM_IP>/api/platform/v1/license | jq '{mode: .currentStatus.modeOfOperation, status: [.. | .status? // empty | strings] | unique}'
```

✅ Expect: `"status": "healthy"`, `mode` `DISCONNECTED`, and a licence status such as `CONFIG_REPORT_READY` or `INITIALIZE_ACTIVATION_COMPLETE`. **Settings → Licenses** in the web UI shows **one** licence: NIM's own. That is expected. The new subscription is confirmed in the report (4.1b), not on this page.

🛑 `{"code":1409,"message":"No valid license..."}` means NIM's own licence is missing or has lapsed. Raise it with CLIENT.

**1.3 Is the NGINX Agent in use?** On Node 1:

```bash
rpm -q nginx-agent; systemctl is-active nginx-agent
```

Also look at **Instances** in the NIM web UI.

✅ Expect: `package nginx-agent is not installed`, and Nodes 1–2 **not** listed under Instances. CLIENT has confirmed they do not use the agent, so the new nodes do not get it either.

🛑 If Node 1 does run `nginx-agent`, or Nodes 1–2 appear under Instances, stop and confirm with CLIENT before going on. The new nodes would then need the agent too, which this SOP does not cover.

**1.4 The new subscription's JWT.** Do **not** upload the new subscription's `license.jwt` to NIM unless CLIENT or F5 confirm it is required. Settle this with F5 support together with the questions in section 4.

---

## 2. Confirm the nodes' usage-report target

The `mgmt {}` block in `nginx.conf` tells each node where to report. CLIENT's `nginx.conf` already points at their NIM. **Do not change it.** On Node 1:

```bash
grep -n -A4 '^mgmt' /etc/nginx/nginx.conf
```

✅ Expect: `usage_report endpoint=<NIM_IP>;` (or NIM's FQDN), normally with `ssl_verify off;` because NIM uses a self-signed certificate.

The new nodes received this block with Node 1's configuration (SOP-02 4.4–4.5), and nginx-sync keeps it identical (SOP-03). Confirm on **each new node**:

```bash
grep -n -A4 '^mgmt' /etc/nginx/nginx.conf
```

✅ Expect: the same lines as on Node 1.

---

## 3. Verify every node reports to NIM

**3.1 On each node:**

```bash
curl -s http://127.0.0.1/api/9/license | jq '{eval, active_till, reporting}'
```

```bash
grep -iE 'usage|mgmt' /var/log/nginx/error.log | tail -3
```

✅ Expect: `reporting.healthy: true`, `fails: 0`, and no recent usage-report errors. Each node has its own `uuid`. Nodes 3–6 show the **new** subscription's `active_till`, and `eval: false`.

**3.2 On the NIM server: requests from each node's IP.** Put the six node IPs in the pattern:

```bash
grep -E '<NODE1_IP>|<NODE2_IP>|<NODE3_IP>|<NODE4_IP>|<NODE5_IP>|<NODE6_IP>' /var/log/nginx/access.log | grep nginx-usage | tail -12
```

✅ Expect, for **every** node: `"POST /api/nginx-usage HTTP/1.1" 200 …` with the NGINX Plus version (`nginx/1.29.3 (nginx-plus-r36-p8)`). A node reports within seconds of a start or reload, then periodically.

If CLIENT cannot give shell access to the NIM server, ask them to run this command, or use the NIM UI's usage / licensing pages.

**3.3 If a node is missing from 3.2:**
- Its `mgmt` block must match Node 1's (section 2). On Node 1, run `nginx-sync.sh -C`.
- It must reach NIM on port 443. On that node: `curl -sk -o /dev/null -w '%{http_code}\n' https://<NIM_IP>/`
- SELinux must allow it: `getsebool httpd_can_network_connect` must be `on` (SOP-02 4.1).

UAT: TC_11.

---

## 4. Usage-report cycle for NIM 2.21 (disconnected)

**This cycle is CLIENT's responsibility** (with F5 and their reseller). We demonstrate it, or support them, so the new nodes appear in the next report. Steps 4.2–4.4 run on a machine with internet access. The report file is carried across by CLIENT's approved transfer method.

**4.1 Download the report from NIM:**

```bash
curl -sk -u admin:"$NIMPW" -o /root/report.zip 'https://<NIM_IP>/api/platform/v1/report/download?format=zip&reportType=telemetry&telemetryAction=download'
```

```bash
ls -l /root/report.zip; python3 -m zipfile -l /root/report.zip
```

- Use `reportType=telemetry&telemetryAction=download` for the periodic reports. `reportType=initial` is only for the **first** report after NIM's licence is added (licence status `CONFIG_REPORT_READY`).
- ✅ Expect: a zip containing `report.json`: a list of entries, each with a `deviceId` and a signed `report`.

The report contains instance and subscription identifiers: treat it as **client data**. To read it:

```bash
python3 -c "import zipfile,json,base64;d=json.loads(zipfile.ZipFile('/root/report.zip').read('report.json'));[print(e.get('cat','-'),json.dumps(json.loads(base64.urlsafe_b64decode(e['report'].split('.')[1]+'='*(-len(e['report'].split('.')[1])%4))),indent=1)) for e in d]"
```

In the lab, the **initial** report contained only NIM's own activation entries (`"reportType": "initialize"`, `"digitalAssetName": "nim"`), not NGINX Plus instance usage. Instance usage is expected in the periodic `telemetry` reports.

**4.1b Confirm both subscriptions, and all six nodes, are in the report.** NIM's **Settings → Licenses** page shows only **NIM's own licence**. That is expected, and it does not mean the new subscription is missing. The report is the proof. Download it **after** the new nodes have been reporting for a while (section 3).

Collect each node's subscription and usage ID. On **each node** (Node 1 to Node 6):

```bash
python3 -c "import base64,json;p=open('/etc/nginx/license.jwt').read().split('.')[1];print('sub:',json.loads(base64.urlsafe_b64decode(p+'='*(-len(p)%4)))['sub'])"; curl -s http://127.0.0.1/api/9/license | jq -r '"uuid: " + ([.. | .uuid? // empty][0] // "-")'
```

Record the `sub` and `uuid` for each node in the record sheet. Nodes 1–2 share one `sub`, Nodes 3–6 share the other, and every `uuid` is different.

Where the report was downloaded, decode it into a text file:

```bash
python3 -c "import zipfile,json,base64;d=json.loads(zipfile.ZipFile('/root/report.zip').read('report.json'));print('\n'.join(json.dumps(e)+json.dumps(json.loads(base64.urlsafe_b64decode(e['report'].split('.')[1]+'='*(-len(e['report'].split('.')[1])%4)))) for e in d))" > /root/report-decoded.txt; wc -l /root/report-decoded.txt
```

Search for both subscriptions:

```bash
for s in <SUB_OLD> <SUB_NEW>; do echo -n "$s: "; grep -c "$s" /root/report-decoded.txt; done
```

Search for every node's usage ID:

```bash
for u in <UUID_N1> <UUID_N2> <UUID_N3> <UUID_N4> <UUID_N5> <UUID_N6>; do echo -n "$u: "; grep -c "$u" /root/report-decoded.txt; done
```

✅ Expect:
- **Both subscriptions found** (1 or more records each). At CLIENT on 27 Oct 2026 the report held 3 records: 2 for the old subscription and 1 for the new one. The number of records does not have to match the number of nodes: records can be per subscription, period or type.
- **All six usage IDs found.** If the IDs are not in the report, it may summarise per subscription instead. Then look in each subscription's record for an instance count, and check it shows 2 (old) and 4 (new).

🛑 If the new subscription is missing: check that Nodes 3–6 report (section 3), wait for the next reporting period, then download a fresh report. If it is still missing, raise it with F5 support. **Do not** upload the new subscription's JWT to NIM to "fix" it (1.4).

Delete `/root/report-decoded.txt` when finished. Like the report, it contains CLIENT's subscription and instance IDs.

UAT: TC_11 (record "report contains both subscriptions, all 6 nodes").

**Questions to settle with F5 support before 4.2–4.4:**
- The report contains both subscriptions (4.1b). Which JWT authenticates 4.2 and 4.3: one submission covering both, or one submission per subscription?
- Does the new subscription's JWT need to be uploaded to NIM (1.4), or is it enough that its usage is in the report?
- Who at CLIENT owns the cycle from now on, and how often does it run?

**4.2 🤝 WITH F5 SUPPORT. Submit the report to F5** (internet-connected machine, with CLIENT's subscription JWT):

```bash
curl -s --location 'https://product.apis.f5.com/ee/v1/entitlements/telemetry/bulk' -H "Authorization: Bearer $(cat <PATH_TO_license.jwt>)" --form 'file=@"report.zip"' | tee submit.json | jq .
```

Note the report ID (UUID) in the reply.

**4.3 🤝 WITH F5 SUPPORT. Check the status and download the acknowledgement:**

```bash
curl -s --location "https://product.apis.f5.com/ee/v1/entitlements/telemetry/bulk/status/<REPORT_ID>" -H "Authorization: Bearer $(cat <PATH_TO_license.jwt>)" | jq .
```

```bash
curl -s --location "https://product.apis.f5.com/ee/v1/entitlements/telemetry/bulk/download/<REPORT_ID>" -H "Authorization: Bearer $(cat <PATH_TO_license.jwt>)" --output acknowledgement.zip
```

**4.4 🤝 WITH F5 SUPPORT. Upload the acknowledgement into NIM:**

```bash
curl -sk -u admin:"$NIMPW" --location 'https://<NIM_IP>/api/platform/v1/report/upload' --form 'file=@"acknowledgement.zip"'
```

```bash
curl -sk -u admin:"$NIMPW" https://<NIM_IP>/api/platform/v1/license | jq '[.. | .status? // empty | strings] | unique'
```

✅ Expect: the upload accepted, and the licence status no longer waiting for a report.

Delete the copies of `report.zip`, `submit.json` and `acknowledgement.zip` from every machine they were placed on, once CLIENT has their own copy.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `{"code":1409,"message":"No valid license..."}` | NIM's own licence missing or expired | Raise with CLIENT |
| Nodes not listed under **Instances** | Usage reporting does not register instances | Expected: CLIENT does not use the NGINX Agent (1.3) |
| No `POST /api/nginx-usage` from a node | Wrong `mgmt` endpoint, no route to NIM on 443, or SELinux boolean off | Checks in 3.3 |
| `ssl_verify` errors in a node's error log | NIM's self-signed certificate | `ssl_verify off;` in `mgmt {}`, as on Node 1 |
| Report over-counts usage | NIM 2.21.0 bug 46867 | Fixed in 2.21.1. Raise with CLIENT |

---

## Record sheet

| Item | Value |
|---|---|
| Date / performed by | |
| NIM version (`rpm -q`) (1.1) | 2.21.___ |
| NIM health / mode / licence status (1.2) | |
| Agent not installed on Node 1, nodes not under Instances (1.3) | |
| New subscription JWT uploaded to NIM? (1.4) | Only if CLIENT / F5 confirm |
| `mgmt` endpoint on Node 1, same on new nodes (2) | |
| Node-side reporting healthy (3.1) | Node 1 ☐ 2 ☐ 3 ☐ 4 ☐ 5 ☐ 6 ☐ |
| NIM receives `200` from each node (3.2) | Node 1 ☐ 2 ☐ 3 ☐ 4 ☐ 5 ☐ 6 ☐ |
| Report downloaded (4.1) | |
| `sub` / `uuid` per node (4.1b) | N1: ___ N2: ___ N3: ___ N4: ___ N5: ___ N6: ___ |
| Report: records per subscription / all 6 usage IDs found (4.1b) | old: ___ new: ___ / ☐ |
| F5 answers to the section 4 questions | |
| Report submitted / acknowledgement uploaded, with F5 support (4.2–4.4) | |
