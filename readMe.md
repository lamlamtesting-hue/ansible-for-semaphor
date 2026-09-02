# Register Icinga 2 Windows Agent

`register-agent.yaml` registers one or more Windows hosts in Icinga Director, then installs and joins the Icinga 2 agent over WinRM. It is meant to run from Semaphore.

There are two plays:

1. **localhost** — Director templates and one host object per `cn`, deploy, tickets, `add_host`
2. **icinga_agents** — WinRM install, PKI, `node setup` using that `cn`

Each JSON agent has **`host_ip`**, **`cn`**, and **`cluster_zone`** (Director Cluster Zone). Zones in this environment: **`master`**, **`LC-icinga-slave01`**, **`lc-satellites`**. **`icinga_master_cn`** is the parent Endpoint CN.

```
Semaphore JSON: host_ip + cn per VM, icinga_master_cn for the master
        │
        ▼
Director object named <cn>  ──deploy──►  Icinga 2 master
        │
        │  ticket for <cn>
        ▼
WinRM to host_ip  ──  node setup --cn <cn> --endpoint <icinga_master_cn>
```

## What you need

| Requirement | Why |
|---|---|
| Icinga Web + Director reachable at `director_url` | Create templates/host and fetch the ticket |
| `icinga_web_user` / `icinga_web_pass` | Director basic auth (Semaphore extra vars) |
| Windows VM with WinRM on **5985** | Play 2 login |
| `icinga_agent_hosts` (`host_ip` + `cn`) and WinRM user/pass | Target address and Icinga CN per VM |
| Semaphore runner: `pywinrm` + `ansible.windows` | WinRM modules |
| Master API **5665** reachable from the VM | `pki save-cert` and `node setup` |

## Extra vars (Semaphore)

Use **Variable Groups → Extra variables → JSON** (not TABLE). Semaphore passes that object as Ansible extra vars. Playbook defaults apply for any key you omit.

Put passwords in the **Secrets** tab of the same group (`icinga_web_pass`, `agent_winrm_pass`) so they are not stored in the JSON editor.

**One VM:**

```json
{
  "host_ip": "10.1.5.10",
  "cn": "win-agent-01",
  "icinga_master_cn": "icinga2",
  "cluster_zone": "master",
  "icinga_web_user": "admin",
  "agent_winrm_user": "Administrator"
}
```

**Several VMs** — each item needs `host_ip`, `cn`, and `cluster_zone` (`master`, `LC-icinga-slave01`, or `lc-satellites`). Default `cluster_zone` / `icinga_master_cn` apply if omitted on an item.

```json
{
  "director_url": "http://10.1.5.151:8080",
  "icinga_master_cn": "icinga2",
  "cluster_zone": "master",
  "icinga_web_user": "admin",
  "agent_winrm_user": "Administrator",
  "icinga_agent_hosts": [
    { "host_ip": "10.1.5.10", "cn": "win-agent-01", "cluster_zone": "master" },
    { "host_ip": "10.1.5.11", "cn": "win-agent-02", "cluster_zone": "LC-icinga-slave01", "icinga_master_cn": "LC-icinga-slave01" },
    { "host_ip": "10.1.5.12", "cn": "win-agent-03", "cluster_zone": "lc-satellites" }
  ]
}
```

Duplicate `host_ip` or `cn` values fail the play. `host_name` is accepted as an alias for `cn`.

Optional keys: `director_url`, `icinga_master_host`, `generic_host_template_name`, `icinga_agent_template_name`, `agent_winrm_port`, `agent_winrm_scheme`.

## Play 1 — Register in Director (`localhost`)

Director create vs update uses two URLs:

- **Create:** `POST /director/host` → 201
- **Modify:** `POST /director/host?name=...` → 200 or 304

Create is treated as success if the object is new, or if Director returns 500 with `"Trying to recreate"` (already exists).

### 1. Base template (`generic-host-lam`)

