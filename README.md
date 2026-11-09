# SOC Home Lab: VirtualBox, Kali, Windows and Splunk

A VirtualBox lab on Ubuntu for practicing SOC monitoring and investigations. Kali hosts Splunk, and the Windows guest is the planned source of endpoint events. Both guests share the `SOC-Lab` internal network.

**Author:** Moataz Mostafa Ali Ibrahim  
**Lab implementation:** October–November 2026  
**Scope:** Lab infrastructure, endpoint log analysis and SIEM setup

## Verified outcome

- Kali Linux and Windows 10 ran as VirtualBox guests.
- Both guests used the internal network `SOC-Lab`.
- Static addresses placed the guests in the same `/24` subnet.
- Final ICMP tests received four replies from four requests in each direction.
- Reviewed Linux journal and Windows Security events locally.
- Splunk Web opened on Kali, and the Free license group was verified.

## Architecture

```mermaid
flowchart TD
    H[Ubuntu host] --> V[VirtualBox]
    V --> K["Kali Linux + Splunk: 192.168.50.10/24"]
    V --> W["Windows 10: 192.168.50.20/24"]
    K <-->|SOC-Lab internal network| W
```

The host runs the hypervisor. The guest segment uses VirtualBox Internal Network rather than a bridge to the physical LAN. Host-to-guest access is not provided by that internal segment. Isolation described here applies to the documented adapter configuration; it is not a claim of a malware-safe sandbox.

## Environment

| Component | Documented configuration |
|---|---|
| Host | Ubuntu Linux |
| Hypervisor | VirtualBox 7.2.6-dfsg, reported during setup |
| Linux guest | Kali Linux; existing split VMDK image reused in a new VM |
| Windows guest | Windows 10, build 10.0.19045.2965 |
| Guest adapter | Adapter 1, Internal Network, name `SOC-Lab`, cable connected |
| Adapter model | Intel PRO/1000 MT Desktop (82540EM) |
| Subnet | `192.168.50.0/24` |
| Kali | `192.168.50.10/24` on `eth0` |
| Windows | `192.168.50.20`, mask `255.255.255.0` |
| SIEM | Splunk Enterprise 10.6.1.1 on Kali, using the Free license group |
| Gateway | Not required for direct communication within this subnet |

These are implementation details from the exercise, not recommendations to use these exact software versions today.

## Virtual machine and network setup

1. Used the existing Windows VM and created a Kali VM using the main VMDK descriptor, keeping its split disk parts together. Kali was not installed from an ISO in this exercise.
2. Powered off the guests and configured Adapter 1 on each as Internal Network with the identical name `SOC-Lab`.
3. Assigned manual IPv4 settings through NetworkManager on Kali and Ethernet TCP/IPv4 properties on Windows.
4. Checked addresses and tested communication in both directions.

## Network validation

### Address checks

Kali:

```bash
ip -br addr
```

Observed: `eth0` was UP with `192.168.50.10/24`.

Windows:

```cmd
ipconfig
```

Observed: `192.168.50.20` with mask `255.255.255.0` and no default gateway.

### Connectivity tests

Windows to Kali:

```cmd
ping 192.168.50.10
```

Kali to Windows:

```bash
ping -c 4 192.168.50.20
```

| Test | Result recorded in the exercise |
|---|---|
| Windows to Kali | Sent 4, received 4, lost 0; maximum RTT 1 ms |
| Kali to Windows, initial test | Sent 4, received 0; 100% loss |
| Kali to Windows, final retest | Sent 4, received 4; 0% loss; average RTT 0.915 ms |

The initial one-way failure made testing both directions necessary. The final retest demonstrates restored ICMP reachability. The available evidence does not establish the exact corrective change, so no firewall modification is claimed here.

## Network evidence

Screenshots from the original lab exercise are included below. Lab addresses are private IPv4 addresses.

### Internal network configuration

Both adapters use the same internal network name, `SOC-Lab`.

![Kali VirtualBox internal-network settings](images/kali-network.jpg)

![Windows VirtualBox internal-network settings](images/windows-network.jpg)

