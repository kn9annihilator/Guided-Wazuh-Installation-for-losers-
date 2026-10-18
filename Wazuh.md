# Wazuh SIEM on VirtualBox + Windows Agent — Complete Installation and Troubleshooting Runbook

**Prepared:** 19 October 2026  
**Wazuh version observed in the lab:** 4.14.9-1  
**Original lab:** Kali Linux VM (`trooper`) in VirtualBox; Windows host with a Wazuh Agent  
**Purpose:** A reproducible start-to-finish guide, including the mistakes encountered in the original installation, the verified fixes, health checks, and the Windows-agent networking plan.

> **Scope and honesty note:** This runbook consolidates the retrievable troubleshooting history and the commands/results preserved from the conversations. It includes all confirmed technical issues available to me, but it cannot guarantee a verbatim transcript of every message from every past chat. Any step marked **not yet verified** was discussed/planned but the available history does not prove it was completed.

---

## 1. The important lesson before starting

The installation failed for several different reasons at different stages. They should not be treated as one generic “Wazuh is broken” problem:

1. **Platform mismatch:** The lab used Kali Linux 2026/rolling. Wazuh's central-component installation guide lists supported operating systems such as Ubuntu LTS, RHEL, Amazon Linux, and CentOS Stream; Kali is not on the listed supported OS set. Package installation may work, but troubleshooting is less predictable.
2. **Network mode:** VirtualBox Bridged networking downloaded a large Indexer package at roughly **78 KiB/s**, while the Windows host downloaded at roughly **46 MB/s**. RX errors on Kali's bridged interface rose from **19,633 to 20,394**. Switching the VM to NAT produced about **43.2 MB/s** in the test. This was a VirtualBox/network-path issue, not evidence that the Wazuh package itself was corrupt.
3. **IP address changed:** Bridged mode used `192.168.0.192`; NAT mode changed Kali to `10.0.2.15`, with gateway `10.0.2.2`. Any config or certificates generated using the old IP were now potentially stale.
4. **Installer/certificate mismatch:** The assisted installer depends on the node names in `config.yml` matching the names used when installing each component and on the generated archive being present. An earlier attempt failed because the archive did not contain the expected `node-1` certificate. Certificate files were regenerated later, but the workflow was subsequently abandoned for a manual package/recovery path.
5. **Partial manual installation:** Indexer and Manager were installed/active, but the Dashboard package was initially absent. A separate Dashboard installation/configuration path then exposed missing certificate files and a bad Indexer URL.
6. **Indexer security initialization:** A request to Indexer returned `OpenSearch Security not initialized` during the broken/partial stage. Later, a client-certificate request returned a **green** cluster with one node and all primary shards assigned, confirming that the Indexer security/cluster stage was eventually healthy.
7. **Dashboard configuration:** The Dashboard first failed because `/etc/wazuh-dashboard/certs/dashboard-key.pem` did not exist. After the certificate files were restored, it failed to connect to `https://localhost:9200` because `localhost` resolved to `::1` and the Indexer was listening on `10.0.2.15:9200`. Changing the Dashboard's Indexer URL to `https://10.0.2.15:9200` fixed the observed connection failure. `curl` then returned `302 Found` with `Location: /app/login?`, which is a normal redirect to the login page.
8. **Windows host ↔ NAT guest:** A Windows host generally cannot connect directly to a NAT guest's `10.0.2.15` address. The planned solution is VirtualBox NAT port forwarding for TCP 1514 and 1515. The available history does **not** confirm that these forwarding rules were actually added or that the Windows Agent successfully connected after them.

**Best clean-install recommendation:** Use a supported Ubuntu LTS VM and the official Wazuh all-in-one installation assistant. If you intentionally continue on Kali, treat it as an unsupported lab variant and expect to troubleshoot platform-specific issues. Do not mix the official assisted installer with manual package installation unless you deliberately choose and document that recovery path.

---

## 2. Architecture and ports

A single-node Wazuh deployment consists of:

| Component | Purpose | Typical port |
|---|---|---:|
| Wazuh Agent | Collects endpoint telemetry | Connects to Manager on TCP 1514 |
| Wazuh Server / Manager | Receives agent events, decodes/rules, generates alerts; includes Filebeat in the central deployment | TCP 1514 agent traffic; TCP 1515 enrollment; TCP 55000 API |
| Wazuh Indexer | Stores/searches alerts and indexed data | TCP 9200 HTTPS API; internal cluster transport commonly TCP 9300 |
| Wazuh Dashboard | Web UI | TCP 443 HTTPS |

The exact listening interfaces depend on configuration. Confirm them on the actual VM rather than assuming a port is exposed.

### Recommended lab network design

**Option A — Bridged networking:** Kali gets an address on the same LAN as Windows. Windows can usually connect directly to the Kali LAN address, provided the Windows/Kali firewalls and network allow it. However, this user's bridged path had severe download performance and rising RX errors.

**Option B — NAT + port forwarding (the mode used during recovery):**

