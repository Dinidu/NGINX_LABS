# SOP-01: Build the NGINX Plus Offline Bundle

| | |
|---|---|
| Project | CLIENT NGINX Plus: additional nodes (2 → 6) |
| Phase | 1 of 3: build the offline bundle (Phase 2: install nodes from the bundle. Phase 3: nginx-sync and F5) |
| Runs on | A RHEL 9 **build server with internet access**. It is **not** one of the NGINX nodes |
| Output | `nginx-plus-<build>-el9-offline.tar.gz` plus its `.sha256` file, ready to carry to the air-gapped NGINX nodes |
| Time | About 30 minutes |
| Validated | Azure lab, 16 Oct 2026: R36-p8 (`1:36-9.el9.ngx`) + `nginx-sync 1.1-1`, RHEL 9.8. 98 RPMs, 42 MB. Clean-room install test passed |

---

## How to read this SOP

| Marker | Meaning |
|---|---|
| *(no marker)* | Do this step in **both** trial and production |
| 🧪 **TRIAL ONLY** | Practice runs in the Azure lab (`<LAB_RESOURCE_GROUP>`). **Skip in production** |
| 🏭 **PROD ONLY** | Production build for CLIENT. **Skip in trial runs** |

Run every command as **root** (`sudo -i`). Paste commands **one line at a time**. Long lines can wrap when copied and lose their second half.

---

## 0. Before you start

### 0.1 Inputs

| Input | 🧪 Trial | 🏭 Prod |
|---|---|---|
| Build server | `lab-n1` (Azure, RHEL 9.8) or any RHEL 9 lab VM | RHEL 9 server with internet access, provided by CLIENT, separate from the NGINX nodes |
| `nginx-repo.crt` / `nginx-repo.key` | NGINX trial from MyF5 (`licenses/nginx-one-eval*.crt/.key`) | From **CLIENT's own subscription** (the one covering the new nodes), supplied by CLIENT / F5 / the reseller |
| NGINX Plus build to bundle | `1:36-9.el9.ngx` | **Exactly the build running on the existing CLIENT nodes.** See 0.2 |
| RHEL release of the build server | Any RHEL 9 | **Same minor release as the NGINX nodes** (e.g. 9.4 → 9.4). See 0.2 |
| `license.jwt` | Not used in this SOP | Not used in this SOP. **The licence is never put in the bundle.** It is copied to each node separately in Phase 2 |

### 0.2 Facts to collect from the existing NGINX node (🏭 PROD ONLY)

Run on the **existing CLIENT Node 1** (<NODE1_IP>) and write the answers in the record sheet (section 9):

```bash
rpm -q nginx-plus
rpm -qa | grep -iE 'nginx|nms'
cat /etc/redhat-release
uname -m
```

- `nginx-plus-36-9.el9.ngx.x86_64` means the build to bundle is `1:36-9.el9.ngx`. The `1:` (the epoch) does not appear in the `rpm -q` output, but dnf needs it.
- Every other NGINX package listed (modules, `nginx-agent`) must also be bundled, or noted as coming from somewhere else. `nginx-agent` comes from NIM, not from this bundle.
- If the build server's minor release differs from the nodes' (`/etc/redhat-release`), **stop and raise it**. Dependencies from a newer minor release can force core library upgrades on the nodes.

### 0.3 Build-server prerequisites

