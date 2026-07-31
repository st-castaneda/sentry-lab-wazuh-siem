# Windows Agent Enrollment (Server 2025 AD DC + Windows 11)

Applies to both the Windows Server 2025 domain controller and the Windows 11 workstation — identical process on both.

## Version pinning

Agent version is pinned to `4.9.2-1` to match the manager exactly. Wazuh strictly rejects any agent running a version newer than its manager, so installs always reference the specific package rather than pulling `latest`.

## Install steps

```powershell
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.9.2-1.msi -OutFile wazuh-agent.msi
msiexec.exe /i wazuh-agent.msi /q WAZUH_MANAGER="<WAZUH-MANAGER-IP>"
NET START WazuhSvc
```

## Verification

Check the agent log to confirm it registered and connected successfully:
```powershell
Get-Content "C:\Program Files (x86)\ossec-agent\ossec.log" -Tail 30
```
Look for:
```
wazuh-agent: INFO: (4102): Connected to the server ([<WAZUH-MANAGER-IP>]:1514/tcp).
```

Confirm on the manager side too, via the dashboard **Endpoints** view or the CLI:
```bash
sudo /var/ossec/bin/manage_agents
```
→ `L` to list registered agents and confirm the new entry appears with the correct name, IP, and version.

## Notes / non-issues encountered

- Windows Server correctly **skipped** a CIS Server-2022 SCA policy — expected behavior, since the SCA module detected the actual OS is Server 2025 and version-gates policies accordingly. Not a failure.
- Win11 automatically ran a full **CIS Windows 11 Enterprise Benchmark** scan as soon as the agent connected — no separate action needed to trigger it.
- CPU briefly spiked to ~100% on Win11 mid-enrollment; traced to Windows Defender plus a pending security update downloading in the background, unrelated to the agent install. Resolved on its own.

## Re-enrollment after a hostname change

If a Windows agent's hostname changes (e.g. renaming the machine), the existing agent **does not** automatically re-register under the new name — it keeps reconnecting using its existing `client.keys`, which ties it to the original registration. To force a clean re-registration under a new hostname:
```powershell
Stop-Service WazuhSvc
Remove-Item "C:\Program Files (x86)\ossec-agent\client.keys" -Force
Start-Service WazuhSvc
```
The stale entry under the old name must also be removed manager-side (`manage_agents` → `R`) or the new registration will fail with a `Duplicate agent name` error, since Wazuh does not allow two agents to share a hostname.

**Important:** confirm the OS-level hostname change actually completed (and survived a reboot) before doing any of the above — a rename command reporting success does not guarantee the change took effect until the machine has been rebooted and `hostname` reflects the new name.