- Kali guest IP observed: `10.0.2.15`
- NAT gateway observed: `10.0.2.2`
- Windows connects to host loopback `127.0.0.1`; VirtualBox forwards selected ports into the guest.
- Forward **TCP host `127.0.0.1:1514` → guest `10.0.2.15:1514`**.
- Forward **TCP host `127.0.0.1:1515` → guest `10.0.2.15:1515`**.
- Dashboard forwarding is optional. If needed, prefer host port `8443` → guest `443` to avoid conflicts with host HTTPS services. It was not needed for this user's stated goal.

> If the guest IP changes after a reboot, update the forwarding target. Better still, verify the current IP every time before debugging agent connectivity.

---

## 3. Before installation: choose the OS and allocate resources

### 3.1 Use a supported OS for a predictable installation

Wazuh's current quickstart lists Ubuntu 16.04/18.04/20.04/22.04/24.04, Amazon Linux 2/2023, CentOS Stream 10, and RHEL versions among its supported/recommended systems. For a new lab, choose **Ubuntu Server 24.04 LTS x86_64** unless you have a specific reason to use another supported release.

The troubleshooting VM was **Kali Linux 26.04.1 LTS / Resolute Raccoon** according to `/etc/os-release`, with kernel `7.0.0-34-generic` at one point. This is not the recommended platform for the clean recipe below.

### 3.2 VM resources

Wazuh's quickstart guidance for a small single-node deployment recommends roughly **4 vCPU, 8 GiB RAM, and 50 GB storage** for 1–25 agents and 90 days of indexed alerts. For a small temporary lab you may be able to start lower, but slow indexing and memory pressure become more likely. The recovered VM had a 60 GB disk in a related lab context, and the user observed slow performance in other VMs; allocate resources deliberately.

Before installing:

- Enable hardware virtualization in the host firmware/VirtualBox environment.
- Give the VM enough RAM/CPU without starving Windows.
- Use a dynamically allocated or fixed virtual disk large enough for the OS, packages, indexes, and logs; storage grows with retention.
- Take a **VirtualBox snapshot** after a clean OS install and before central-component installation. The original VM had no snapshot.
- Decide the network mode before generating certificates/config. Do not change IP modes halfway through the installer unless you update the configuration and regenerate certificates consistently.

### 3.3 Record the baseline

Run in the Linux guest:

```bash
cat /etc/os-release
uname -a
uname -m
ip -br address
ip route
free -h
df -h
```

Expected architecture for the downloaded packages in this lab was `amd64` / `x86_64`. Record the guest IP and gateway.

---

## 4. Recommended clean installation: official all-in-one assistant

This is the clean path for a supported Ubuntu LTS guest. It installs the Indexer, Server/Manager, and Dashboard on one machine. The official quickstart for Wazuh 4.14 uses the `4.14` installer URL.

### 4.1 Update Ubuntu and install prerequisites

```bash
sudo apt update
sudo apt full-upgrade -y
sudo apt install -y curl ca-certificates gnupg apt-transport-https
sudo reboot
```

After reboot, verify networking and time again:

```bash
ip -br address
ip route
timedatectl
```

### 4.2 Download and run the official installer

Use a dedicated working directory so the installer archive and passwords are easy to find:

```bash
mkdir -p ~/wazuh-install
cd ~/wazuh-install
curl -fL --retry 3 --retry-delay 2 -o wazuh-install.sh https://packages.wazuh.com/4.14/wazuh-install.sh
chmod 700 wazuh-install.sh
sha256sum wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```

- `-a` means install all central components on this single node.
- Do not separately run the per-component installer options if the all-in-one run succeeds.
- Let the command finish. Do not close the terminal or interrupt package installation while it is actively working.
- Save the final summary, Dashboard URL, username, and password in a password manager or other secure place.

The official assistant creates `wazuh-install-files.tar` in the working directory. It includes generated configuration/certificates and `wazuh-passwords.txt`.

### 4.3 Retrieve the generated credentials if needed

From the directory containing the archive:

```bash
sudo tar -O -xvf wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt
```

Protect this archive and password output: it contains sensitive credentials and private-key material. Do not upload it to a public repository or share it in screenshots.

### 4.4 Check services immediately

```bash
sudo systemctl --no-pager --full status wazuh-indexer
sudo systemctl --no-pager --full status wazuh-manager
sudo systemctl --no-pager --full status wazuh-dashboard
sudo systemctl --no-pager --full status filebeat
```

Quick active-state check:

```bash
systemctl is-active wazuh-indexer wazuh-manager wazuh-dashboard filebeat
```

All should normally report `active`. If not, stop here and inspect the failing service's logs before continuing.

### 4.5 Check the Dashboard locally

On the guest, open:

```text
https://127.0.0.1
```

Or use the guest's actual configured IP from the installation summary. A browser certificate warning is expected with the generated self-signed CA; confirm that the URL is the expected local lab endpoint before proceeding. Use the generated `admin` password.

A command-line HTTP check can confirm that the web service responds:

```bash
curl -k -I https://127.0.0.1:443
```

A `302 Found` response with `Location: /app/login?` is a successful redirect to the login page; it is not a failure by itself.