| Requirement | 🧪 Trial | 🏭 Prod |
|---|---|---|
| Red Hat package repositories | Provided by Azure RHUI. Ignore the "not registered with an entitlement server" message | Server **registered** (`subscription-manager status` shows *Current*/*Disabled* SCA) or attached to Satellite, with BaseOS and AppStream enabled |
| Outbound HTTPS to `pkgs.nginx.com`, `cs.nginx.com`, `nginx.org` | Open | Ask the CLIENT network team to allow it, or provide a proxy (see 1.4) |
| Free space under `/root` | 1 GB | 1 GB |

---

## 1. Log in and check the build server

**1.1 Log in.**

🧪 **TRIAL ONLY.** Start the lab VMs and log in. Run this from your Mac:

```bash
az vm start --ids $(az vm list -g <LAB_RESOURCE_GROUP> --query "[].id" -o tsv)
ssh -i ~/.ssh/id_lab azureuser@<LAB_N1_PUBLIC_IP>
```

If SSH times out, your public IP has changed. Update the NSG rule `allow-ssh-myip` in `nsg-nginx-lab`.

🏭 **PROD ONLY.** SSH to the CLIENT build server with the account CLIENT provided.

**1.2 Become root:**

```bash
sudo -i
```

**1.3 Check the server:**

```bash
hostname
cat /etc/redhat-release
uname -m
df -h /root
dnf repolist
```

✅ Expect: RHEL 9.x, `x86_64`, at least 1 GB free, and the BaseOS and AppStream repositories listed.

**1.4 Test internet access to F5:**

```bash
curl -sI https://cs.nginx.com/static/files/plus-9.repo | head -1
curl -sI https://pkgs.nginx.com/ | head -1
```

✅ Expect: an HTTP status line from each, any code. A timeout means there is no route.

🏭 **PROD ONLY.** If CLIENT uses a proxy, set it for dnf and curl before continuing:

```bash
echo "proxy=http://<PROXY_HOST>:<PORT>" >> /etc/dnf/dnf.conf
export https_proxy=http://<PROXY_HOST>:<PORT>
```

---

## 2. Prepare the folders and the repository credentials

**2.1 Create the folders:**

```bash
mkdir -p /etc/ssl/nginx
mkdir -p /root/nginx-bundle/rpms
chmod 700 /etc/ssl/nginx
```

**2.2 Copy the certificate and key to the build server.**

🧪 **TRIAL ONLY.** Run from your Mac:

```bash
cd ~/<PROJECT_DIR>/licenses
scp -i ~/.ssh/id_lab "nginx-one-eval (5).crt" azureuser@<LAB_N1_PUBLIC_IP>:~/nginx-repo.crt
scp -i ~/.ssh/id_lab "nginx-one-eval (6).key" azureuser@<LAB_N1_PUBLIC_IP>:~/nginx-repo.key
```

🏭 **PROD ONLY.** Transfer CLIENT's `nginx-repo.crt` and `nginx-repo.key` using the method CLIENT approves, into the login user's home directory.

**2.3 Move them into place and lock them down.** Replace `<LOGIN_USER>` with `azureuser` in trial, or your CLIENT account in production:

```bash
mv /home/<LOGIN_USER>/nginx-repo.crt /home/<LOGIN_USER>/nginx-repo.key /etc/ssl/nginx/
chown root:root /etc/ssl/nginx/nginx-repo.*
chmod 644 /etc/ssl/nginx/nginx-repo.crt
chmod 600 /etc/ssl/nginx/nginx-repo.key
restorecon -Rv /etc/ssl/nginx
```

**2.4 Verify the certificate and key:**

```bash
openssl x509 -in /etc/ssl/nginx/nginx-repo.crt -noout -subject -enddate
openssl x509 -in /etc/ssl/nginx/nginx-repo.crt -noout -pubkey | sha256sum
openssl pkey -in /etc/ssl/nginx/nginx-repo.key -pubout | sha256sum
```

✅ Expect: the two `sha256sum` values are **identical**, which means the key matches the certificate. `notAfter` is in the future.
- 🧪 Trial subject: `CN = F5-TRL-...`
- 🏭 Prod subject: CLIENT's subscription. Record it in section 9.

---

## 3. Add the F5 repository

**3.1 Download the repository file:**

```bash
curl -fsSo /etc/yum.repos.d/plus-9.repo https://cs.nginx.com/static/files/plus-9.repo
cat /etc/yum.repos.d/plus-9.repo
```

✅ Expect: `sslclientcert=/etc/ssl/nginx/nginx-repo.crt` and `sslclientkey=/etc/ssl/nginx/nginx-repo.key`. The URL says `centos/9`, which is normal for RHEL 9.

**3.2 Save F5's package signing key into the bundle.** The nodes need it to verify packages offline:

```bash
curl -fsSo /root/nginx-bundle/nginx_signing.key https://nginx.org/keys/nginx_signing.key
rpm --import /root/nginx-bundle/nginx_signing.key
```

**3.3 Test access to the repository:**

```bash
dnf makecache --repo nginx-plus
```

✅ Expect: `nginx-plus repo ... Metadata cache created`. A `400` or SSL error means the certificate or key is wrong, unreadable or expired. Go back to step 2.4.

**3.4 List the available builds.** Versions carry an epoch (`1:`), so search for `:36-`, not ` 36-`:

```bash
dnf --showduplicates list nginx-plus --repo nginx-plus 2>/dev/null | grep ':36-'
```

**3.5 Set the build to bundle:**

🧪 **TRIAL ONLY:**

```bash
NGX_VER=1:36-9.el9.ngx
```

🏭 **PROD ONLY.** Use the value from 0.2, with the `1:` epoch added:

```bash
NGX_VER=1:<BUILD_FROM_EXISTING_NODE>
```

⚠️ The repository also contains **R37**. Never install or download plain `nginx-plus` without a version.

---

## 4. Download the packages and all their dependencies

**4.1 Download NGINX Plus, nginx-sync and the complete dependency tree:**

```bash
dnf download --resolve --alldeps --destdir /root/nginx-bundle/rpms nginx-plus-$NGX_VER.x86_64 nginx-sync initscripts-service
```

`initscripts-service` provides `/sbin/service`. nginx-sync uses it to reload the peers (`service nginx reload`), but the nginx-sync package does not depend on it. On a minimal RHEL install without it, nginx-sync **silently skips the reload**. Including it makes sure every node has it.

`--alldeps` downloads every dependency, including those already installed on the build server. The bundle therefore does not depend on what is installed on the target nodes.

If `dnf download` is not found, run `dnf install -y dnf-plugins-core` and repeat.

🏭 **PROD ONLY.** Add any extra NGINX packages found in 0.2 (for example dynamic modules) to the same command, each with its exact version.

**4.2 Remove any other nginx-plus version.** ⚠️ This is mandatory. `nginx-sync` requires "any nginx-plus", so step 4.1 also pulls in the **latest R37**:

```bash
ls /root/nginx-bundle/rpms/nginx-plus-*
find /root/nginx-bundle/rpms -name 'nginx-plus-[0-9]*.rpm' ! -name "nginx-plus-${NGX_VER#*:}.x86_64.rpm" -print -delete
ls /root/nginx-bundle/rpms/nginx-plus-*
```

✅ Expect: the final `ls` shows **exactly one** file, `nginx-plus-36-9.el9.ngx.x86_64.rpm` (or your production build).

**4.3 Count the packages:**

```bash
ls /root/nginx-bundle/rpms/*.rpm | wc -l
du -sh /root/nginx-bundle/rpms
```

✅ Trial reference: 98 RPMs, about 45 MB. Production numbers may differ slightly. Record them.

---

## 5. Turn the folder into a local repository and validate it

**5.1 Build the repository index:**

```bash
dnf install -y createrepo_c
createrepo_c /root/nginx-bundle/rpms
ls /root/nginx-bundle/rpms/repodata/repomd.xml
```

**5.2 Verify every package signature.** Red Hat packages are checked with Red Hat's key, NGINX packages with F5's key:

```bash
rpm -K /root/nginx-bundle/rpms/*.rpm | grep -vc 'digests signatures OK'
```

✅ Expect: `0`. Anything else lists unsigned or tampered packages. Investigate before going further.

**5.3 Dry run with every online repository disabled:**

```bash
dnf --disablerepo='*' --repofrompath=offline,/root/nginx-bundle/rpms --enablerepo=offline --setopt=offline.gpgcheck=0 install --assumeno nginx-plus-$NGX_VER nginx-sync initscripts-service
```

✅ Expect: `nginx-plus` shows **your exact build** (not 37.x) and `nginx-sync 1.1-1.el9.ngx`, then `Operation aborted.`

**5.4 Clean-room test.** This proves the bundle is complete by resolving it into an **empty** system root using only the bundle:

```bash
mkdir -p /tmp/cleanroot
dnf --installroot=/tmp/cleanroot --releasever=9 --setopt=reposdir=/dev/null --setopt=cachedir=/tmp/cleanroot-cache --repofrompath=offline,/root/nginx-bundle/rpms --setopt=offline.gpgcheck=0 install --assumeno nginx-plus-$NGX_VER nginx-sync initscripts-service
rm -rf /tmp/cleanroot /tmp/cleanroot-cache
```

✅ Expect: `Install  <N> Packages` (trial: 98) and `Operation aborted.`, with **no** `nothing provides` or `No match` errors. Any such error names a missing dependency. Download it into `rpms/` (`dnf download --destdir /root/nginx-bundle/rpms <package>`), then repeat 5.1–5.4.

---

## 6. Package the bundle

**6.1 Write the checksums, the version list and the build information:**

```bash
cd /root/nginx-bundle
sha256sum rpms/*.rpm > SHA256SUMS
rpm -qp --qf '%{NAME}-%{EPOCH}:%{VERSION}-%{RELEASE}\n' rpms/nginx-*.rpm > VERSIONS.txt
printf "built_on=%s\nbuilt_at=%s\nbuild_os=%s\nngx_ver=%s\nrpm_count=%s\n" "$(hostname)" "$(date -u +%FT%TZ)" "$(cat /etc/redhat-release)" "$NGX_VER" "$(ls rpms/*.rpm | wc -l)" > BUILD-INFO.txt
cat VERSIONS.txt BUILD-INFO.txt
```

✅ Expect: `VERSIONS.txt` has **exactly two lines** (more if production modules were added):

```
nginx-plus-1:36-9.el9.ngx
nginx-sync-(none):1.1-1.el9.ngx
```

**6.2 Create the archive and its checksum:**

```bash
cd /root
BUNDLE=nginx-plus-${NGX_VER#*:}-offline
BUNDLE=${BUNDLE/.el9.ngx/-el9}
rm -f $BUNDLE.tar.gz $BUNDLE.tar.gz.sha256
tar czf $BUNDLE.tar.gz nginx-bundle
sha256sum $BUNDLE.tar.gz | tee $BUNDLE.tar.gz.sha256
ls -lh $BUNDLE.tar.gz
tar tzf $BUNDLE.tar.gz | grep -c '\.rpm$'
```

✅ Expect: `nginx-plus-36-9-el9-offline.tar.gz` (trial: 42 MB) and a matching `.sha256` file. The RPM count matches 4.3. **Record the SHA-256 in section 9.**

The bundle contains:

```
nginx-bundle/
├── rpms/              all RPMs + repodata/ (a ready-made local dnf repository)
├── nginx_signing.key  F5 package signing key
├── SHA256SUMS         per-RPM checksums
├── VERSIONS.txt       NGINX packages and exact versions
└── BUILD-INFO.txt     build server, date, OS, version, RPM count
```

It never contains `license.jwt`, `nginx-repo.crt` or `nginx-repo.key`.

---

## 7. Transfer the bundle to the NGINX nodes

**7.1 Move the archive.**

🧪 **TRIAL ONLY.** The bundle is already on `lab-n1`. Phase 2 copies it to `lab-n2` with `scp` over the private network.

🏭 **PROD ONLY:**
1. Copy `$BUNDLE.tar.gz` **and** `$BUNDLE.tar.gz.sha256` to the CLIENT-approved transfer medium (client USB, or a jump host if the client allows one).
2. Put it through CLIENT's malware scanning or transfer process if required.
3. Copy it to the first NGINX node (for example to `/root/`).

**7.2 Verify the archive on every node that receives it, before unpacking:**

```bash
cd /root
sha256sum -c nginx-plus-*-offline.tar.gz.sha256
```

✅ Expect: `nginx-plus-...-offline.tar.gz: OK`. **Do not install if it fails.** Copy the file again.

---

## 8. Clean up the build server

🧪 **TRIAL ONLY.** Keep the lab as it is for Phase 2. When you finish for the day, stop the VMs from your Mac:

```bash
az vm deallocate --ids $(az vm list -g <LAB_RESOURCE_GROUP> --query "[].id" -o tsv)
```

🏭 **PROD ONLY.** Agree with CLIENT which of these apply:

```bash
dnf config-manager --set-disabled nginx-plus
shred -u /etc/ssl/nginx/nginx-repo.key
rm -f /etc/ssl/nginx/nginx-repo.crt
```

- Keep `/root/nginx-bundle` and the archive until all nodes are installed and UAT is signed off.
- Keep the repository credentials only if CLIENT wants the build server ready for future upgrades.

---

## 9. Record sheet

Fill this in during the run and keep it with the UAT evidence.

| Item | 🧪 Trial reference (16 Oct 2026) | Value for this run |
|---|---|---|
| Date / performed by | 16 Oct 2026 / Dinidu | |
| Build server hostname / RHEL release | lab-n1 / 9.8 | |
| NGINX node RHEL release (0.2) | n/a | |
| Certificate subject / expiry | F5-TRL-392935 / 13 Nov 2026 | |
| `NGX_VER` | `1:36-9.el9.ngx` (R36-p8) | |
| nginx-sync version | `1.1-1.el9.ngx` | |
| RPM count (4.3) | 98 | |
| Signature failures (5.2) | 0 | |
| Dry run correct version (5.3) | ✅ | |
| Clean-room test (5.4) | ✅ 98 packages, no errors | |
| Archive name / size | nginx-plus-36-9-el9-offline.tar.gz / 42 MB | |
| Archive SHA-256 | *(per build)* | |
| Checksum verified on target (7.2) | | |

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `grep ' 36-'` returns nothing in 3.4 | Versions have an epoch: `1:36-9…` | Use `grep ':36-'` |
| Dry run shows `nginx-plus 1:37.x` | `nginx-sync` pulled in the latest nginx-plus | Repeat 4.2, run `createrepo_c --update /root/nginx-bundle/rpms`, then rerun 5.3 |
| `dnf makecache` gives 400 / SSL error | Certificate or key wrong, mismatched, expired or unreadable | Rerun 2.4. Check the permissions, and run `restorecon -Rv /etc/ssl/nginx` |
| "This system is not registered with an entitlement server" | 🧪 Azure RHUI image | Ignore in trial. 🏭 In production, register the build server (0.3) |
| `dnf download: command not found` | Plugin not installed | `dnf install -y dnf-plugins-core` |
| `nothing provides …` in 5.4 | A dependency is missing from the bundle | Download it into `rpms/`, then rerun 5.1–5.4 |
| Phase 2 dry run on a node shows **upgrades** of glibc/openssl etc. | Bundle built on a newer RHEL minor release than the node | 🏭 Rebuild on a build server with the same minor release as the nodes |
| `copy`/`install` "missing destination operand" | A long line wrapped when pasted | Paste the command again as a single line |
