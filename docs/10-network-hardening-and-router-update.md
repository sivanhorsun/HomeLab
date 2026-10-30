# Phase 10 — Network Hardening and Router Update

**Result:** I reviewed the collaborator's effective Proxmox permissions and available storage, created an ISO-upload role, and granted the collaborator full `sudo` access on the Pi. Network discovery exposed Telnet listeners on the router and switch; both were disabled. I also disabled unused FTP, HTTP, and API services on the MikroTik, verified the Pi's time synchronization, and corrected its time zone. After downloading an encrypted router backup, I updated RouterOS and RouterBOOT separately to **7.23.8**, then verified that the closed services and recorded firewall configuration survived the reboots.

> **Historical scope:** October 28–29, 2026, following Phase 9. This phase records management-service hardening and maintenance of the existing flat lab network. It did not introduce another VLAN, change Tailscale grants, or restrict internet access to update repositories. Creating the ISO-upload role was confirmed; assigning and testing it was not. Versions and resource figures describe the recorded sessions, not a live inventory.

Previous phase: [VM Permissions, PHP Review, and SSH Hardening](09-vm-permissions-php-review-and-ssh-hardening.md). Project overview: [README](../README.md).

> **Publication convention:** Private LAN addresses are retained. Tailscale addresses remain illustrative: Pi `100.101.10.10`, MacBook `100.101.10.20`, Proxmox `100.101.10.30`, and Ubuntu `100.101.10.40`. `lab-user`, `lab-user@pve`, and `collaborator-lab` replace personal account and pool names. Actual overlay identifiers, serial numbers, MAC addresses, fingerprints, account emails, passwords, and raw backups are omitted. Command examples preserve the recorded operations with these substitutions; they are not an instruction to repeat completed changes.

## 1. Objective

After completing key-only SSH on Ubuntu VM 100, I continued with the permissions and network services around the guests. The practical goals were to understand the collaborator's remaining access problems, remove unused plaintext management services, resolve the apparent clock discrepancy, and update the lab router without losing the existing configuration.

The work stayed within the original project direction: learning networking, Linux administration, virtualization, and security through a working home lab.

## 2. Starting point after Phase 9

| Component | State at the start of this phase |
| --- | --- |
| Ubuntu VM 100 | Public-key SSH worked; password-only authentication was rejected after the SSH configuration reload |
| Collaborator VM 101 | Creation and activity appeared in Proxmox Tasks; a complete guest check remained pending |
| Proxmox permissions | Pool, storage, and network roles existed; the collaborator reported difficulty uploading images and adding storage |
| Raspberry Pi | Kali client with remote SSH over Tailscale; collaborator account initially lacked `sudo` |
| Lab network | Shared `192.168.88.0/24` LAN; MikroTik routed traffic through its wireless household uplink |
| Router policy | Earlier IPv4 household-subnet and inbound-WAN drops, plus a routed-IPv6 drop, were already configured |

The PHP update and Ubuntu SSH work remain documented in [Phase 9](09-vm-permissions-php-review-and-ssh-hardening.md). This phase did not repeat them or claim that they had hardened every other machine.

## 3. Work sequence

| Stage | Recorded work |
| --- | --- |
| Permission review | Read effective Proxmox permissions and storage usage; created `LabISOUploader` |
| Pi administration | Added the collaborator to `sudo` and verified group membership |
| Network observation | Practiced `journalctl`, `tcpdump`, and Nmap; found Telnet listeners |
| Clock investigation | Verified NTP synchronization and changed the Pi's time zone |
| Management hardening | Disabled Telnet on router and switch; disabled FTP, HTTP, and API services on the router |
| Router review | Inspected users, scripts, scheduler output, and software version |
| Backup and update | Downloaded an encrypted backup; updated RouterOS, then RouterBOOT, with separate reboots |
| Final validation | Rechecked selected ports, IPv4 rules, interface lists, and the IPv6 forward drop |

## 4. Effective Proxmox permissions

From the **Proxmox host shell**, I inspected the collaborator's effective permissions:

```bash
pveum user permissions lab-user@pve
```