### 4.6 Check Indexer health

Use the generated admin password when prompted:

```bash
curl -k -u admin https://127.0.0.1:9200/
curl -k -u admin https://127.0.0.1:9200/_cluster/health?pretty
curl -k -u admin https://127.0.0.1:9200/_cat/nodes?v
```

If the assistant's setup expects client-certificate authentication or a specific user/password, follow the exact credential/certificate method it generated rather than assuming a request without credentials must succeed. A response such as `Unauthorized` is an authentication result, not necessarily a networking failure.

For the original Kali recovery, this command worked and returned a green one-node cluster:

```bash
curl -k \
  --cert /etc/wazuh-indexer/certs/admin.pem \
  --key /etc/wazuh-indexer/certs/admin-key.pem \
  'https://10.0.2.15:9200/_cluster/health?pretty'
```

Confirmed output values were `status: green`, `number_of_nodes: 1`, `number_of_data_nodes: 1`, `active_primary_shards: 3`, `active_shards: 3`, `unassigned_shards: 0`, and `active_shards_percent_as_number: 100.0`.

---

## 5. Alternative: the manual assisted installer workflow (only if not using `-a`)

Do not combine this section with the all-in-one `-a` installation. It is the official multi-stage workflow and was the intended approach in an earlier attempt, but the user's IP/name/archive mismatch made it fail.

### 5.1 Download the matching installer and template

```bash
mkdir -p ~/wazuh-install
cd ~/wazuh-install
curl -fL -o wazuh-install.sh https://packages.wazuh.com/4.14/wazuh-install.sh
curl -fL -o config.yml https://packages.wazuh.com/4.14/config.yml
```

Edit `config.yml` so every component's node name and IP are correct **for the final network design**. For one host, the indexer, server, and dashboard entries can refer to the same guest IP while keeping their distinct node names, for example `node-1`, `wazuh-1`, and `dashboard`. Preserve the exact YAML indentation and list structure from the downloaded template.

Before generating files, inspect the actual file; do not accidentally create an empty `config.yml` or use `config.yaml` by mistake:

```bash
pwd
ls -l wazuh-install.sh config.yml
sed -n '1,200p' config.yml
```

Generate the configuration/certificates/password archive:

```bash
sudo bash ./wazuh-install.sh --generate-config-files
ls -lh wazuh-install-files.tar
```

The archive must be present in the working directory used for subsequent installer commands. Keep the node names identical across `config.yml`, installer arguments, and certificate filenames.

### 5.2 Install each central component in the documented order

On the indexer node:

```bash
sudo bash ./wazuh-install.sh --wazuh-indexer node-1
sudo bash ./wazuh-install.sh --start-cluster
```

On the server node (or same VM in an all-in-one-by-stages setup):

```bash
sudo bash ./wazuh-install.sh --wazuh-server wazuh-1
```

On the dashboard node (or same VM):

```bash
sudo bash ./wazuh-install.sh --wazuh-dashboard dashboard
```

Follow the exact sequence in the Wazuh 4.14 documentation for your chosen deployment type; the cluster initialization step is only run once. Verify service status and health after each major stage. If you use separate machines, securely copy the generated archive to each node; do not regenerate different certificates independently on each component.

### 5.3 Why the earlier assisted attempt failed

The earlier config described both `node-1` and `wazuh-1` at `192.168.0.192`. After the VM changed from Bridged to NAT, the guest address became `10.0.2.15`. An installer error reported that the archive did not contain a certificate for `node-1`. The archive/config naming and generation had become inconsistent. The correct fix is to verify the config and archive together and regenerate the configuration archive as a coherent set—not to manually rename random certificate files or continue with stale IPs.

---

## 6. Networking: troubleshoot performance before blaming the package

This was a major issue in the original session.

### 6.1 Measure the path with a disposable download test

The original command was:

```bash
curl -L -o /dev/null \
  -w '\nSpeed: %{speed_download} bytes/sec\n' \
  --max-time 30 \
  https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-indexer/wazuh-indexer_4.14.9-1_amd64.deb
```

Observed before changing networking:

- Download reached about `353M` of `875,498,912` bytes and grew very slowly.
- Kali's rate was about **78 KiB/s** while the Windows host could download at about **46 MB/s**.
- `ip -s link show eth0` showed RX errors increasing from `19,633` to `20,394` on Bridged mode.

After switching VirtualBox Adapter 1 from **Bridged Adapter** to **NAT**, rebooting, and rerunning the test, the full file (`834.9M`) downloaded in about 20 seconds at `43,213,413 bytes/sec` (~43.2 MB/s).

### 6.2 Safe procedure when a package download appears stuck

1. Check `ip route`, `ip -s link show eth0`, and a small connectivity test.
2. Compare the guest download against the host download.
3. If bridged networking is showing interface errors or a large performance mismatch, change one thing at a time: shut down the VM, switch Adapter 1 to NAT, boot, and retest.
4. Check the guest IP after switching; NAT usually changes it to `10.0.2.15` in a default VirtualBox NAT setup.
5. Do not delete an APT partial package immediately; APT may be able to resume it. Inspect the process and package cache first.
6. If the VM is using NAT, plan port forwarding for Windows-agent traffic before attempting to connect the host agent.

