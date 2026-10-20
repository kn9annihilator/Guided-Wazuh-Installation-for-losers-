# Wazuh Agent Deployment on VirtualBox: NAT, Host-Only, Bridged, NAT Network, and Dashboard Enrollment

**Purpose:** A practical runbook for connecting Windows or other
endpoints to a Wazuh manager running in a Kali Linux VirtualBox VM.

**Reference environment used in the examples** - Wazuh manager/server:
`v4.14.9`, revision `rc2` - Manager VM: Kali Linux on VirtualBox 7.2.x -
Endpoint example: Windows host OS - Wazuh 4.x default ports: TCP `1514`
for agent traffic; TCP `1515` for enrollment - Windows agent default
installation directory: `C:\Program Files (x86)\ossec-agent`



------------------------------------------------------------------------

## 1. Understand the three separate pieces

1.  **Network reachability:** Can the endpoint open a TCP connection to
    the manager?
2.  **Enrollment / authentication:** Is the endpoint registered with the
    manager and using the correct agent key?
3.  **Dashboard visibility:** Does the dashboard show the enrolled agent
    and its current status?

A successful TCP test proves only reachability. It does not prove that
enrollment or authentication succeeded.

### Wazuh 4.x ports

  --------------------------------------------------------------------------
  Port              Protocol          Purpose            Needed when
  ----------------- ----------------- ------------------ -------------------
  `1514`            TCP (in this      Agent-to-manager   Always for this TCP
                    guide)            event/telemetry    setup
                                      channel            

  `1515`            TCP               Automatic agent    When using
                                      enrollment using   `authd`/automatic
                                      the enrollment     enrollment
                                      service            

  `55000`           TCP/HTTPS API     Wazuh server API;  Only when the
                                      can be used by     chosen workflow
                                      API-based          needs the API
                                      workflows          

  `443`             TCP/HTTPS         Wazuh Dashboard    For browser access
                                      web UI in a        to the dashboard,
                                      typical            not agent telemetry
                                      installation       
  --------------------------------------------------------------------------

**Do not confuse the Dashboard with the manager.** Adding an agent in
the Dashboard does not make the Dashboard a relay for agent traffic. The
endpoint still needs a working route to the manager's
agent-communication port.

Wazuh 5.x changes the default agent communication/enrollment model
(typically TCP 1517). This guide's port examples are for Wazuh 4.x,
including the reference Wazuh 4.14.9 environment.

------------------------------------------------------------------------

## 2. Choose the VirtualBox network mode

  ------------------------------------------------------------------------------------
  VirtualBox mode Windows host →    Other LAN devices →  Kali internet  Typical use
                  Kali VM           Kali VM              access         
  --------------- ----------------- -------------------- -------------- --------------
  **NAT + port    Yes, through      Possible only if     Yes            Good for a
  forwarding**    forwarded host    forwarding binds to                 single VM; can
                  ports             a reachable host                    be extended
                                    interface and                       carefully
                                    host/network                        
                                    firewalls allow it                  

  **Host-only**   Yes, directly via Normally no          No, unless a   Private lab
                  the host-only                          second adapter link between
                  adapter IP                             is added       host and VM

  **Bridged**     Yes, using Kali's Yes, if LAN          Yes, through   Multiple
                  LAN IP            routing/firewall     the LAN        physical
                                    permits it                          endpoints on
                                                                        the same
                                                                        network

  **NAT Network** Usually via       Other VMs on the     Yes            A private
                  configured        same NAT Network can                network shared
                  forwarding        communicate;                        by multiple
                                    external devices                    VMs
                                    need                                
                                    forwarding/routing                  

  **Internal      No direct host    Only VMs attached to No, unless a   Isolated
  Network**       connection        the same internal    router VM is   VM-to-VM lab
                                    network              added          

  **Generic /     Depends on        Depends on topology  Depends on     Specialized
  cloud           driver/provider                        topology       setups; not
  networking**                                                          the simple
                                                                        default
  ------------------------------------------------------------------------------------

**Practical choices** - Only Windows host needs to send telemetry:
**NAT + localhost port forwarding** is a valid choice. - Host and
several lab VMs need access, but no physical LAN access is wanted:
**NAT + Host-only Adapter** (two adapters) is usually simplest. - Other
computers on the same LAN must send telemetry: **Bridged**, or NAT
forwarding bound to the host's LAN interface with strict firewall
rules. - Multiple VMs need to communicate with one another on a private
shared network: consider **NAT Network** or **Internal Network**,
depending on internet/host requirements.

