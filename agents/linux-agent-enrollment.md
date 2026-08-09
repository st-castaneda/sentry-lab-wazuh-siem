# Linux Agent Enrollment (Rocky Linux 9.8)

## Version pinning

Same rule as the Windows agents: pin to `wazuh-agent-4.9.2-1`, not the repo's default/latest package. The manager rejects any agent version newer than itself, so this must be specified explicitly on every install.

## Install steps

```bash
curl -o wazuh-agent.rpm https://packages.wazuh.com/4.x/yum/wazuh-agent-4.9.2-1.x86_64.rpm
WAZUH_MANAGER="<WAZUH-MANAGER-IP>" rpm -ihv wazuh-agent.rpm
systemctl daemon-reload
systemctl enable wazuh-agent
systemctl start wazuh-agent
```

## Verification

```bash
systemctl status wazuh-agent
tail -f /var/ossec/logs/ossec.log
```
Confirm a line similar to:
```
wazuh-agent: INFO: (4102): Connected to the server ([<WAZUH-MANAGER-IP>]:1514/tcp).
```

On the manager side:
```bash
sudo /var/ossec/bin/manage_agents
```
→ `L` to confirm the new agent entry (ID, name, IP, version).

## Notes

- Rocky was later rebuilt headless (2GB RAM, minimal ISO install) as part of a broader lab decision to keep this VM lean — the agent reinstall after rebuild followed the same steps above with a fresh enrollment.
- `Disconnected` status in the dashboard for this agent is expected and normal whenever the Rocky VM itself is powered off — it reflects live connectivity, not a broken enrollment. No action needed when this VM isn't part of the day's active profile.
- Non-AD-joined Linux hosts in this lab (Rocky, the Wazuh manager) resolve DNS through the LAN firewall/gateway directly rather than the AD DC, avoiding an unnecessary dependency on DC uptime for basic name resolution.