| ACL path | Relevant permissions observed |
| --- | --- |
| `/pool/collaborator-lab` and `/vms/101` | `VM.Allocate`, `VM.Config.Disk`, other VM configuration permissions, console access, power management, and additional VM operations |
| `/storage/local-lvm` | `Datastore.AllocateSpace`, `Datastore.Audit` |
| `/storage/local` | Audit permissions; `Datastore.AllocateTemplate` was absent |
| `/sdn/zones/localnetwork/vmbr0` | `SDN.Use`, `SDN.Audit` |

The account already had disk-configuration and storage-allocation permissions in the relevant scopes. The report did not establish that the storage problem was caused by a missing permission. The exact failed operation and error message were still needed.

This inspection added evidence beyond the Phase 9 role-assignment table. Deliberate tests of denied operations outside the intended scope were not completed.

## 5. Available host storage

On **Proxmox**:

```bash
pvesm status
```

| Storage | Type | Total (KiB) | Used (KiB) | Available (KiB) | Used |
| --- | --- | ---: | ---: | ---: | ---: |
| `local` | `dir` | 71,021,728 | 10,209,856 | 57,158,448 | 14.38% |
| `local-lvm` | `lvmthin` | 148,107,264 | 8,827,192 | 139,280,071 | 5.96% |

Both were active. Their reported available space was approximately **54.5 GiB** and **132.8 GiB**, respectively.

Free host storage does not establish how much space the filesystem inside VM 101 can use. The collaborator reported an Ubuntu installation, but the guest's `df -h /`, `lsblk`, disk configuration, and exact error had not been collected. No disk enlargement, partition growth, filesystem resize, or successful guest update was confirmed in this phase.

## 6. Preparing ISO-upload permissions

I created a custom Proxmox role named **`LabISOUploader`** with:

- `Datastore.Audit`
- `Datastore.AllocateTemplate`

The intended assignment was to `lab-user@pve` at **`/storage/local`**, where the installation images are stored, with Propagate enabled. The role was created once with both privileges.

**Completion boundary:** the conversation confirmed role creation, but did not confirm the final ACL assignment or a successful ISO upload from the collaborator account. Those checks remain open. No global Administrator role or Proxmox root account was granted through this change.

## 7. Installation image versus installed operating system

We clarified that an ISO is installation or boot media. Attaching a different ISO to an existing VM does not automatically replace the operating system already installed on its virtual disk.

The collaborator wanted to choose another OS because only Ubuntu media was available. Upload permission would address image availability; creating a new guest or deliberately reinstalling an existing one is a separate operation. No guest reinstall or image replacement was recorded here.

## 8. Administrative access on the Raspberry Pi

Inside **Kali on the Pi**, the existing administrator added the collaborator to the `sudo` group:

```bash
sudo usermod -aG sudo lab-user
id lab-user
```

The returned group list included **`27(sudo)`**, alongside the account's own group and `users`. The collaborator then reconnected.

This grants the normal administrative capability associated with the Pi's `sudo` group. It is **full sudo access**, not a rule limited to package updates or one application. It does not grant Proxmox host root access, change the Tailscale policy, or create an account inside a VM. A separate `sudo whoami` result was not supplied.

## 9. Reading live system logs

On the **Pi**:

```bash
sudo journalctl -f
```

`-f` follows new journal entries. **Ctrl+C** stops following the log without stopping the services being observed.

| Observation | Interpretation and limit |
| --- | --- |
| Tailscale peer-contact messages through DERP | Connection activity; the lines alone did not establish an intrusion |
| PHP session cleanup through cron/systemd | Scheduled maintenance of session files; not evidence that a PHP web server was listening |
| `sudo` entry for `journalctl -f` | The administrator's own log-viewing command |
| `/etc/resolv.conf` overwritten by `dhcpcd` | Tailscale detected a competing DNS configuration change, restored its expected configuration, then reported the warning cleared |

The DNS contention was observed but not permanently resolved in this phase. No DNS service was disabled and no replacement resolver configuration was applied.

## 10. Packet-capture practice

On the **Pi**:

```bash
tcpdump -D
sudo tcpdump -i eth0
```

The first command listed available capture interfaces. The second displayed packets visible to the Pi's Ethernet interface. No `-w` output file or saved packet capture was recorded.

An `eth0` capture is not a view of every conversation crossing the switch. It observes traffic available on that interface. Tailscale traffic on the underlying interface remains encrypted, and ordinary SSH also protects its session contents. Stopping the capture with Ctrl+C does not interrupt those connections.

