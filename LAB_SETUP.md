# Lab setup — Commvault MCP + Recovery Assurance on an 11.46 CommCell (Windows)

Do the steps in order. Each step has a **check**; don't move on until it passes.

## Step 0 — Prerequisites on the machine that will run the MCP

| Need | How to check / get |
|---|---|
| Python 3.11+ | `py -3.11 --version` → install from python.org if missing (tick "Add to PATH") |
| uv | `uv --version` → if missing: `py -3.11 -m pip install uv` |
| Network to Command Center | open `https://<webserver-host>/commandcenter` in a browser on this machine |
| Commvault **access token + refresh token** | Command Center → create an access token for a lab **service account** ([docs](https://documentation.commvault.com/11.38/expert/creating_access_token.html)). A read-only role is enough for these tools. Keep both tokens handy; never paste them in chat |

## Step 1 — Copy and unpack

Copy `Commvault-MCP-Recovery-Assurance.zip` to the machine and extract it, e.g. to `C:\CV-MCP\`.

```powershell
cd C:\CV-MCP\Commvault-MCP-Recovery-Assurance\commvault-mcp-server
uv sync
```
**Check:** finishes without errors and creates `.venv\`. (No internet on this machine? Stop and tell Claude: an offline package can be built.)

## Step 2 — Offline self-test (no CommCell yet)

```powershell
uv run pytest tests_recovery
uv run demo\run_demo.py
```
**Check:** `40 passed`, and the demo prints sections 1–6 ending with "No changes were made to Commvault".

## Step 3 — Official Commvault MCP setup (connects it to your lab)

```powershell
uv run setup.py
```
Answers:
- Instance ID → press Enter (default)
- Metallic setup? → **no**
- `CC_SERVER_URL` → `https://<webserver-host>` (the host you open Command Center on, no `/commandcenter`)
- Transport mode → **stdio**
- Authentication → **traditional token-based**
- Access token / refresh token → paste them (stored in Windows Credential Manager, not in files)
- Secret key → any value you choose

Lab uses a self-signed certificate? Open `.env` in this folder and add a line:
```
SSL_VERIFY=false
```

## Step 4 — Prove the stock MCP talks to the CommCell

```powershell
uv run python -c "from src.tools.client_tools import get_client_list; print(get_client_list())"
```
**Check:** prints `{'clients': [...]}` with your lab's clients. Note the exact name of the client you'll use as the seed.

Common failures: `401` → regenerate tokens and re-run `setup.py`; `SSL` error → `SSL_VERIFY=false`; timeout → wrong `CC_SERVER_URL` or firewall.

## Step 5 — Run the read-only probe

```powershell
uv run lab\lab_probe.py --seed <CLIENT-NAME> --app TradingDemo --group "<recovery group name>"
```
(`--group` only if you have a Cleanroom recovery group.) It uses the MCP's own configuration from Step 3, only makes GET calls, and writes everything to `lab\probe_output\`. It never writes tokens.

**Check:** ends with `Done. N GET calls.`

## Step 6 — Get the results to Claude

- Same PC as `C:\WEB\CV MCP`? Copy the `lab\probe_output` folder to `C:\WEB\CV MCP\lab\probe_output`.
- Different machine? Zip `lab\probe_output` and copy it to `C:\WEB\CV MCP\lab\`.

Then tell Claude **"probe done"**. Claude compares the raw responses with the code, fixes any field mismatches, and gives you an updated build to re-run.

## Step 7 — (after the fixes) Use it from Claude Desktop against the lab

`%APPDATA%\Claude\claude_desktop_config.json`:
```json
{
  "mcpServers": {
    "Commvault-Lab": {
      "command": "C:\\CV-MCP\\Commvault-MCP-Recovery-Assurance\\commvault-mcp-server\\.venv\\Scripts\\python.exe",
      "args": ["C:\\CV-MCP\\Commvault-MCP-Recovery-Assurance\\commvault-mcp-server\\src\\server.py"]
    }
  }
}
```
This loads all stock Commvault MCP tools **plus** the 5 Recovery Assurance tools.