Creates a host **template** with check settings only: `hostalive`, 3 attempts, 1m / 30s intervals. Non-agent hosts can reuse this without becoming agents.

### 2. Agent template (`icinga-agent-lam`)

Creates a second template that **imports** the generic one and sets agent flags (same as the Director UI):

| Var | Default | Meaning |
|---|---|---|
| `cluster_zone` | `master` | Cluster zone |
| `has_agent` | `y` | This host is an Icinga 2 agent |
| `master_should_connect` | `y` | Master opens the connection to the agent |
| `accept_config` | `y` | Agent accepts config from the parent |

Agent flags belong on a **template**. Director uses that to generate the agent Zone and Endpoint.

### 3. Always write agent flags

Create is a no-op if the template already exists, so the next task **updates** `icinga-agent-lam` with the same flags. Re-runs stay consistent.

### 4. Host objects (`cn` per IP)

Each JSON item is one Director host named `cn` with address `host_ip`. GET `/director/host?name=<cn>`:

- **404** → create object: address, import agent template, `vars.os: Windows`
- **200** → update existing host (address, import, agent flags)

Inheritance (shared templates, many hosts):

```
generic-host-lam          check command / intervals     ← once
        ↑
icinga-agent-lam          has_agent, zone, connect     ← once
        ↑
   win-agent-01, …        one Director object per cn
```

### 5. Deploy

`POST /director/config/deploy` runs **once** after all hosts are created/updated, then tickets are fetched.

### 6. Ticket

`GET /director/host/ticket?name=<cn>` for each host. Play 2 uses that ticket in `icinga2 node setup`.

### 7. In-memory inventory

`add_host` uses `cn` as the inventory name and `host_ip` as `ansible_host`. It copies `icinga_host_name` (the cn), the ticket, and `icinga_master_cn` (parent) onto the host. Per-host `agent_winrm_user` / `agent_winrm_pass` / `agent_winrm_port` on a list item override the shared defaults.

## Play 2 — Install agent on Windows (`icinga_agents`)

No `become` (that is Linux sudo). Modules are `ansible.windows.*`.

### 1. Install Icinga 2

Download `Icinga2-v2.16.5-x86_64.msi` from packages.icinga.com and install silently (`/qn /norestart`). Return code 3010 (reboot required) is accepted.

### 2. Trusted parent cert

If `trusted-parent.crt` is missing, run:

`icinga2.exe pki save-cert --host <master> --port 5665`

The VM must reach the master on **5665**, not 8080.

### 3. Node setup (first install only)

`win_stat` checks:

`C:\ProgramData\icinga2\var\lib\icinga2\certs\<cn>.crt`

- **Missing** → first install: `icinga2.exe node setup` with ticket, CN, zone, parent master, `--accept-commands`, `--accept-config`, `--disable-confd`
- **Present** → skip (already enrolled)

`--global_zones director-global` is **not** passed. The Windows MSI already defines `director-global`; passing it again fails with `The global zone 'director-global' is already specified`.

If a previous run wrote the `.crt` but setup did not finish, delete that cert on the VM so this task runs again.

### 4. Service

Enable and start the `icinga2` Windows service. If node setup ran, a handler restarts it so API/zone changes take effect.

## Idempotency

| Object | Re-run behavior |
|---|---|
| Director templates | Create ignored if present; flags updated |
| Director host | Create or update |
| MSI | `win_package` present |
| Parent cert | Skipped if `trusted-parent.crt` exists |
| Agent cert / node setup | Skipped if `<cn>.crt` exists |
| Service | Started / auto |

## Connection vs check ports

| Port | Used for |
|---|---|
| **8080** | Director / Icinga Web (from Semaphore) |
| **5985** | WinRM HTTP (from Semaphore to the VM) |
| **5665** | Icinga 2 API / PKI (from the VM to the master) |

Do not set `icinga_master_port` to 8080. That is Web only.