### Kali IPv4 validation

The final command output shows `eth0` UP with `192.168.50.10/24`.

![Kali IPv4 address validation](images/kali-ipv4.jpg)

### Windows-to-Kali connectivity

Windows shows its IPv4 configuration and four successful ping replies.

![Windows IPv4 configuration and ping to Kali](images/windows-ping.jpg)

### Kali-to-Windows connectivity

The initial test shows 100% loss; the final retest shows four replies and 0% loss.

![Kali ping initial failure and successful retest](images/kali-ping.jpg)

## Endpoint authentication log analysis

I reviewed native logs on both lab guests before adding Splunk. This established what each endpoint records and what a failed authentication event can tell an analyst.

### Linux authentication logs

On Kali, `/var/log/auth.log` was absent, so I searched the systemd journal. I also used `sudo tail -f /var/log/dpkg.log` to practice following a text log; the visible entries were existing package history.

```bash
sudo journalctl --no-pager |
  grep -Ei 'failed password|authentication failure|failed login' |
  tail -n 20
```

The search returned three historical `pam_unix(lightdm:auth)` failures on October 20, at 08:08:18, 08:08:29 and 08:08:38 as displayed. The events span 20 seconds. The `tty=:0` field is consistent with a graphical login; the remote host and remote user fields were blank. These records establish repeated authentication failures but do not identify an account, a remote source or an attack.

### Windows authentication logs

On Windows 10, I opened **Event Viewer → Windows Logs → Security**, entered `4624` and `4625` in the **Event IDs** field, and reviewed event details. I used the General view and XML to distinguish the requesting account from the account targeted by the failed logon.

| Evidence | What I observed |
|---|---|
| Event ID `4624` | A successful logon was present. The visible SYSTEM subject was the requester context; the target account was not visible in the captured view. |
| Event ID `4625`, record `703` | A failed interactive logon (type `2`) was recorded on 20 October 2026 at `09:40:30Z` in the event XML. |
| Failure fields | Both Status and SubStatus were `0xC000006E`. The displayed reason was generic; it did not establish that a password was mistyped. |
| Caller and source | The caller was `msedge.exe`; the source IP and port were not populated. The caller field alone does not establish malicious behavior. |

A single failed interactive logon is insufficient to classify brute force or remote access. The next analytical check would be to compare nearby failures and successes for the same account and examine process-creation evidence if available. The Linux and Windows findings are separate local observations; I did not correlate them as one incident.

## Splunk SIEM setup

I installed Splunk on the Kali guest and verified that Splunk Web opened locally and that the standalone instance was using the Free license group.

1. I downloaded the Splunk Linux `.deb` package and placed it in a read-only VirtualBox shared folder named `splunk-transfer`. The folder did not appear under `/media`, so I mounted it manually inside Kali and copied the installer to my home directory:

   ```bash
   sudo mkdir -p /mnt/splunk-transfer
   sudo mount -t vboxsf -o ro splunk-transfer /mnt/splunk-transfer
   sudo cp /mnt/splunk-transfer/splunk-10.6.1.1-b2ae2962645c-linux-amd64.deb "$HOME/"
   ```

2. I installed the package. Splunk refused the initial root startup without an explicit option, so I used the following commands for this lab run:

   ```bash
   sudo dpkg -i "$HOME/splunk-10.6.1.1-b2ae2962645c-linux-amd64.deb"
   sudo /opt/splunk/bin/splunk start --run-as-root
   ```

3. I opened `http://localhost:8000` in the Kali guest and signed in to verify the Splunk Web home page.

   ![Splunk Web home page after the initial administrator sign-in](images/splunk-web-home.png)

4. Under **Settings → Licensing → Change license group**, I switched from **Trial** to **Free**. I reopened Licensing and verified **Free license group**, with no licensing alerts or violations.

   ![Splunk Licensing page showing the active Free license group](images/splunk-free-license.png)

The screenshots show the initial sign-in and the license verification after switching to Free. The network tests verify ICMP reachability. Windows Security event forwarding and search were verified in the next lab stage; correlation and alerting have not yet been tested.

