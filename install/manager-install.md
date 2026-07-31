# Wazuh Manager Install — All-in-One (Manager + Indexer + Dashboard)

Installed on a dedicated Ubuntu Server VM (`wazuh-manager`), separate from the Active Directory domain.

## 1. Pre-install decisions

**DNS:** Configured to resolve via the LAN firewall/gateway rather than the AD DC. This VM isn't domain-joined, so there's no reason to create a dependency on the DC's uptime just for name resolution — if the DC is powered off (common in this lab's profile-based power-on approach), the manager stays reachable regardless.

**Vulnerability-detection module:** Disabled in `ossec.conf` *before* the first run:
```xml
<vulnerability-detection>
  <enabled>no</enabled>
</vulnerability-detection>
```
This was a lesson learned the hard way on a prior build — the module re-fetches the full CVE feed on an hourly schedule with no cleanup of old feed data, which filled the disk (`/var/ossec/queue/vd*` ballooned to ~32GB). Disabling it up front avoids rebuilding the manager from scratch a second time.

## 2. Installation

Followed the official Wazuh all-in-one quickstart script for v4.9.2 (manager + indexer + dashboard on a single node). Key install-time notes:

- Auto-generated admin credentials were saved directly into a password manager rather than simplified — good credential hygiene is itself worth demonstrating, not just doing.
- Confirmed the dashboard was reachable at `https://<WAZUH-MANAGER-IP>` post-install before moving on to agent enrollment.

## 3. Post-install hardening / cleanup

- Excluded `alerts.json` from the host's logrotate config — Wazuh manages its own archival internally, and an external logrotate policy fighting over the same file causes conflicts.
- Added explicit Proxmox firewall ACCEPT rules for tcp/22, 80, 443 on this VM. Proxmox's per-VM firewall defaults to Input Policy DROP with zero rules, which silently blocks everything until rules are added — learned this the hard way on this same VM before the rebuild, so it's now a proactive step for every new VM in the lab.
- Confirmed `qemu-guest-agent` installed and running, so Proxmox snapshot/shutdown operations behave correctly:
  ```bash
  sudo apt install qemu-guest-agent -y
  sudo systemctl enable --now qemu-guest-agent
  ```
  (The "unit files have no installation config" message on `enable` is expected — there's no `[Install]` section for this unit, but `--now` starts it regardless.)

## 4. Verification

- Confirmed live from the Proxmox host: `qm agent <VMID> ping` → clean response.
- Snapshot taken at this known-good state before beginning agent enrollment.

## Why it worked

Disabling vulnerability-detection pre-install addresses the root cause of the earlier disk-fill incident rather than just monitoring for it after the fact. Keeping the manager DNS-independent from the DC reflects a real operational constraint of this lab (not every VM runs simultaneously) and avoids a fragile dependency chain.

## Prevention

Any future manager rebuild should disable vulnerability-detection *before* first boot, not after — retrofitting this on a running instance still requires clearing the already-bloated queue directory manually.