------------------------------------------------------------------------

## 3. Before configuring networking: check the manager

On Kali, run:

``` bash
sudo /var/ossec/bin/wazuh-control info
sudo ss -lntp | grep -E ':(1514|1515|55000)\b'
```

Expected version for the reference setup: `v4.14.9` (revision `rc2`).

Confirm the manager's active configuration:

``` bash
sudo grep -n -A 12 -B 2 '<remote>' /var/ossec/etc/ossec.conf
sudo grep -n -A 18 -B 2 '<auth>' /var/ossec/etc/ossec.conf
```

For Wazuh 4.x, a typical configuration includes a `remote` block using
TCP `1514` and an enabled `auth` block on TCP `1515`. Do not paste
duplicate XML blocks into the file. If editing manager configuration,
preserve the existing structure, validate changes, and restart only the
affected service when necessary.

Check service status:

``` bash
sudo systemctl status wazuh-manager --no-pager
```

Find Kali's addresses:

``` bash
ip -br addr
ip route
```

The `lo` interface is loopback and is not the VM's LAN/host-only
address.

------------------------------------------------------------------------

## 4. Option A --- NAT with port forwarding (Windows host agent)

This is the reference setup used when the Kali VM has **Adapter 1 =
NAT** and the agent runs on the Windows host itself.

### 4.1 Add VirtualBox rules

1.  Shut down the Kali VM.
2.  VirtualBox Manager → select Kali VM → **Settings → Network → Adapter
    1**.
3.  Keep **Attached to: NAT**.
4.  Open **Advanced → Port Forwarding**.
5.  Add these rules:

  ----------------------------------------------------------------------------------
  Rule             Protocol   Host IP            Host Port Guest IP       Guest Port
  ---------------- ---------- ------------- -------------- ---------- --------------
  `Wazuh-Agent`    TCP        `127.0.0.1`           `1514` Leave              `1514`
                                                           blank      
                                                           unless     
                                                           your setup 
                                                           requires a 
                                                           fixed      
                                                           guest      
                                                           address    

  `Wazuh-Enroll`   TCP        `127.0.0.1`           `1515` Leave              `1515`
                                                           blank      
                                                           unless     
                                                           your setup 
                                                           requires a 
                                                           fixed      
                                                           guest      
                                                           address    
  ----------------------------------------------------------------------------------

6.  Save and start Kali.

Binding the host side to `127.0.0.1` makes these forwarded ports
available only on the Windows host's loopback interface. This is
appropriate for a host-only endpoint and avoids exposing these ports to
the LAN.

### 4.2 Test from Windows

Open PowerShell:

``` powershell
Test-NetConnection 127.0.0.1 -Port 1514
Test-NetConnection 127.0.0.1 -Port 1515
```

For both, look for `TcpTestSucceeded : True`.

If either fails: - Check that Kali is running and the rules are saved
under the correct VM/adapter. - On Kali, verify the manager is listening
on the guest-side port. - Check whether the manager service is
running. - Do not proceed to agent enrollment until the relevant port
test succeeds.

### 4.3 Configure the Windows agent

Use **one** enrollment method from Section 7. For a Windows host
connecting through the rules above, the manager address is `127.0.0.1`,
because the host is connecting to its own forwarded port---not directly
to Kali's NAT address such as `10.0.2.15`.

For a manually configured Windows agent, the manager section normally
resembles:

``` xml
<client>
  <server>
    <address>127.0.0.1</address>
    <port>1514</port>
    <protocol>tcp</protocol>
  </server>
</client>
```

This is an excerpt only: edit the existing `ossec.conf`; do not replace
the whole file with this snippet.

Default Windows path:

``` text
C:\Program Files (x86)\ossec-agent\ossec.conf
```

Restart the agent from an elevated PowerShell:

``` powershell
Restart-Service WazuhSvc
Get-Service WazuhSvc
```

**For more physical/LAN agents:** Do not leave host binding at
`127.0.0.1` and expect other computers to connect. You would need to
bind the forwarded host ports to the Windows host's reachable LAN
address (or another deliberately chosen interface), configure Windows
Firewall, and ensure network routing. See Section 5.3 and the security
notes.

------------------------------------------------------------------------

## 5. Option B --- Host-only adapter

