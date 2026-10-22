# NGINX Labs: SOPs

Standard operating procedures for scaling out an air-gapped NGINX Plus (R36) deployment on RHEL 9, validated in a lab.

| SOP | Purpose |
|---|---|
| [SOP-01](SOP-01-NGINX-Plus-Offline-Bundle-Build.md) | Build an offline RPM bundle on a RHEL 9 server with internet access |
| [SOP-02](SOP-02-NGINX-Plus-Node-Install-From-Bundle.md) | Install a new node from the bundle and sync its config with nginx-sync |
| [SOP-03](SOP-03-NIM-Usage-Reporting.md) | NGINX Instance Manager (2.21) licensing and usage reporting |
| [SOP-04](SOP-04-PROD-Staged-Install-From-Node1-Bundle.md) | 🏭 Prod only: install new nodes from the RPMs already on Node 1, before the licence arrives (NGINX left stopped) |
| [SOP-05](SOP-05-PROD-Pre-Licence-Readiness.md) | 🏭 Prod only: everything that can be done on the new nodes before the licence arrives (SSH, OS settings, network checks) |

Order onsite while the licence is pending: SOP-04 → SOP-05 → (licence arrives) SOP-02 from step 3.5 → SOP-03.

Steps are marked 🧪 **TRIAL ONLY** (lab practice) or 🏭 **PROD ONLY**. Placeholders such as `<NODE1_IP>`, `<NIM_IP>` and `CLIENT` must be replaced with real values. Never commit licence files, keys, tokens or passwords.
