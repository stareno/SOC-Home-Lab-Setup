# SOC Home Lab: VirtualBox, Kali, Windows and Splunk

A VirtualBox lab on Ubuntu for practicing SOC monitoring and investigations. Kali hosts Splunk, and the Windows guest is the planned source of endpoint events. Both guests share the `SOC-Lab` internal network.

**Author:** Moataz Mostafa Ali Ibrahim  
**Lab implementation:** October–November 2026  
**Scope:** Lab infrastructure, connectivity validation and SIEM setup

## Verified outcome

- Kali Linux and Windows 10 ran as VirtualBox guests.
- Both guests used the internal network `SOC-Lab`.
- Static addresses placed the guests in the same `/24` subnet.
- Final ICMP tests received four replies from four requests in each direction.
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

The screenshots show the initial sign-in and the license verification after switching to Free. The network tests verify ICMP reachability. Windows log forwarding, event ingestion, correlation and alerting have yet to be tested.

## Skills demonstrated

| Skill | Evidence from this project |
|---|---|
| Virtualization | Ran Kali and Windows guests on an Ubuntu host |
| Virtual networking | Configured matching internal-network adapters |
| IPv4 addressing | Assigned and checked static addresses in one subnet |
| Cross-platform administration | Used Linux and Windows networking tools |
| Troubleshooting | Compared an initial failed test with a successful retest |
| SIEM setup | Installed Splunk on Kali and verified local web access and the Free license |
| Technical reporting | Recorded commands, outcomes and limits of the findings |

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

## Lessons learned

- Identical internal-network names matter: guest adapters must share the intended virtual segment.
- Direct same-subnet IP communication does not require a default gateway.
- Testing one direction alone can hide an asymmetric reachability problem.
- A successful ping is a specific observation, not proof that all services work.

## Next steps for the lab

- Configure Windows event collection into Splunk and verify that test events are searchable.
- Use the collected authentication events to investigate failed logons.

## References

- Oracle VirtualBox 7.2 User Guide, Virtual Networking: https://docs.oracle.com/en/virtualization/virtualbox/7.2/user/networkingdetails.html
- Kali Linux documentation: https://www.kali.org/docs/

## Attribution

Prepared from my SOC internship lab work, with AI assistance organizing the documentation.