Host-only provides a private network between Windows and Kali. It is
often the easiest lab arrangement if you want to avoid NAT port
forwarding.

### 5.1 Keep internet access and add the private link

1.  Shut down Kali.
2.  **Adapter 1:** keep `NAT` for internet access.
3.  **Adapter 2:** enable it and select `Host-only Adapter` (or the
    corresponding Host-only Network option in your VirtualBox version).
4.  Select the existing VirtualBox host-only network/adapter.
5.  Start Kali.

### 5.2 Find the host-only addresses

On Kali:

``` bash
ip -br addr
```

Find the IPv4 address on the host-only interface (often a `192.168.56.x`
address, but use the actual output).

On Windows, open PowerShell:

``` powershell
ipconfig
```

Find **VirtualBox Host-Only Ethernet Adapter** and note its IPv4
address. The Windows and Kali host-only addresses should be in the same
subnet.

Test from Windows, replacing the example with Kali's actual host-only
IP:

``` powershell
Test-NetConnection <KALI_HOST_ONLY_IP> -Port 1514
Test-NetConnection <KALI_HOST_ONLY_IP> -Port 1515
```

### 5.3 Configure the agent

Set the manager address to Kali's **host-only IP**, for example:

``` xml
<address>192.168.56.101</address>
```

The example is not guaranteed to match your setup. Use the address
actually shown by `ip -br addr`.

If TCP tests fail while the host-only IP is correct: - Confirm Wazuh is
listening on an interface reachable through host-only, not only on
Kali's loopback. - Check Kali firewall rules. - Check the manager's
`remote` listener configuration. - Check the enrollment listener if
using automatic enrollment.

Host-only does not provide internet access to the VM by itself. Keep a
separate NAT adapter if Kali also needs internet.

------------------------------------------------------------------------

## 6. Option C --- Bridged adapter

Bridged mode places Kali on the same physical LAN as the Windows host,
like another machine on that network.

### 6.1 Configure VirtualBox

1.  Shut down Kali.
2.  Open **Settings → Network → Adapter 1**.
3.  Set **Attached to: Bridged Adapter**.
4.  Select the actual Windows network interface in use (Wi-Fi or
    Ethernet).
5.  Start Kali.

### 6.2 Find Kali's LAN IP

On Kali:

``` bash
ip -br addr
ip route
```

Use Kali's LAN IPv4 address---not `127.0.0.1` and not the NAT-only
address from a different adapter.

From Windows, test:

``` powershell
Test-NetConnection <KALI_LAN_IP> -Port 1514
Test-NetConnection <KALI_LAN_IP> -Port 1515
```

### 6.3 Configure the agent and firewall

Use Kali's reachable LAN IP as the manager address in the agent
configuration or dashboard deployment instructions.

For agents to connect, the manager must listen on a reachable interface
and Kali's firewall/network must permit the required ports. Check your
actual listener before changing firewall rules:

``` bash
sudo ss -lntp | grep -E ':(1514|1515)\b'
```

Bridged networking may be blocked by Wi-Fi client isolation, enterprise
networks, VPNs, or network access controls. A DHCP-assigned IP may also
change; use a DHCP reservation or another stable addressing method if
agents depend on that address.

**Security:** Bridged mode makes the VM more visible on the LAN. Do not
expose Wazuh ports or the dashboard to untrusted networks. Limit source
addresses in firewalls wherever possible.

------------------------------------------------------------------------

## 7. Option D --- NAT Network, Internal Network, and other modes

### NAT Network

-   Create/select a NAT Network in VirtualBox Network Manager and attach
    the required VMs to it.
-   VMs on the same NAT Network can communicate with each other.
-   To connect from the Windows host or from external LAN devices, use
    the appropriate port forwarding/routing and firewall design.
-   The manager address must be reachable from each agent's network.

### Internal Network

-   Provides communication only among VMs attached to the same named
    internal network.
-   The Windows host cannot normally connect directly to the guest
    through this network.
-   To use it for Wazuh, place a router/gateway VM or add another
    adapter that provides a route from the endpoint network.

### Bridged + NAT (two adapters)

-   Useful when the VM needs normal NAT connectivity as well as a
    bridged interface, but often unnecessary.
-   Ensure the manager listens on the intended interface and use the
    correct IP for agents.
-   Avoid ambiguous routing or exposing services on both interfaces
    unintentionally.

### Generic, cloud, or other advanced modes