Useful diagnostics:

```bash
ip -br address
ip route
ip -s link show eth0
getent hosts packages.wazuh.com
curl -I --max-time 15 https://packages.wazuh.com/
```

---

## 7. If a clean installation is genuinely needed: uninstall and verify

**Warning:** This removes Wazuh central components and can delete stored alert/index data and configuration. Do not run this against a working installation unless you deliberately want to erase it. Preserve anything you need first.

The official installation assistant supports:

```bash
cd ~/wazuh-install
sudo bash ./wazuh-install.sh --uninstall
```

For a manual package-based installation, Wazuh's component uninstall documentation uses package purge and removal of component directories. Review the current official uninstall page first. On Debian-based systems, the component packages/directories may include:

```bash
sudo apt-get remove --purge wazuh-dashboard wazuh-manager wazuh-indexer filebeat -y
```

Do **not** blindly run `rm -rf` commands copied from a guide without verifying the target paths and confirming you want to erase indexed data/configuration. Only remove directories that correspond to components you intend to remove, and only after a backup if needed.

In the original cleanup attempt, these checks showed no installed Wazuh/Filebeat packages or standard component directories, but these files still existed:

- `~/wazuh-install.sh`
- `~/wazuh-install-files.tar`
- `~/wazuh-install-files.tar.bak`
- empty/root-owned `~/config.yml`
- `/etc/apt/sources.list.d/wazuh.list`

The targeted cleanup command used then was:

```bash
sudo rm -f ~/wazuh-install.sh \
  ~/wazuh-install-files.tar \
  ~/wazuh-install-files.tar.bak \
  ~/config.yml \
  /etc/apt/sources.list.d/wazuh.list
sudo apt update
```

Verification commands:

```bash
dpkg -l | grep -Ei 'wazuh|filebeat'
find /etc /var/lib /var/ossec /usr/share -maxdepth 2 -iname '*wazuh*' 2>/dev/null
ls -la ~ | grep -Ei 'wazuh|config.yml'
```

That targeted cleanup only removed the confirmed items; it was not proof that every possible Wazuh artifact had been deleted. Prefer the official uninstaller on a supported OS for a genuinely clean start.

---

## 8. Dashboard troubleshooting — confirmed failures and fixes from this lab

### 8.1 Dashboard package missing

After Indexer and Manager were installed, the commands showed the Indexer and Manager active, but the Dashboard service was missing. `apt-cache policy` showed the Dashboard had no installed version but candidate `4.14.9-1`.

The repository had to be restored before APT could find the package. The following repository/key setup was used in the Kali recovery:

```bash
sudo apt-get install -y gnupg apt-transport-https curl
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | \
  gpg --no-default-keyring \
      --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import
sudo chmod 644 /usr/share/keyrings/wazuh.gpg
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | \
  sudo tee /etc/apt/sources.list.d/wazuh.list
sudo apt update
apt-cache policy wazuh-indexer wazuh-manager wazuh-dashboard
```

Then Dashboard was installed at the same observed version:

```bash
sudo apt-get install -y wazuh-dashboard=4.14.9-1
```

**Important:** This was a recovery procedure on Kali, not the recommended clean installation method. On a supported OS, the all-in-one assistant is less error-prone.

### 8.2 Dashboard certificate key missing (`ENOENT`)

The Dashboard log showed:

```text
ENOENT: no such file or directory, open '/etc/wazuh-dashboard/certs/dashboard-key.pem'
```

The directory `/etc/wazuh-dashboard/certs/` did not exist. The archive `/home/jugtenpik45/wazuh-install-files.tar` contained:

```text
wazuh-install-files/root-ca.pem
wazuh-install-files/root-ca.key
wazuh-install-files/dashboard.pem
wazuh-install-files/dashboard-key.pem
wazuh-install-files/admin.pem
wazuh-install-files/admin-key.pem
```

The confirmed recovery commands were:

```bash
sudo install -d -o wazuh-dashboard -g wazuh-dashboard -m 500 /etc/wazuh-dashboard/certs
sudo tar -xOf /home/jugtenpik45/wazuh-install-files.tar \
  wazuh-install-files/dashboard.pem > /etc/wazuh-dashboard/certs/dashboard.pem
sudo tar -xOf /home/jugtenpik45/wazuh-install-files.tar \
  wazuh-install-files/dashboard-key.pem > /etc/wazuh-dashboard/certs/dashboard-key.pem
sudo tar -xOf /home/jugtenpik45/wazuh-install-files.tar \
  wazuh-install-files/root-ca.pem > /etc/wazuh-dashboard/certs/root-ca.pem
sudo chown -R wazuh-dashboard:wazuh-dashboard /etc/wazuh-dashboard/certs
sudo chmod 500 /etc/wazuh-dashboard/certs
sudo chmod 400 /etc/wazuh-dashboard/certs/*
```