## Windows event ingestion and basic SPL searches

This stage extends the lab by forwarding real Windows Security events from the Windows 10 guest to Splunk Enterprise on Kali.

### Configure the receiver and verify reachability

In Splunk Web on Kali, I enabled **Settings → Forwarding and receiving → Configure receiving** on TCP port `9997`. From the Windows guest, I verified that the receiver was reachable:

```powershell
Test-NetConnection 192.168.50.10 -Port 9997
```

Observed: `TcpTestSucceeded: True` from Windows `192.168.50.20` to Kali `192.168.50.10`.

### Install and configure Universal Forwarder

I installed Splunk Universal Forwarder **10.6.1.1 (Windows x64)** on Windows 10 and set the receiving indexer to `192.168.50.10:9997`. The Security event log input was enabled in:

```text
C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf
```

Configuration:

```ini
[WinEventLog://Security]
disabled = 0
index = main
```

After saving the input, I restarted the **SplunkForwarder** Windows service so it would load the new configuration. Splunk received events with host `DESKTOP-OAE6UKR`, source/sourcetype `WinEventLog:Security`, and index `main`.

### Basic SPL searches

The time picker is part of the search context. The same event filter returned different results for **Last 24 hours** and **All time** because the two failed-logon events were historical.

| Purpose | SPL | Observed result |
|---|---|---|
| Confirm Security events are arriving | `index="main" sourcetype="WinEventLog:Security"` | 954 events in the selected Last 24 hours range |
| Find failed logons in the recent range | `index="main" sourcetype="WinEventLog:Security" EventCode=4625` | 0 events in Last 24 hours |
| Find failed logons across indexed history | Same filter, with **All time** selected | 2 events dated October 20, 2026 |
| Purpose | SPL | Observed result |
|---|---|---|
| Confirm Security events are arriving | `index="main" sourcetype="WinEventLog:Security"` | 954 events in the selected Last 24 hours range |
| Find failed logons in the recent range | `index="main" sourcetype="WinEventLog:Security" EventCode=4625` | 0 events in Last 24 hours |
| Find failed logons across indexed history | Same filter, with **All time** selected | 2 events dated October 20, 2026 |

To count the matching events, I added an aggregation command after the pipe:

```spl
index="main" sourcetype="WinEventLog:Security" EventCode=4625
| stats count AS failed_login_count
```

Observed: `failed_login_count = 2`.

These counts describe the indexed events only. The two historical EventCode 4625 records do not by themselves establish brute force or malicious activity. The TryHackMe **Introduction to SIEM** room was completed at 100%.

## Skills demonstrated

| Skill | Evidence from this project |
|---|---|
| Virtualization | Ran Kali and Windows guests on an Ubuntu host |
| Virtual networking | Configured matching internal-network adapters |
| IPv4 addressing | Assigned and checked static addresses in one subnet |
| Cross-platform administration | Used Linux and Windows networking tools |
| Troubleshooting | Compared an initial failed test with a successful retest |
| Authentication analysis | Filtered native Linux and Windows logs and assessed the limits of failed-logon evidence |
| SIEM setup and ingestion | Installed Splunk on Kali, verified the Free license, received Windows Security events through a Universal Forwarder, and searched them with SPL |
| Technical reporting | Recorded commands, outcomes and limits of the findings |

## Lessons learned

- Identical internal-network names matter: guest adapters must share the intended virtual segment.
- Direct same-subnet IP communication does not require a default gateway.
- Testing one direction alone can hide an asymmetric reachability problem.
- A successful ping is a specific observation, not proof that all services work.

## Next steps for the lab

- Investigate the two historical failed-logon events in context and document what additional evidence would be needed before classifying activity.
- Build and test a basic alert or dashboard from the ingested Windows Security data.

## References

- Oracle VirtualBox 7.2 User Guide, Virtual Networking: https://docs.oracle.com/en/virtualization/virtualbox/7.2/user/networkingdetails.html
- Kali Linux documentation: https://www.kali.org/docs/

## Attribution

Prepared from my SOC internship lab work, with AI assistance organizing the documentation.