These depend on the selected driver/provider and network topology. Do
not assume they behave like NAT, host-only, or bridged. Confirm
endpoint-to-manager routing and firewall access before enrolling agents.

------------------------------------------------------------------------

## 8. Enrollment methods --- choose one

There are two common approaches for Wazuh 4.x. Do not confuse a
**pre-shared agent key** with an **enrollment password**.

### Method 1: Dashboard "Deploy new agent" (recommended for guided setup)

1.  Sign in to the Wazuh Dashboard.
2.  Go to **Agents management → Summary → Deploy new agent** (wording
    may vary slightly by release).
3.  Select the endpoint OS and the matching package/version.
4.  Enter the manager address that is reachable from that endpoint:
    -   Windows host + NAT forwarding bound to localhost: `127.0.0.1`
    -   Host-only: Kali's host-only IP
    -   Bridged: Kali's LAN IP
5.  Follow the exact command/instructions shown by your dashboard for
    your selected OS and version.
6.  Run the installation/configuration command as Administrator on
    Windows.
7.  Start the agent service if the dashboard's command does not already
    do so.
8.  Confirm the agent appears in the dashboard and reaches **Active**.

The Dashboard guides deployment, but it does not relay agent traffic.
For the normal Wazuh 4.x arrangement, the endpoint still needs TCP 1514
to the manager. Automatic enrollment also needs TCP 1515. If a workflow
specifically uses the Wazuh API, it may need TCP 55000 as well.

Do not copy a command intended for another manager address or network
mode. A command generated for a LAN-reachable server will not work
unchanged if your Windows host must use a localhost NAT forward.

### Method 2: Pre-register the agent and import its client key

Use this when the Windows agent GUI explicitly asks for **Manager IP**
and **Auth key**, or when you prefer to register the agent manually.

1.  On Kali, run:

    ``` bash
    sudo /var/ossec/bin/manage_agents
    ```

2.  Choose **Add agent** and create a unique agent name, such as
    `windows-host`.

3.  For the agent IP prompt, choose `any` if the agent's source IP can
    vary and your policy permits it.

4.  Record the assigned agent ID.

5.  Choose **Extract key**, select that ID, and copy the complete key.

6.  In the Windows agent configuration GUI:

    -   **Manager IP:** use the address reachable from Windows for the
        selected network mode.
    -   **Auth key:** paste the complete key from `manage_agents`.

7.  Apply/save the configuration and restart `WazuhSvc`.

8.  Confirm the agent becomes active.

The key from **Extract key** is an agent client key. It is not the same
thing as an enrollment password. For manual key import, the agent must
be pre-registered and the key must match that entry.

### Avoid duplicate agent names

If the log reports:

``` text
Duplicate agent name: windows-host. Unable to add agent
```

The manager already has an agent entry with that name. Do not repeatedly
retry automatic enrollment with the same name. Either: - Use the
existing registered entry/key for that endpoint, or - Remove the
stale/incorrect entry in the Dashboard or `manage_agents`, then enroll
again with a unique name.

Do not remove agent ID `000` (the manager itself). Confirm the ID and
identity before deleting an entry.

------------------------------------------------------------------------

## 9. Quick validation checklist

### On Kali

``` bash
sudo systemctl is-active wazuh-manager
sudo ss -lntp | grep -E ':(1514|1515|55000)\b'
sudo /var/ossec/bin/agent_control -lc
```

Expected: - Manager service reports `active`. - Required ports have
listeners. - Your endpoint appears in the agent list. `ID 000` alone is
the manager, not a Windows endpoint.

### On Windows

``` powershell
Get-Service WazuhSvc
Test-NetConnection <MANAGER_ADDRESS> -Port 1514
```

If using automatic enrollment, also test:

``` powershell
Test-NetConnection <MANAGER_ADDRESS> -Port 1515
```

Use the manager address appropriate for the chosen network mode. For NAT
forwarding bound to `127.0.0.1`, use `127.0.0.1` from the Windows host.
For host-only or bridged, use Kali's corresponding reachable IP.

Default log path:

``` powershell
Get-Content "C:\Program Files (x86)\ossec-agent\ossec.log" -Tail 30
```

If the default directory differs, inspect
`C:\Program Files\ossec-agent\ossec.log`.

### In the Dashboard

Open **Agents management → Summary** and check: - The endpoint is listed
under the expected name. - Status is **Active**. - The last keep-alive
updates.

