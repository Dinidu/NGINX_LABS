# SOP-05: Monitoring and Logging Integration (Filebeat and AppDynamics) for the New Nodes

| | |
|---|---|
| Project | CLIENT NGINX Plus: additional nodes (2 → 6) |
| Purpose | Give each new node the same log shipping (**Filebeat** → ELK) and monitoring (**AppDynamics Machine Agent**) as Node 1 |
| Runs on | **Node 1** (<NODE1_IP>, read-only checks and copying files) and each **new node** (Nodes 3–6) |
| Before this | SOP-02 finished on the new node (NGINX running with Node 1's configuration) |
| Status | ⚠️ **Not validated in the lab** (the lab has no ELK or AppDynamics). Every part starts by reading Node 1's real setup. If Node 1 differs from what a step expects, stop and agree the next step with the ELK / AppDynamics owners |

---

## How you work on CLIENT nodes

SSH in as **`<ADMIN_USER>`**, then `sudo su` for root. Check the host name in the prompt before every step, and paste **one line at a time**.

**Rules:**
- ✅ Install the **same versions** as Node 1, from packages the ELK / AppDynamics teams supply (the nodes have no internet).
- ❌ Never print or paste access keys, passwords or keystore contents into chat, tickets or this SOP. Commands below hide them.
- ❌ Don't copy Filebeat's **registry** (`/var/lib/filebeat/registry`) from Node 1. It records how far Node 1 has read *its own* logs. Copied, it would make the new node skip or mis-read its logs.
- ❌ Don't give a new node Node 1's AppDynamics **unique host ID**. Each node must appear as its own host in the Controller (step 4.3).
- ❌ Don't change anything on Node 1. It is only read and copied from.

---

## 1. On Node 1: record the existing setup (read-only)

### 1.1 Filebeat

```bash
rpm -q filebeat; systemctl is-enabled filebeat; systemctl is-active filebeat
```

```bash
ls -la /etc/filebeat/ /etc/filebeat/modules.d/ 2>/dev/null | grep -v '\.disabled$'
```

```bash
filebeat modules list 2>/dev/null | sed -n '/Enabled:/,/Disabled:/p'
```

```bash
grep -vE '^\s*#|^\s*$' /etc/filebeat/filebeat.yml | sed -E 's/((password|api_key|token)\s*:\s*).*/\1<hidden>/I'
```

```bash
filebeat keystore list 2>/dev/null || echo "no keystore"
```

Files the config points to (CA and client certificates, extra inputs):

```bash
grep -rhoE '/[A-Za-z0-9._/-]+\.(crt|pem|key|yml|yaml)' /etc/filebeat/ | sort -u
```

✅ Record: Filebeat version, enabled modules (for example `nginx`), the inputs and log paths (`/var/log/nginx/*.log`), the output (`output.logstash` hosts and port, or `output.elasticsearch`), the service tag / `fields`, keystore keys (names only), and every certificate path.

🛑 If Filebeat is not installed on Node 1, or the output is something other than Logstash or Elasticsearch, stop and agree the approach with the ELK team.

### 1.2 AppDynamics

Find how the agent is installed:

```bash
rpm -qa | grep -iE 'appd|machine-?agent'; systemctl list-unit-files | grep -iE 'appd|machine'
```

```bash
ps -eo pid,user,args | grep -iE '[m]achineagent|[a]ppdynamics' | cut -c1-200
```

```bash
ls -d /opt/appdynamics/* 2>/dev/null
```

✅ Expect the **AppDynamics Machine Agent**: an RPM such as `appdynamics-machine-agent-<version>`, or a folder like `/opt/appdynamics/machine-agent`, running as a service. Set its folder for the next commands (adjust to what you found):

```bash
MA=/opt/appdynamics/machine-agent; ls $MA; cat $MA/version* 2>/dev/null | head -3
```

Controller connection settings (the access key is hidden):

```bash
grep -E '<(controller-host|controller-port|controller-ssl-enabled|account-name|sim-enabled|unique-host-id|application-name|tier-name|node-name|machine-path)>' $MA/conf/controller-info.xml
```

```bash
grep -c '<account-access-key>' $MA/conf/controller-info.xml; ls -l /etc/sysconfig/appdynamics-machine-agent 2>/dev/null
```

NGINX monitoring extension (how AppDynamics reads NGINX metrics):

```bash
ls $MA/monitors/
```

```bash
grep -rhiE 'url|uri|host|port|status|/api/' $MA/monitors/*/config.yml $MA/monitors/*/monitor.xml 2>/dev/null | grep -viE 'password|key' | sort -u
```

✅ Record: install type (RPM or folder), version, service name, `controller-host` / `controller-port` / SSL, `account-name`, `sim-enabled`, whether `unique-host-id` is set, any application / tier / node names, and the NGINX extension folder with the URL it polls (for example `http://localhost/api/...` or a status page).

🛑 If Node 1 runs a different AppDynamics component (for example an app server agent, or nothing at all), stop and agree the approach with the AppDynamics owner.

### 1.3 Licences and packages (with the owners)

- **AppDynamics:** each Machine Agent uses a licence in the Controller. Ask the AppDynamics owner to confirm there are licences for **4 more** hosts.
- **Packages:** ask for the **same versions** as Node 1: the Filebeat RPM (`filebeat-<version>-x86_64.rpm`) and the Machine Agent package (RPM, or the zip bundle). If they cannot supply them, look for the original files on Node 1:

```bash
find / \( -path /proc -o -path /sys -o -path /run \) -prune -o \( -iname 'filebeat*.rpm' -o -iname '*machine*agent*' -o -iname 'appdynamics*' \) -type f -print 2>/dev/null | grep -viE '/logs?/' | head -20
```

---

## 2. On each new node: check it can reach ELK and AppDynamics

Use the hosts and ports from 1.1 (Logstash or Elasticsearch) and 1.2 (Controller):

```bash
for t in <LOGSTASH_HOST>:<PORT> <CONTROLLER_HOST>:<PORT>; do h=${t%:*}; p=${t##*:}; if [[ ! $h =~ ^[0-9.]+$ ]] && ! getent hosts $h >/dev/null; then echo "$t  NO-DNS"; elif timeout 5 bash -c "</dev/tcp/$h/$p" 2>/dev/null; then echo "$t  OPEN"; else echo "$t  BLOCKED/CLOSED"; fi; done
```

✅ Expect: `OPEN` for every target. `NO-DNS` or `BLOCKED/CLOSED`: run the same line on Node 1. If Node 1 shows `OPEN`, ask CLIENT's network team to open the same rule for the new node.

Copy the packages from 1.3 to the new node into `/home/<ADMIN_USER>/` (CLIENT's approved transfer method, or `scp` as `<ADMIN_USER>` from Node 1).

---

## 3. Filebeat on the new node

**3.1 Install the same version, offline:**

```bash
dnf --disablerepo='*' install -y /home/<ADMIN_USER>/filebeat-<VERSION>-x86_64.rpm
```

```bash
rpm -q filebeat
```

✅ Expect: the same version as Node 1 (1.1).

**3.2 On Node 1: pack Filebeat's configuration** (and its keystore, if 1.1 showed one), plus the certificate files from 1.1. Set `CERTS` to the certificate paths found (or leave it empty):

```bash
CERTS=""; KS=""; [ -f /var/lib/filebeat/filebeat.keystore ] && KS=/var/lib/filebeat/filebeat.keystore; echo "Packing: /etc/filebeat $KS $CERTS"
```

```bash
tar czf /tmp/node1-filebeat.tgz /etc/filebeat $KS $CERTS
```

```bash
chown <ADMIN_USER>:<ADMIN_USER> /tmp/node1-filebeat.tgz; chmod 600 /tmp/node1-filebeat.tgz
```

```bash
su - <ADMIN_USER> -c 'scp /tmp/node1-filebeat.tgz <ADMIN_USER>@<NEW_NODE>:~/'
```

The archive contains credentials. Delete it from Node 1 once every new node has it: `rm -f /tmp/node1-filebeat.tgz`.

**3.3 On the new node: unpack and label:**

```bash
tar czf /root/etc-filebeat-default-$(date +%F-%H%M).tgz /etc/filebeat
```

```bash
tar xzf /home/<ADMIN_USER>/node1-filebeat.tgz -C / && rm -f /home/<ADMIN_USER>/node1-filebeat.tgz
```

```bash
restorecon -Rv /etc/filebeat /var/lib/filebeat; ls -l /etc/filebeat/
```

If the config sets a fixed host name or node name (for example `name:` or a `fields` entry with Node 1's host name), change it to this node's name:

```bash
grep -nE '^\s*name:|hostname|<NODE1_HOSTNAME>' /etc/filebeat/filebeat.yml
```

**3.4 Test, then start:**

```bash
filebeat test config
```

```bash
filebeat test output
```

```bash
filebeat keystore list 2>/dev/null
```

✅ Expect: `Config OK`, the output test showing the connection and TLS handshake to Logstash / Elasticsearch as `OK`, and the same keystore key names as Node 1.

⚠️ If the keystore keys are missing, or the output fails with an authentication error, the copied keystore may not have worked on this node. Recreate each key with the value from the ELK team: `filebeat keystore create --force`, then `filebeat keystore add <KEY_NAME>` (it prompts for the value; do not type it on the command line).

```bash
systemctl enable --now filebeat; systemctl is-active filebeat
```

```bash
journalctl -u filebeat -n 20 --no-pager | grep -iE 'error|harvester|connection|publish' | tail -10
```

✅ Expect: `active`, harvesters started for `/var/log/nginx/access.log` and `error.log`, and no connection errors.

**3.5 Prove it end to end.** Send a request you can search for:

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1/sop05-filebeat-$(hostname -s)
```

✅ Expect: in Kibana (ELK team), the request `/sop05-filebeat-<hostname>` under the same service tag / index as Node 1's logs, with this node's host name.

---

## 4. AppDynamics Machine Agent on the new node

**4.1 Install the same version as Node 1.**

If Node 1 used an **RPM** (1.2):

```bash
dnf --disablerepo='*' install -y /home/<ADMIN_USER>/<MACHINE_AGENT_PACKAGE>.rpm
```

If Node 1 used a **folder** (zip bundle): unpack the same bundle into the same folder, for example:

```bash
mkdir -p /opt/appdynamics/machine-agent && cd /opt/appdynamics/machine-agent && unzip -q /home/<ADMIN_USER>/<MACHINE_AGENT_BUNDLE>.zip
```

(`unzip` must be installed: `rpm -q unzip`.) The same service as Node 1 arrives with the configuration in 4.2 (its unit file is included in the archive).

**4.2 On Node 1: pack the agent's configuration and extensions** (not its logs or local data):

```bash
MA=/opt/appdynamics/machine-agent; SVC=$(systemctl list-unit-files | grep -ioE '[a-z0-9-]*(appd|machine)[a-z0-9-]*\.service' | head -1); echo "$MA $SVC"
```

```bash
tar czf /tmp/node1-appd.tgz $MA/conf $MA/monitors $(ls /etc/sysconfig/appdynamics-machine-agent 2>/dev/null) $(systemctl show -p FragmentPath --value $SVC 2>/dev/null)
```

```bash
chown <ADMIN_USER>:<ADMIN_USER> /tmp/node1-appd.tgz; chmod 600 /tmp/node1-appd.tgz; su - <ADMIN_USER> -c 'scp /tmp/node1-appd.tgz <ADMIN_USER>@<NEW_NODE>:~/'
```

The archive contains the Controller access key. Delete it from Node 1 once every new node has it: `rm -f /tmp/node1-appd.tgz`.

**4.3 On the new node: unpack, then make it this node's own:**

```bash
tar xzf /home/<ADMIN_USER>/node1-appd.tgz -C / && rm -f /home/<ADMIN_USER>/node1-appd.tgz
```

```bash
restorecon -Rv /opt/appdynamics /etc/systemd/system 2>/dev/null | tail -3; systemctl daemon-reload
```

Check the host identity settings (set `MA` to the same folder as on Node 1):

```bash
MA=/opt/appdynamics/machine-agent; grep -nE '<(unique-host-id|node-name|machine-path)>' $MA/conf/controller-info.xml
```

- `<unique-host-id>` **empty**: leave it. The agent uses this node's host name.
- `<unique-host-id>` set to **Node 1's name**: change it to this node's host name (`hostname -s`), in the same style as Node 1's value. Do the same for `<node-name>` if it is set.

```bash
vi $MA/conf/controller-info.xml
```

If the NGINX extension (1.2) polls a URL with Node 1's host name or IP, change it to `localhost` / `127.0.0.1` in the same file under `$MA/monitors/<EXTENSION>/`.

**4.4 Start and check registration:**

```bash
systemctl enable --now <SERVICE>; systemctl is-active <SERVICE>
```

```bash
sleep 60; grep -iE 'started|registered|error|exception' $MA/logs/machine-agent.log | tail -10
```

✅ Expect: `active`, the log shows the agent started and registered with the Controller, and no repeated errors (for example `401`, `licen`, `connection refused`).

**4.5 Prove it end to end.** In the AppDynamics Controller (AppDynamics owner):
- The new node appears under **Servers** with its **own** host name, alongside Node 1 and Node 2, reporting CPU / memory.
- The NGINX extension metrics appear for the new node in the **Metric Browser** under the same path as Node 1's (for example *Custom Metrics → NGINX*).

---

## 5. Clean up and record

On the new node: `rm -f /home/<ADMIN_USER>/*.rpm /home/<ADMIN_USER>/*.zip` (once the installs are confirmed).

On Node 1, after all new nodes are done: `rm -f /tmp/node1-filebeat.tgz /tmp/node1-appd.tgz`.

**Rollback (per node):**

| Component | Action |
|---|---|
| Filebeat | `systemctl disable --now filebeat`, then `dnf remove -y filebeat`. Restore `/root/etc-filebeat-default-<date>.tgz` if needed |
| AppDynamics | `systemctl disable --now <SERVICE>`, then `dnf remove -y <package>` (RPM), or remove the agent folder and unit file (zip). Ask the AppDynamics owner to remove the host from the Controller |

---

## Record sheet (one per node)

| Item | Value |
|---|---|
| Date / performed by | |
| New node hostname / IP | |
| Node 1: Filebeat version / modules / output host:port (1.1) | |
| Node 1: Machine Agent version / install type / service / Controller host:port (1.2) | |
| AppDynamics licences for 4 hosts confirmed by (1.3) | |
| ELK and Controller reachable from the new node (2) | |
| Filebeat installed, same version (3.1) | |
| `test config` OK / `test output` OK / keystore keys present (3.4) | |
| Filebeat active, logs visible in Kibana with this host name (3.4–3.5) | |
| Machine Agent installed, same version (4.1) | |
| `unique-host-id` / `node-name` set for this node, or left empty (4.3) | |
| Agent registered, host visible in Controller, NGINX metrics present (4.4–4.5) | |
