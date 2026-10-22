# Lab setup: Commvault MCP + Recovery Assurance on an 11.46 CommCell (Windows, from scratch)

For a brand-new lab machine. Do the parts in order. Each step has a **Check**; don't move on until it passes.
All commands are PowerShell. Never paste a Commvault token into a chat; the setup scripts store them in Windows
Credential Manager.

| Part | What | Time |
|---|---|---|
| A | Install Python + uv, unpack, offline self-test | 10 min |
| B | Connect the MCP to the CommCell (read-only) | 5 min |
| C | Write credential (to create recovery groups) | 5 min |
| D | Install Claude Code and register the MCP | 5 min |
| E | Run the demo flow | 10 min |
| F | Clean up | 2 min |

---

## Part A: Install and self-test

### A1. Python 3.11+
```powershell
py --version
```
If it's missing or older than 3.11, either:
```powershell
winget install -e --id Python.Python.3.12
```
or install from python.org (tick **Add python.exe to PATH**). Open a **new** PowerShell window afterwards.
**Check:** `py --version` shows 3.11 or later.

### A2. uv
```powershell
py -m pip install --user uv
```
(or the official installer: `powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"`)
Open a new PowerShell window. **Check:** `uv --version` prints a version. If it's "not recognized", use
`py -m uv` wherever this guide says `uv`.

### A3. Unpack
Copy `Commvault-MCP-Recovery-Assurance.zip` to the machine and extract it to `C:\CV-MCP\`.
```powershell
cd C:\CV-MCP\Commvault-MCP-Recovery-Assurance\commvault-mcp-server
uv sync
```
**Check:** it finishes without errors and creates a `.venv` folder.

### A4. Offline self-test (no CommCell needed)
```powershell
uv run pytest tests_recovery
uv run demo\run_demo.py
```
**Check:** `105 passed`, and the demo ends with "No changes were made to Commvault" / "Read-only plan".

---

## Part B: Connect to the CommCell (read-only)

### B1. Read credential
In Command Center, create an **access token + refresh token** for a lab service account
([how](https://documentation.commvault.com/11.38/expert/creating_access_token.html)). Read access is enough here.

### B2. Run the official setup
```powershell
uv run setup.py
```
Answers:
- Instance ID: press Enter
- Metallic setup: **no**
- `CC_SERVER_URL`: `https://demolabcs1.cv.lab` (the host you open Command Center on, **without** `/commandcenter`)
- Transport mode: **stdio**
- Authentication: **traditional token-based**
- Access token / refresh token: paste them (hidden; stored in Windows Credential Manager)
- Secret key: any value you choose

The lab uses a self-signed certificate. Open `.env` in this folder (`notepad .env`) and add:
```
SSL_VERIFY=false
```

### B3. Prove it talks to the CommCell
```powershell
uv run python -c "from src.tools.client_tools import get_client_list; print(get_client_list())"
```
**Check:** prints `{'clients': [...]}` including dc1, vmwsql1, vmwlnx1, vmwwin1, exchange1.
Failures: `401` → new tokens, re-run `setup.py`; SSL error → `SSL_VERIFY=false` missing; timeout → wrong `CC_SERVER_URL`.

### B4. Read-only preview of the incident flow
```powershell
uv run lab\preview_group.py --defender lab\defender\incident_dc1_identity.json --exposure lab\defender\exposure_dc1_identity.json --base "Sample Application 1" --name "IR-defender-test" --dwell 24
```
**Check:** "playbook(s) ['identity_compromise'] … Defender incident 41077"; members dc1 (P1), vmwsql1 (P2),
vmwwin1 (P3); exchange1 listed as a manual step; vmwlnx1 not included (nothing in the Exposure graph reaches it);
"Threat Scan: off"; a CRITICAL warning that dc1 has Active Directory and File System backups but no VM-level backup;
ends with "READ-ONLY: nothing was created".
(No group called "Sample Application 1" in this lab? Use the name of any existing Cleanroom recovery group: only its
cleanroom target is copied.)

---

## Part C: Write credential (needed to create recovery groups)

1. In Command Center, create an **access token + refresh token** for an account allowed to create Cleanroom recovery
   groups. Commvault access tokens expire; the refresh token lets the tool renew them itself.
2. Store them (both inputs are hidden):
   ```powershell
   uv run lab\setup_write_token.py
   ```
   **Check:** "Stored … (access + refresh token)".
3. Turn writes on: add to `.env`
   ```
   RECOVERY_WRITE_ENABLED=true
   ```
4. Optional but recommended for a clean demo: run a **full backup** of dc1, vmwsql1, vmwwin1 and vmwlnx1, so Commvault
   doesn't report "last full backup copy is more than 45 days old".

---

## Part D: Claude Code (terminal) + the MCP

Claude Desktop installs are blocked on Skillable labs; Claude Code runs in PowerShell and supports MCP.