## 11. Discovering TCP listeners on the lab LAN

From the **Pi**, we scanned the lab subnet:

```bash
sudo nmap -Pn 192.168.88.0/24
```

| Address | Device in this lab | Open TCP ports reported |
| --- | --- | --- |
| `192.168.88.1` | MikroTik router | 21, 22, 23, 53, 80, 8291 |
| `192.168.88.2` | Proxmox host | 22, 111, 3128 |
| `192.168.88.249` | Ubuntu VM 100 | 22 |
| `192.168.88.252` | TP-Link switch | 22, 23, 80, 443 |
| `192.168.88.250` | Raspberry Pi at this time | 22; the other 999 scanned ports were reported filtered |

The scan reported five hosts up. This was the default common-TCP-port selection, **not all TCP ports**, and did not test UDP. `-Pn` skips the normal host-discovery stage; it does not establish application identity or prove that every address is active.

Two interpretation details mattered:

- **`.252` is the switch; Proxmox is `.2`.** The initial informal description mixed them up.
- Nmap labeled TCP 3128 `squid-http`, but the earlier Proxmox process inventory identified **`spiceproxy`** on that port. This scan did not use service-version detection. The absence of 8006 from this report was not evidence that the Proxmox interface was closed.

The Pi's `.250` address was a current DHCP observation, not a newly configured reservation.

## 12. What the Telnet attempt established

The collaborator connected from the Pi to the router:

```bash
telnet -4 192.168.88.1 23
```

The connection reached a **Login** prompt, confirming that the service was reachable from the lab. It still required credentials, and no successful router authentication was shown. The later `telnet>` prompt belonged to the Telnet client; `ls` there was not a command executed on the router.

The security issue was plaintext management traffic, including credentials when entered, rather than a demonstrated password bypass. An open Telnet listener was not proof of exploitation, a particular CVE, or public internet exposure. We decided to disable the unused service and keep SSH as the management path.

## 13. NTP and firewall direction

We reviewed the MikroTik IPv4 filter rules before considering changes for NTP. For the Pi to synchronize with an external time server, it initiates traffic to **UDP 123** and receives replies associated with that exchange.

The recorded policy retained general outbound internet access and accepted established/related return traffic. It did not require an unsolicited inbound-WAN opening on UDP 123 for this client use.

**No NTP-specific firewall exception or port forward was added.** The following client-side results provided direct evidence that synchronization was already working.

## 14. Verifying synchronization on the Pi

On the **Pi**:

```bash
timedatectl status
timedatectl timesync-status
```

| Field | Recorded value |
| --- | --- |
| System clock synchronized | `yes` |
| NTP service | `active` |
| Initial time zone | `Etc/UTC` |
| Server | `0.debian.pool.ntp.org`, resolved to `69.10.208.170` |
| NTP version / stratum | 4 / 2 |
| Offset | `+737us` |
| Delay | `67.895ms` |
| Jitter | `1.668ms` |
| Packet count | 546 |

These results showed successful synchronization with an external server. The apparent local-time discrepancy did not call for opening inbound NTP. No new NTP package installation was confirmed in the record.

## 15. Correcting the displayed time zone

On the **Pi**:

```bash
sudo timedatectl set-timezone America/Chicago
timedatectl status
```

The next output showed local time **11:40:32 CDT**, universal time **16:40:32 UTC**, and `America/Chicago (CDT, -0500)`. Synchronization remained `yes`, NTP remained `active`, and the RTC remained in UTC.

This changed how local time was displayed. It did not replace the time source or require a new firewall rule.

## 16. Reaching the router through the Pi

The established remote path was **MacBook → Tailscale → Pi → lab router**. From the Pi:

```bash
ssh admin@192.168.88.1
```

Before trusting the router's SSH host key on the Pi, I compared its fingerprint with the existing entry on the MacBook:

```bash
ssh-keygen -F 192.168.88.1 -l -E sha256
```

The fingerprints matched. The router key was then accepted on the Pi. No host key was regenerated, no known-host entry was deleted to bypass the check, and no public router-management port was opened.

## 17. SSH warnings and transient connection failures