A successful port test alone does not prove successful enrollment. A
listed agent that is not active still needs authentication or
communication troubleshooting.

------------------------------------------------------------------------

## 10. Troubleshooting by symptom

  ----------------------------------------------------------------------------------------------------
  Symptom                                 Likely meaning          First action
  --------------------------------------- ----------------------- ------------------------------------
  `Test-NetConnection ... -Port 1514`     Wrong address, missing  Confirm network mode/IP and the
  fails                                   forward, listener not   manager listener
                                          reachable, or firewall  

  `1514` passes but `1515` fails          Agent traffic path      Check enrollment rule/listener; if
                                          works, enrollment path  using manual key import, enrollment
                                          does not                port may not be needed for that step

  Log keeps showing an old IP             The active              Check `<address>` in `ossec.conf`,
                                          configuration may not   save, restart `WazuhSvc`, inspect
                                          have been updated or    only new log lines
                                          service was not         
                                          restarted               

  `Duplicate agent name`                  Existing registration   Reuse the existing entry or remove
                                          uses that name          the correct stale entry before
                                                                  enrolling again

  Agent appears but is not Active         Enrollment may have     Check port 1514, agent logs, manager
                                          succeeded but the       logs, and the registered entry
                                          ongoing connection is   
                                          not established, or the 
                                          agent key/config does   
                                          not match               

  `No authentication password provided`   No enrollment password  This is not necessarily an error
                                          was configured          when
                                                                  `<use_password>no</use_password>`;
                                                                  don't invent a password

  Agent is absent; only ID `000` appears  No endpoint agent is    Complete enrollment and verify the
                                          registered in the       agent name/key
                                          manager list            

  Dashboard opens, but agent cannot       Browser access to the   Test the agent ports from the
  connect                                 dashboard does not      endpoint to the manager
                                          prove port 1514/1515    
                                          reachability            

  Works until VM restarts                 Address may have        Check current IP, VirtualBox rules,
                                          changed or a network    service status, and DHCP
                                          rule/listener did not   reservations
                                          persist                 
  ----------------------------------------------------------------------------------------------------

Useful manager logs:

``` bash
sudo tail -n 80 /var/ossec/logs/ossec.log
```

Useful Windows logs:

``` powershell
Get-Content "C:\Program Files (x86)\ossec-agent\ossec.log" -Tail 50
```

Do not repeatedly reinstall the agent as the first response to a
connectivity or duplicate-name error. Verify the endpoint address,
ports, enrollment state, and key first.

------------------------------------------------------------------------

## 11. Security and scaling notes

-   For one Windows host using NAT, binding forwarded ports to
    `127.0.0.1` is the narrowest exposure.
-   For multiple physical agents, each endpoint must have a network
    route to the manager. Localhost forwarding is not reachable from
    other computers.
-   If you expose forwarded ports on a LAN interface, restrict access
    with host and guest firewalls and use trusted networks only.
-   Do not expose the Wazuh Dashboard or API to the public internet just
    to enroll agents.
-   Prefer stable addressing (DHCP reservation/static IP/DNS) for agents
    that must reconnect after VM or network restarts.
-   Keep a record of each agent's name, ID, endpoint, network mode,
    manager address, and enrollment method. Never publish or share agent
    authentication keys.
-   When adding many agents, consider a stable manager VM/network design
    rather than relying on a host-only localhost rule for every
    endpoint.

------------------------------------------------------------------------

## 12. Official references

-   [Wazuh 4.14 --- Windows agent
    installation](https://documentation.wazuh.com/4.14/installation-guide/wazuh-agent/wazuh-agent-package-windows.html)
-   [Wazuh --- Agent enrollment
    requirements](https://documentation.wazuh.com/current/user-manual/agent/agent-enrollment/requirements.html)
-   [Wazuh --- Windows deployment
    variables](https://documentation.wazuh.com/current/user-manual/agent/agent-enrollment/deployment-variables/deployment-variables-windows.html)
-   [Wazuh --- Agent enrollment
    troubleshooting](https://documentation.wazuh.com/current/user-manual/agent/agent-enrollment/troubleshooting.html)
-   [Oracle VirtualBox 7.2 --- Virtual
    networking](https://docs.oracle.com/en/virtualization/virtualbox/7.2/user/networkingdetails.html)

**Last updated:** 2026-10-20\
**Scope:** Wazuh 4.x examples; verify ports and instructions for other
major versions.