This was a repair of the existing installation. In a clean official assisted installation, the installer should deploy the certificates itself; if it does not, stop and inspect the archive/node names instead of fabricating certificate files.

### 8.3 Dashboard could not connect to Indexer (`ECONNREFUSED ::1:9200`)

The effective configuration had:

```yaml
server.host: 0.0.0.0
server.port: 443
opensearch.hosts: https://localhost:9200
opensearch.ssl.verificationMode: certificate
opensearch.requestHeadersAllowlist: ["securitytenant","Authorization"]
opensearch_security.multitenancy.enabled: false
opensearch_security.readonly_mode.roles: ["kibana_read_only"]
server.ssl.enabled: true
server.ssl.key: "/etc/wazuh-dashboard/certs/dashboard-key.pem"
server.ssl.certificate: "/etc/wazuh-dashboard/certs/dashboard.pem"
opensearch.ssl.certificateAuthorities: ["/etc/wazuh-dashboard/certs/root-ca.pem"]
uiSettings.overrides.defaultRoute: /app/wz-home
opensearch_security.cookie.ttl: 900000
opensearch_security.session.ttl: 900000
opensearch_security.session.keepalive: true
```

Indexer was listening at `10.0.2.15:9200`, and `localhost` resolved to IPv6 loopback `::1` in the failing connection. The confirmed fix was to edit `/etc/wazuh-dashboard/opensearch_dashboards.yml` and change only this line:

```yaml
opensearch.hosts: https://10.0.2.15:9200
```

Then restart and verify:

```bash
sudo systemctl restart wazuh-dashboard
sudo systemctl --no-pager --full status wazuh-dashboard
curl -k -I https://127.0.0.1:443
```

The final check returned:

```text
HTTP/1.1 302 Found
location: /app/login?
osd-name: trooper
```

This was the successful outcome: Dashboard responded and redirected to the login route. A later final login from the browser was asked about but is **not confirmed in the retrievable history**.

> IP-address URLs can become stale if the VM's network mode/IP changes. On a supported clean installation, preserve the installer-generated settings unless logs prove they are wrong. Do not change `localhost` merely by guesswork; first verify what address/port the Indexer actually listens on and read the Dashboard error log.

---

## 9. Indexer troubleshooting — confirmed checks

### 9.1 Indexer log directory missing

The original command returned:

```text
ls: cannot access '/var/log/wazuh-indexer': No such file or directory
```

A service startup failure referenced a missing `/var/log/wazuh-indexer/gc.log`. The directory was created with the correct ownership/permissions, then the failed state was reset and the service started:

```bash
sudo install -d -o wazuh-indexer -g wazuh-indexer -m 0750 /var/log/wazuh-indexer
sudo systemctl reset-failed wazuh-indexer
sudo systemctl start wazuh-indexer
sudo systemctl --no-pager --full status wazuh-indexer
```

This fixed the observed startup blocker. Do not create log directories with arbitrary ownership; match the service account.

### 9.2 `OpenSearch Security not initialized`

Earlier, Indexer returned:

```text
OpenSearch Security not initialized.
```

At that time, the deployment was partial/inconsistent. The installed security configuration directory was:

```text
/etc/wazuh-indexer/opensearch-security/
```

It contained files such as `action_groups.yml`, `allowlist.yml`, `audit.yml`, `config.yml`, `internal_users.yml`, `nodes_dn.yml`, `roles_mapping.yml`, `roles.yml`, `tenants.yml`, and `whitelist.yml`. The helper scripts existed under:

```text
/usr/share/wazuh-indexer/plugins/opensearch-security/tools/
```

including `securityadmin.sh` and `wazuh-passwords-tool.sh`.

The active Indexer configuration included:

```yaml
plugins.security.ssl.http.pemcert_filepath: /etc/wazuh-indexer/certs/node-1.pem
plugins.security.ssl.http.pemkey_filepath: /etc/wazuh-indexer/certs/node-1-key.pem
plugins.security.ssl.http.pemtrustedcas_filepath: /etc/wazuh-indexer/certs/root-ca.pem
plugins.security.ssl.transport.pemcert_filepath: /etc/wazuh-indexer/certs/node-1.pem
plugins.security.ssl.transport.pemkey_filepath: /etc/wazuh-indexer/certs/node-1-key.pem
plugins.security.ssl.transport.pemtrustedcas_filepath: /etc/wazuh-indexer/certs/root-ca.pem
plugins.security.ssl.http.enabled: true
plugins.security.ssl.transport.enforce_hostname_verification: false
plugins.security.ssl.transport.resolve_hostname: false
plugins.security.authcz.admin_dn:
  - "CN=admin,OU=Wazuh,O=Wazuh,L=California,C=US"
node.name: node-1
network.host: 10.0.2.15
cluster.initial_master_nodes: node-1
plugins.security.nodes_dn:
  - CN=node-1,OU=Wazuh,O=Wazuh,L=California,C=US
```