| Message or observation | What happened |
| --- | --- |
| Non-post-quantum key-exchange warning | The client warned about the negotiated exchange; this did not mean that SSH was unencrypted |
| Padding / message-authentication-code error | An initial connection failed; the root cause was not established |
| Verbose SSH retry | Reached password authentication using `curve25519-sha256` and `aes128-gcm@openssh.com` |
| Authentication timeout | A later attempt exceeded the authentication window; retrying and entering the router password allowed login |

The successful connection did not establish why the earlier attempt failed. No router cipher policy was changed, and the later software update was not followed by a test proving that the post-quantum warning or original error had disappeared.

## 18. Disabling Telnet on the MikroTik

In the **RouterOS terminal**, with SSH access working:

```routeros
/ip service disable [find where name="telnet"]
/ip service print detail where name="telnet"
```

The service appeared with the **`X` disabled flag**. A subsequent port check was reported closed. The final post-upgrade scan later confirmed TCP 23 closed again.

This disabled the router's Telnet service. It did not disable SSH or add a network-wide prohibition on all traffic using port 23.

## 19. Reaching the switch and disabling its Telnet service

On the **MacBook**, I opened a local SSH forward through the Pi. The overlay address is normalized here:

```bash
ssh -N -L 127.0.0.1:8443:192.168.88.252:443 kali@100.101.10.10
```

After authentication, the terminal stayed open without presenting a shell. That was expected: `-N` runs the forwarding session without a remote command. I opened **`https://localhost:8443`**, reached the TP-Link management interface, and disabled Telnet there.

The subsequent port check was reported closed. This used the Pi's existing lab reachability; it did not advertise the whole lab through Tailscale or expose the local forwarding socket on all MacBook interfaces.

The switch was not rebooted in this phase. Persistence of this latest service change across a switch restart remains unverified. Switch HTTP was not disabled in this session.

## 20. Reviewing the remaining router services

The MikroTik service list was inspected before disabling additional listeners:

| Service | Port | Observed state before the remaining changes |
| --- | --- | --- |
| FTP | TCP 21 | Enabled |
| SSH | TCP 22 | Enabled; in use for administration |
| Telnet | TCP 23 | Already disabled |
| HTTP WebFig (`www`) | TCP 80 | Enabled |
| HTTPS WebFig (`www-ssl`) | TCP 443 | Disabled; no certificate configured |
| Winbox | TCP 8291 | Enabled |
| API | TCP 8728 | Enabled |
| API over TLS | TCP 8729 | Enabled; no certificate configured |

Dynamic DNS, DHCP, and discovery entries also appeared. They were not all classified as unnecessary or disabled as a group. The service inventory and the firewall answer different questions: a listener may exist while some network paths to it are blocked.

## 21. Disabling unused FTP, HTTP, and API listeners

In the **RouterOS terminal**, the unused management services were disabled:

```routeros
/ip service disable [find where name="ftp"]
/ip service disable [find where name="www"]
/ip service disable [find where name="api"]
/ip service disable [find where name="api-ssl"]
```

The service flags were checked, then the following scan was run from **Kali**, not from RouterOS:

```bash
nmap -Pn -p 21,23,80,8728,8729 192.168.88.1
```

All five selected ports were reported **closed**. SSH and Winbox remained the enabled management services from the reviewed list. No HTTPS replacement for the router's HTTP interface was configured; its HTTP WebFig page was therefore no longer an available management path.

These commands disabled services hosted by the router. They did not block lab clients from making outbound HTTP or FTP connections. The API change removed an unused programmatic router-management interface, not the router's ability to route normal application traffic.

## 22. Router account review and the device-history question

In **RouterOS**:

```routeros
/user print detail without-paging
```

The output showed a single account, **`admin`**, in group **`full`**, with no account-level source-address restriction displayed.

We also discussed whether the enabled services implied that the router had previously been used. The service list did not establish that. The earlier configuration included default firewall rules before the custom lab isolation rules were added, but the current output cannot reconstruct every moment since the device was first powered on.

No compromise, prior ownership, or complete historical exposure was proven. No factory reset or Netinstall was performed in this phase.

## 23. Scripts and scheduled tasks

In **RouterOS**:

```routeros
/system script print without-paging
/system scheduler print without-paging
```

The script list contained two entries labeled `defconf`, owned by `*sys`, with run counts of zero:

| Script | Visible behavior |
| --- | --- |
| `dark-mode` | Toggles the all-LEDs-off setting |
| `wps-accept` | Starts WPS push-button acceptance on eligible enabled AP interfaces |

The displayed code did not show a download, data transfer to another server, or account creation. The scheduler command returned no entries.

This was a limited review of visible configuration, not a forensic guarantee. The existence of `wps-accept` alone did not establish that WPS was actively accepting clients; the actual wireless settings were not checked here.

## 24. Selecting the update channel and creating a backup

The initial resource output showed **RouterOS 7.20.8**, a hAP ax³ with 1 GiB RAM, approximately 92.9 MiB free storage, low CPU load, and zero bad blocks.

The update checker was initially on **stable** and offered `7.25`. After selecting **long-term**, it offered **`7.23.8`**. That was the chosen target; `7.25` was not installed.

Before updating, I created a password-protected binary configuration backup. The password below is a placeholder:

```routeros
/system backup save name=before-update-2026-10-29 password="YOUR_BACKUP_PASSWORD" encryption=aes-sha256
/file print where name~"before-update-2026-10-29"
```

The file list confirmed **`before-update-2026-10-29.backup`**, approximately **37.3 KiB**. It was created after the management-service changes and before the upgrade. This binary backup was not a sanitized text export and is not included in this repository.

## 25. Downloading the backup through SFTP

The first SFTP attempt through the Pi ended with a broken pipe. A later attempt explicitly selected the cipher observed in the successful SSH session:

```bash
sftp -o Ciphers=aes128-gcm@openssh.com -J kali@100.101.10.10 admin@192.168.88.1
```

Once at the **`sftp>` prompt**, the backup was downloaded to the MacBook's Downloads directory. The account path is normalized:

```text
get before-update-2026-10-29.backup /Users/<MAC_USER>/Downloads/before-update-2026-10-29.backup
```

The transfer reached **100%**, about **37 KB**. This established that an off-router copy had been downloaded. A checksum comparison or restore test was not recorded.

The retry did not prove that cipher selection alone caused the recovery. FTP and HTTP did not have to be re-enabled for the transfer. The raw backup and its password remain outside the public repository.

## 26. Updating RouterOS

In **RouterOS**:

```routeros
/system package update download
```

The update status reported **Downloaded, please reboot router to upgrade it**. I then rebooted:

```routeros
/system reboot
```

The router SSH session closed, and the remote Pi connection timed out while the uplink was unavailable. After connectivity returned, I logged in again and checked:

```routeros
/system resource print
```

The result showed **`version: 7.23.8 (long-term)`**, an uptime of roughly seven minutes, low load, and zero bad blocks. RouterOS had updated successfully; the boot firmware still needed its own inspection.

## 27. Updating RouterBOOT separately

In **RouterOS**:

```routeros
/system routerboard print
```

After the RouterOS update, the output still showed `current-firmware: 7.20.8`, while `upgrade-firmware` was `7.23.8`. I applied the boot-firmware update and performed a second reboot:

```routeros
/system routerboard upgrade
/system reboot
```

After reconnecting, `/system routerboard print` showed:

| Field | Verified final value |
| --- | --- |
| `current-firmware` | `7.23.8` |
| `upgrade-firmware` | `7.23.8` |
| `factory-firmware` | `7.20.8` |

The unchanged factory value did not mean the update had failed. The active RouterBOOT version and the available upgrade version matched. RouterOS and RouterBOOT were updated in two separate steps, with two reboots.

## 28. Validation after both router reboots

From **Kali**, I repeated the targeted scan:

```bash
nmap -Pn -p 21,23,80,8728,8729 192.168.88.1
```

| Port | Service reviewed | Final scan state |
| --- | --- | --- |
| TCP 21 | FTP | Closed |
| TCP 23 | Telnet | Closed |
| TCP 80 | HTTP WebFig | Closed |
| TCP 8728 | RouterOS API | Closed |
| TCP 8729 | RouterOS API over TLS | Closed |

I also read the configuration in **RouterOS**:

```routeros
/ip firewall filter print without-paging
/interface list member print without-paging
/ipv6 firewall filter print without-paging
```