### D1. Install
```powershell
irm https://claude.ai/install.ps1 | iex
```
Open a new PowerShell window. **Check:** `claude --version` prints a version.
(If the download is blocked by the lab's web filter, ask the lab owner to allow `claude.ai` / `downloads.claude.ai`.)

### D2. Log in
```powershell
claude
```
Follow the browser prompt. Needs a Claude Pro/Max/Team/Enterprise account or an Anthropic Console (API) account.
Prefer a work account or a Console API key with a small spend limit. Type `/exit` when logged in.

### D3. Register the Commvault MCP
```powershell
cd C:\CV-MCP\Commvault-MCP-Recovery-Assurance\commvault-mcp-server
claude mcp add --scope user commvault-lab -- "C:\CV-MCP\Commvault-MCP-Recovery-Assurance\commvault-mcp-server\.venv\Scripts\python.exe" "C:\CV-MCP\Commvault-MCP-Recovery-Assurance\commvault-mcp-server\src\server.py"
claude mcp list
```
**Check:** `commvault-lab ... ✔ Connected`. Inside Claude Code, `/mcp` shows the server with its tools
(stock Commvault tools plus the 11 Recovery Assurance tools).

After any update of the zip: extract it over the folder, then exit and restart `claude` so it reloads the server.

---

## Part E: The demo flow

Start Claude Code **from the repo folder** (so the file paths below resolve):
```powershell
cd C:\CV-MCP\Commvault-MCP-Recovery-Assurance\commvault-mcp-server
claude
```
For a demo, make sure **auto mode is off** (status line; Shift+Tab cycles) so every tool call is shown and approved.

**1. The existing plan is wrong** (read-only)
> Check the Cleanroom recovery group "Sample Application 1": is it in the right recovery order, and is it safe to use after an identity compromise?

Expect: recovers LATEST (unsafe), database and app at the same priority, dc1 missing.

**2. Incident to runbook to Commvault** (one prompt)
> Read lab\defender\incident_dc1_identity.json (a Microsoft Defender incident) and lab\defender\exposure_dc1_identity.json (the Microsoft Exposure Management export). Assume 24 hours of attacker dwell time. Build the recovery runbook and create it in Commvault, using "Sample Application 1" as the base.

Expect, in order:
- playbook derived from the incident (Mimikatz + DCSync → identity compromise), the blast radius along the attack
  path (svc-sql → SQL Admins → vmwwin1 → cached Domain Admin credentials → dc1, exchange1), the runbook;
- a preview, then `create_recovery_group`, then **an approval form asking for your name** (the AI cannot approve);
- result `CREATED` with verified priorities, `autoScale: false`, `default (vmwsql1)` skipped (covered by the VM
  restore), the dc1 CRITICAL warning, and Commvault's own readiness per member.

Then in Command Center → Cleanroom → Recovery groups, open the new `IR-…` group and click Reload: same priorities,
point-in-time recovery point, Threat Scan off.

**3. Drift** (optional)
> Has anything drifted from the approved recovery definition?

If the result is `NOT_APPROVED` without a form: the client didn't show the MCP approval form (fail-closed). For a test
only, set `RECOVERY_APPROVAL_MODE=host` in `.env` to rely on Claude Code's own tool-approval prompt instead (weaker;
tell Commvault engineering which mode you used).
If the result is `WRITE_FAILED` / 401: re-run `uv run lab\setup_write_token.py` with a fresh token pair, then say "retry".

The files in `lab\defender\` are **synthetic** (Microsoft's documented formats with the lab's names). With a real tenant:
export the incident (Defender API `GET /api/incidents/{id}` or Graph `GET /security/incidents/{id}?$expand=alerts`), put
its users/devices into `lab\defender\blast_radius.kql`, run it in Defender → Hunting → Advanced hunting, export the results
as JSON, and use those two files instead.

---

## Part F: Clean up
1. Delete the test groups (`IR-…`) in Command Center.
2. `.env`: set `RECOVERY_WRITE_ENABLED=false`.
3. In Claude Code: `/logout`. If you used a personal account, also sign out other sessions in your Claude account settings.
4. Log out of Command Center and delete any HAR files you captured (they contain session tokens).

---

## Appendix: script-only checks (no Claude needed)

| Purpose | Command |
|---|---|
| Read-only probe (raw API responses to `lab\probe_output\`) | `uv run lab\lab_probe.py --seed dc1 --group "Sample Application 1"` |
| Runbook from a hand-written graph | `uv run lab\run_scenario.py --scenario identity_compromise --graph lab\graphs\lab_identity_dc1.json --group "Sample Application 1"` |
| Runbook from Defender + Exposure Management | `uv run lab\run_scenario.py --defender lab\defender\incident_dc1_identity.json --exposure lab\defender\exposure_dc1_identity.json --dwell 24 --group "Sample Application 1"` |
| Runbook from Cortex + Wiz exports | `uv run lab\run_scenario.py --cortex incident.json --wiz wiz_graph.json --group "Sample Application 1"` |
| Create a group from the console (type the name to confirm) | `uv run lab\create_group.py --defender lab\defender\incident_dc1_identity.json --exposure lab\defender\exposure_dc1_identity.json --dwell 24 --base "Sample Application 1" --name "IR-console-test"` |

Outputs go to `lab\scenario_output\` (runbook.md / runbook.json / group_preview.json / create_result.json).
To send results to Claude for analysis, copy `lab\probe_output` (or `scenario_output`) to `C:\WEB\CV MCP\lab\`.

Offline demo without a CommCell (simulated "TradingDemo" CommCell; no group creation):
`claude mcp add --scope user commvault-sim -- "<repo>\.venv\Scripts\python.exe" "<repo>\demo\sim_server.py"`