The admin and node certificate/key public-key hashes were verified to match, and certificate subjects were checked. After recovery, the admin client-certificate health check returned a green cluster. **Do not manually run `securityadmin.sh` or change security YAML blindly** on a new installation. Prefer the official installer to initialize security; if it fails, inspect logs and use the exact Wazuh version's documented recovery procedure.

### 9.3 Useful Indexer diagnostics

```bash
sudo systemctl status wazuh-indexer --no-pager -l
sudo journalctl -u wazuh-indexer -b --no-pager -n 150
sudo ss -lntp | grep -E ':(9200|9300)\b'
sudo ls -la /etc/wazuh-indexer/certs
sudo grep -nEv '^[[:space:]]*(#|$)' /etc/wazuh-indexer/opensearch.yml
```

Check one problem at a time: service state → logs → listening socket → TLS/certificate path → authentication/security initialization.

---

## 10. Manager checks

In the recovery session, the following Manager ports were listening on `0.0.0.0`:

- TCP `1514` — agent communication
- TCP `1515` — agent enrollment
- TCP `55000` — Wazuh server API

Verify on the VM:

```bash
sudo systemctl status wazuh-manager --no-pager -l
sudo journalctl -u wazuh-manager -b --no-pager -n 150
sudo ss -lntp | grep -E ':(1514|1515|55000)\b'
```

A listening socket proves the service is listening inside the guest; it does **not** prove the Windows host can reach it through VirtualBox NAT or the firewall.

---

## 11. Windows host Agent connection (NAT setup)

**Status in the recovered history:** This was the next planned stage after the Dashboard began returning `302 Found`. The available history does not prove that NAT forwarding was configured or that the Windows Agent was successfully connected. Follow these checks without deleting an existing enrollment/key unnecessarily.

### 11.1 Add VirtualBox port forwarding

Power off the VM if VirtualBox requires it to edit the adapter. In VirtualBox:

1. Select the Kali/Ubuntu VM → **Settings** → **Network** → **Adapter 1**.
2. Confirm **Attached to: NAT**.
3. Expand **Advanced** → **Port Forwarding**.
4. Add these rules (TCP):

| Name | Protocol | Host IP | Host Port | Guest IP | Guest Port |
|---|---|---|---:|---|---:|
| Wazuh-agent | TCP | `127.0.0.1` | `1514` | `10.0.2.15` | `1514` |
| Wazuh-enrollment | TCP | `127.0.0.1` | `1515` | `10.0.2.15` | `1515` |

If Windows reports that a host port is already in use, choose another host port (for example 11514 → guest 1514 and 11515 → guest 1515) and configure the agent's destination ports accordingly. Standard same-number forwarding is simplest when available.

Boot the VM and verify the guest IP and listeners:

```bash
ip -br address
sudo ss -lntp | grep -E ':(1514|1515)\b'
```

If the guest IP is not `10.0.2.15`, update the Guest IP in the VirtualBox forwarding rules.

### 11.2 Check Windows-to-guest TCP connectivity

Open PowerShell on Windows:

```powershell
Test-NetConnection 127.0.0.1 -Port 1514
Test-NetConnection 127.0.0.1 -Port 1515
```

Look for `TcpTestSucceeded : True`. If false, troubleshoot in this order:

1. Is the Manager listening on the guest ports?
2. Are the VirtualBox forwarding rules correct and pointed at the current guest IP?
3. Are Windows security software/firewall rules interfering?
4. Is the Wazuh service healthy?

Do not attempt enrollment until basic TCP reachability is working.

### 11.3 Inspect the existing Windows Agent before changing it

Common config paths are:

```text
C:\Program Files (x86)\ossec-agent\ossec.conf
C:\Program Files\ossec-agent\ossec.conf
```

Common service name: `WazuhSvc`.

PowerShell checks:

```powershell
Get-Service WazuhSvc
Get-Content 'C:\Program Files (x86)\ossec-agent\ossec.conf'
```

Use the actual installation path if different. Inspect the existing `<client><server><address>...` value and preserve the current agent registration/key unless you have a reason to re-enroll. The user previously had a working Windows-host agent in an earlier Wazuh lab, so avoid destroying its registration without first inspecting it.

For the NAT-forwarded design, the Manager address in the agent configuration should normally be `127.0.0.1` because that is the Windows host endpoint of the VirtualBox forwarding rule. The agent's communication destination remains TCP 1514 unless a different host-side forwarded port was chosen.

### 11.4 Enrollment

If the agent is already enrolled and has a valid existing key, first try to restore connectivity using the correct Manager address and restart the service. Do not re-enroll just because the network address changed.

If installing/enrolling a new agent, use the current Wazuh Windows-agent installation documentation and configure the Manager/registration server address to the Windows-side forwarded endpoint (`127.0.0.1` for this design). Enrollment normally uses TCP 1515; API-based enrollment may use TCP 55000. Do not expose port 55000 to the host unless you specifically need API enrollment and understand the access controls.

Restart the agent service after configuration changes:

```powershell
Restart-Service WazuhSvc
Get-Service WazuhSvc
```

Check the Windows Agent log, commonly under the agent's `logs\ossec.log` path, and check the Manager logs/agent list on the server. Confirm the endpoint appears as **Active** in the Dashboard before declaring success.