| Area | What the output confirmed |
| --- | --- |
| IPv4 router input | Established/related/untracked acceptance, invalid drop, ICMP and loopback exceptions, and the drop for traffic not arriving from LAN remained present |
| Household isolation | The forward drop for LAN traffic destined to `192.168.1.0/24` remained present |
| New inbound WAN forwarding | The `connection-state=new` drop on the WAN interface list remained present |
| Other IPv4 forwarding rules | IPsec rules, FastTrack, established/related acceptance, invalid drop, and the default WAN rule remained present |
| LAN interface list | `bridge` |
| WAN interface list | `ether1` and `wifi1`; the wireless uplink remained the established path |
| IPv6 forwarding | `LAB: block routed IPv6 pending configuration` remained an unconditional forward drop before FastTrack6 |

These were configuration and selected-port checks after reboot. A new household-gateway counter test, deliberate inbound-WAN test, and routed-IPv6 traffic test were not performed in this phase. The IPv6 forward rule concerns routed traffic through the MikroTik; it does not establish isolation between same-LAN devices or describe the inner traffic of a Tailscale connection.

The firewall was inspected, not rewritten. General internet egress remained permitted under the existing design.

## 29. End-of-phase state

| Work item | Status and evidence |
| --- | --- |
| Proxmox permission review | Effective permissions read; disk allocation/configuration permissions already present |
| Host storage | Both stores active with substantial reported free space; VM 101 guest-space problem unresolved |
| ISO-upload role | `LabISOUploader` created; assignment and upload test pending |
| Pi collaborator access | Membership in `sudo` verified; full administrative capability, not a package-only exception |
| Pi clock | NTP active and synchronized; local zone changed to `America/Chicago` |
| Router Telnet, FTP, HTTP, API | Disabled; five selected ports still closed after both reboots |
| Switch Telnet | Disabled and reported closed; restart persistence not tested |
| Router review | One admin account, two visible default scripts, no scheduler entries; limited inspection only |
| Pre-upgrade backup | Encrypted file created and downloaded to the MacBook; restore not tested |
| RouterOS / RouterBOOT | Both verified at `7.23.8` |
| Firewall / interface lists | Existing IPv4 and IPv6 configuration present after update; LAN/WAN memberships verified |
| Topology / Tailscale | Existing flat LAN and overlay grants retained; no new public port forwarding |

## 30. Next work

1. **Capture the updated router state.** Create and download a separately named post-upgrade backup, preserve the pre-upgrade copy, and prepare a sanitized text export if it will be documented publicly. Plan a recovery test separately.
2. **Finish the collaborator's Proxmox checks.** Verify the `LabISOUploader` assignment and an actual image upload. Collect the exact storage error and inspect VM 101's disk layout and filesystem before making changes.
3. **Verify VM 101 itself.** Confirm its installed OS, guest updates, Guest Agent, intended remote access, and operations denied outside the collaborator's scope.
4. **Continue management-service review.** Confirm the switch change is saved, test persistence in a maintenance window, and review its remaining HTTP access, firmware, time settings, and certificate handling.
5. **Follow up on the Pi.** Investigate recurring DNS configuration contention if it returns. Complete the planned SSH-key and configuration-backup work; the new sudo membership does not replace those tasks.
6. **Complete recovery and isolation testing.** Back up and restore VM 100, schedule the remaining package updates, and deliberately test the outstanding firewall boundaries before adding intentionally vulnerable guests.

## Command locations at a glance

| Terminal or interface | Commands or operations used there |
| --- | --- |
| MacBook terminal | SSH tunnel, SFTP through the Pi, checking the previously stored router host key |
| Kali on the Pi | `id`, `usermod`, `journalctl`, `tcpdump`, `nmap`, `timedatectl`, SSH to the router |
| RouterOS terminal | `/ip service`, `/user`, `/system script`, `/system scheduler`, backup/update commands, firewall and interface-list inspection |
| Proxmox host shell | `pveum user permissions`, `pvesm status` |
| TP-Link web interface | Disabling the switch's Telnet service |
| Proxmox web interface | Creating the custom ISO-upload role |

## Evidence and limits

This chapter is based on the terminal output and explicit completion reports recorded during the sessions. It does not establish that the whole lab is free of malware, that every service is reachable from every network, or that every planned permission change was completed. No router serial number, actual SSH fingerprint, personal account identifier, live overlay address, password, or binary backup is published here.

The next phase should begin from the outstanding checks above rather than repeating the service-disabling or firmware-update steps.
