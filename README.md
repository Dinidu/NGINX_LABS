# NGINX Labs: SOPs

Standard operating procedures for scaling out an air-gapped NGINX Plus (R36) deployment on RHEL 9, validated in a lab.

| SOP | Purpose |
|---|---|
| [SOP-01](SOP-01-NGINX-Plus-Offline-Bundle-Build.md) | Build an offline RPM bundle on a RHEL 9 server with internet access |
| [SOP-02](SOP-02-NGINX-Plus-Node-Install-From-Bundle.md) | Install a new node from the bundle and sync its config with nginx-sync |
| [SOP-03](SOP-03-NIM-Usage-Reporting.md) | NGINX Instance Manager (2.21) licensing and usage reporting |

Steps are marked 🧪 **TRIAL ONLY** (lab practice) or 🏭 **PROD ONLY**. Placeholders such as `<NODE1_IP>`, `<NIM_IP>` and `CLIENT` must be replaced with real values. Never commit licence files, keys, tokens or passwords.