### 11.5 If you want the Dashboard on Windows too (optional)

The user's stated need was agent-only connectivity, so this is optional. If needed, create a separate VirtualBox forwarding rule such as host `127.0.0.1:8443` → guest `10.0.2.15:443`, then open `https://127.0.0.1:8443` in Windows. Avoid exposing the Dashboard publicly.

---

## 12. Common errors and the first diagnostic to run

| Symptom | Meaning / first check | Next action |
|---|---|---|
| Large package download crawls in Kali but is fast on Windows | Guest network path may be faulty; check `ip -s link show eth0` | Test NAT after recording the old network/IP; rerun the same curl benchmark |
| `node-1` certificate missing from `wazuh-install-files.tar` | Installer archive and node name/config are inconsistent | Inspect `config.yml`, archive contents, and current working directory; regenerate as one consistent set |
| `OpenSearch Security not initialized` | Indexer security initialization did not complete or the deployment is inconsistent | Check install logs and official installer stages; do not just change the password or manually rerun security tools at random |
| `/var/log/wazuh-indexer` missing / `gc.log` failure | Required service log path is absent | Create the directory with `wazuh-indexer:wazuh-indexer` ownership and mode `0750`, then restart and inspect logs |
| Dashboard service does not exist | Package not installed or installation incomplete | `apt-cache policy wazuh-dashboard`; restore the repository only if appropriate; prefer all-in-one installer for clean setup |
| Dashboard `ENOENT ... dashboard-key.pem` | Config references a certificate key that is not present | Inspect generated archive and node names; restore only the correct generated cert/key pair or rerun the supported installer |
| Dashboard `ECONNREFUSED ::1:9200` | Dashboard is trying IPv6 loopback but Indexer is listening elsewhere | Check `ss -lntp`, Indexer URL in Dashboard config, and logs; in this lab `https://10.0.2.15:9200` fixed it |
| `curl -k -I https://127.0.0.1:443` returns `302 Found` / `/app/login?` | Dashboard is responding and redirecting to login | Open it in a browser and authenticate; this response alone does not prove login works |
| `Unauthorized` from Indexer | Request reached Indexer but authentication/authorization was not accepted | Use the correct credentials/client certificate; do not diagnose as a network outage by itself |
| Windows cannot reach `10.0.2.15` in NAT mode | Expected NAT isolation | Add TCP port forwarding and use the host-side address `127.0.0.1` in the agent config |
| Agent service runs but endpoint is not Active | Service running is not the same as Manager connectivity/enrollment | Test TCP 1514/1515, inspect `ossec.conf`, agent log, Manager logs and enrollment/key status |

---

## 13. Useful verification commands (copy/paste checklist)

### Network and OS

```bash
cat /etc/os-release
uname -m
ip -br address
ip route
ip -s link show eth0
```

### Package state

```bash
dpkg -l | grep -Ei 'wazuh|filebeat'
apt-cache policy wazuh-indexer wazuh-manager wazuh-dashboard
```

### Services

```bash
systemctl is-active wazuh-indexer wazuh-manager wazuh-dashboard filebeat
systemctl --no-pager --full status wazuh-indexer wazuh-manager wazuh-dashboard filebeat
```

### Logs

```bash
sudo journalctl -u wazuh-indexer -b --no-pager -n 150
sudo journalctl -u wazuh-manager -b --no-pager -n 150
sudo journalctl -u wazuh-dashboard -b --no-pager -n 150
sudo journalctl -u filebeat -b --no-pager -n 150
```

### Listening sockets

```bash
sudo ss -lntp | grep -E ':(443|9200|9300|1514|1515|55000)\b'
```

### Dashboard and Indexer

```bash
curl -k -I https://127.0.0.1:443
curl -k -u admin https://127.0.0.1:9200/_cluster/health?pretty
```

The second command prompts for a password. If your setup uses a different credential/certificate method, use the one produced by your installer.

---

## 14. Secure finishing steps

After installation and after confirming that all components work:

- Store `wazuh-passwords.txt` and `wazuh-install-files.tar` securely; these contain sensitive material.
- Take a VirtualBox snapshot of the working state and, separately, back up important Wazuh configuration/data as appropriate.
- Disable the Wazuh APT repository after installation to avoid accidental version upgrades, as recommended by Wazuh. On Debian/Ubuntu this can be done with:

  ```bash
  sudo sed -i 's/^deb /#deb /' /etc/apt/sources.list.d/wazuh.list
  sudo apt update
  ```

  Only do this after all intended components are installed and tested. Re-enable it deliberately when performing a planned upgrade.
- Keep the Dashboard and Indexer ports private to the lab. Do not forward 9200 or 9300 to the Windows host unless you have a clear need; they are not required for normal agent telemetry.
- Keep only the host-side forwarding rules needed for the lab. Do not forward these ports from the router to the public Internet.
- Document the guest IP, network mode, forwarding rules, Wazuh version, agent name, and the date of the last verified working state.

---

## 15. Original troubleshooting timeline (recovered facts)

This timeline records what happened in the prior troubleshooting sessions so the same sequence is not repeated.

1. **Earlier assisted-install attempt:** Wazuh 4.14.9 was used. A `config.yml` described `node-1` and `wazuh-1` with IP `192.168.0.192`; a separate `config.yaml` was empty. The installer reported that no certificate for `node-1` was found in `wazuh-install-files.tar`. The archive was regenerated later and included certificate files such as `root-ca.pem`, `root-ca.key`, `node-1.pem`, `node-1-key.pem`, `admin.pem`, and `admin-key.pem`. The install was retried with `bash wazuh-install.sh --wazuh-indexer node-1`.
2. **Download performance problem:** On VirtualBox Bridged networking, the Indexer package was about `875,498,912` bytes (~835 MiB). APT had a partial download around 353–355 MB. Kali measured about 78 KiB/s while Windows measured about 46 MB/s. RX errors on `eth0` increased from 19,633 to 20,394.
3. **Network mode changed:** The VM was switched from Bridged to NAT and rebooted. The guest IP changed from `192.168.0.192` to `10.0.2.15`, gateway `10.0.2.2`. The download test then measured about 43,213,413 bytes/sec and downloaded roughly 834.9 MB in about 20 seconds.
4. **Requested clean restart:** The user requested a complete fresh start and removal of Wazuh files. At that moment, checks found no Wazuh/Filebeat packages or standard component directories, but `~/wazuh-install.sh`, `~/wazuh-install-files.tar`, its `.bak`, empty/root-owned `~/config.yml`, and `/etc/apt/sources.list.d/wazuh.list` remained. These confirmed files were targeted for cleanup.
5. **Manual package route:** Wazuh repository/key setup was restored. APT showed `wazuh-indexer` and `wazuh-manager` at version `4.14.9-1`, while `wazuh-dashboard` was not installed but had candidate `4.14.9-1`. Indexer and Manager eventually became active. Their listening ports included TCP 1514, 1515, and 55000.
6. **Indexer log-path repair:** `/var/log/wazuh-indexer` was absent and the service had failed on the missing `gc.log` path. The directory was created with owner/group `wazuh-indexer` and mode `0750`, the failed state was reset, and Indexer started.
7. **Security initialization:** Indexer once returned `OpenSearch Security not initialized`. Later the admin client-certificate health request succeeded and returned a green one-node cluster with all three primary shards active and zero unassigned shards.
8. **Dashboard installation:** `wazuh-dashboard=4.14.9-1` was installed. Its config used `server.host: 0.0.0.0`, `server.port: 443`, and initially `opensearch.hosts: https://localhost:9200`.
9. **Dashboard certificate repair:** The Dashboard service failed because `/etc/wazuh-dashboard/certs/dashboard-key.pem` was missing. The key, certificate, and root CA were extracted from `wazuh-install-files.tar` into `/etc/wazuh-dashboard/certs`, with ownership `wazuh-dashboard:wazuh-dashboard`, directory mode `500`, and files mode `400`.
10. **Dashboard-to-Indexer URL repair:** Dashboard logs showed connection refusal to IPv6 loopback `::1:9200`. Indexer was listening at `10.0.2.15:9200`; the config was changed to `opensearch.hosts: https://10.0.2.15:9200`.
11. **Last confirmed endpoint result:** `curl -k -I https://127.0.0.1:443` returned `HTTP/1.1 302 Found`, `location: /app/login?`, and `osd-name: trooper`. This confirms the Dashboard HTTP endpoint was responding. The last known next step was to open the Dashboard in a browser, confirm login, check all service states, and then configure NAT forwarding/Windows Agent connectivity. Those final steps were not confirmed in the available history.

---

## 16. Official references used to verify the clean procedure

These are official Wazuh documentation pages consulted while preparing the runbook. They describe the supported installation process; the Kali-specific failures/fixes above come from the user's recovered lab history.

- Wazuh Quickstart (single-node all-in-one installation, requirements, `-a`, credentials, repository updates): https://documentation.wazuh.com/current/quickstart.html
- Wazuh Indexer assisted installation (configuration generation, node names, cluster start and health checks): https://documentation.wazuh.com/current/installation-guide/wazuh-indexer/installation-assistant.html
- Wazuh Dashboard step-by-step installation and certificate deployment: https://documentation.wazuh.com/current/installation-guide/wazuh-dashboard/step-by-step.html
- Wazuh Dashboard assisted installation: https://documentation.wazuh.com/current/installation-guide/wazuh-dashboard/installation-assistant.html
- Wazuh agent enrollment troubleshooting and required ports: https://documentation.wazuh.com/current/user-manual/agent/agent-enrollment/troubleshooting.html
- Wazuh central-component uninstall instructions: https://documentation.wazuh.com/current/installation-guide/uninstalling-wazuh/central-components.html

**Final rule:** Start with a supported OS, choose the network topology first, use one consistent official installation path, preserve the generated archive/passwords, verify each component before proceeding, and never debug a later layer until the earlier layer is healthy.
