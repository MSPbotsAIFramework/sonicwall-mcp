# sonicwall-mcp

MCP server for **SonicWall** firewalls — device inventory, security posture,
config reads, and attack analytics — over the
[Model Context Protocol](https://modelcontextprotocol.io/) (Streamable
HTTP/SSE transport), built against SonicWall's **NSM SaaS cloud API**.

Built for [PRD-17726](https://app.clickup.com/t/2280862/PRD-17726) after two
other candidates were ruled out:

- The community repo [`gensecaihq/Sonicwall-MCP-Server`](https://github.com/gensecaihq/Sonicwall-MCP-Server)
  doesn't work at all — a real `docker build && docker run` reproduced a
  container that can't start (wrong entrypoint file), and its API client
  fabricates fake log/threat data on a connection failure instead of
  erroring. Not fixable with a small patch; effectively a full rewrite.
- SonicWall's own blog announced an official open-source MCP at
  `github.com/sonicwall/sonicwall-mcp` — that URL 404s right now (checked via
  the GitHub API, a direct HTTP request, and the Wayback Machine, which has
  no snapshot of it either). The announcement is real; the repo it links to
  currently isn't reachable.

<div align="center">

## MSPbots Connector

**The MCP Gateway built for MSPs: visible, controlled, and your data stays where it belongs.**

[**Get started at connect.mspbots.ai →**](https://connect.mspbots.ai)

</div>

---

MSPbots Connector gives your AI clients one MCP URL to reach your entire MSP stack (RMM, PSA, documentation, security, finance), and gives you control over who can call which tool and a record of everything that was called.

**Hosted service: [connect.mspbots.ai](https://connect.mspbots.ai)** · Free to use.

### Why Connector

![MSPbots Connector](https://stgoss.mspbots.ai/files/mcp/mspbots-mcp.png)

| | |
|---|---|
| **Visible** | Every tool call is logged: who, which vendor, when. Usage dashboard, CSV export, SIEM streaming. |
| **Controlled** | Role-based, per-tool allowlist. Deny by default. |
| **Protected** | Credentials encrypted per record. AI clients never see your vendor passwords or tokens. |
| **MSP-native** | Built for the MSP stack only: RMM, PSA, documentation, security, finance. |

### Quick start

1. **Sign up** at [connect.mspbots.ai](https://connect.mspbots.ai).
2. **Connect your vendors.** Pick from 150+ connectors and authorize each one on the vendor's own page, or bring your own API keys. No extra vendor accounts needed.
3. **Set access.** Create roles and decide which tools each role may call. Nothing is allowed until you allow it.
4. **Add your MCP URL to your AI client.** Copy the URL from your Connector dashboard into Claude, ChatGPT, Cursor, VS Code, Gemini, Codex, or any MCP-capable client.
5. **Ask your AI to do real work**, and watch every call appear in the audit log.

### Features

**Unified gateway**
- One MCP URL for all your AI clients
- 150+ connectors across the MSP stack
- Custom MCP servers supported, free
- Authorization happens on each vendor's own page; the AI never sees passwords or tokens
- Bring your own credentials: use the vendor API keys you already have
- Multiple accounts per app; connect once, reuse across AI clients
- Tool search: finds the right action for the task
- One sign-in; vendor credentials are injected behind the scenes and OAuth tokens refresh automatically

**Visibility**
- Audit log for every tool call (who, which vendor, when)
- Usage dashboard: call volume, top vendors, error rate, per-user activity
- Export audit logs to CSV, or stream to Loki, Graylog, or LogScale (SIEM)
- Optional recording of tool arguments and result summaries (off by default, enabled per organization)

**Control**
- Per-role, per-tool allowlist, deny by default
- owner / admin / member roles
- SCIM provisioning with Entra ID, Okta, JumpCloud, and Google Workspace

**Security**
- Credentials encrypted one record at a time (AES-256-GCM, per-record key)
- AI clients authenticate with OAuth 2.1 + PKCE
- Vendor responses are scanned for prompt-injection patterns and returned as structured data. This is one layer of defense, not a guarantee against every attack.
- Static egress IP for customer firewall and API allowlisting

### Pricing

**Free.**

Links: [connect.mspbots.ai](https://connect.mspbots.ai) · [mspbots.ai](https://mspbots.ai)

---

## Why NSM SaaS instead of the local SonicOS on-box API

SonicOS also has its own official REST API (`POST /api/sonicos/auth`, RFC
2617 Basic Auth) — but that API lives on the firewall's own management IP,
normally on the customer's LAN behind NAT, same reachability problem as
`unifi-network-mcp`. **NSM SaaS is different: it's SonicWall's own
cloud-hosted fleet-management platform**, and a firewall managed through it
maintains its own outbound connection to NSM — this server never needs to
reach the customer's network directly, only SonicWall's cloud
(`mysonicwall.com` / `api.mysonicwall.com` / `nsm-<region>.sonicwall.com`).

**The one precondition this doesn't remove**: the firewall has to actually
be onboarded to an NSM SaaS tenant. A SonicWall a customer owns but never
registered with NSM is invisible to this API — confirm with the customer
that they're an NSM SaaS subscriber before assuming this will work.

## Tools

Authorization requires the `X-Sonicwall-Msw-Api-Key` request header (see the authorization notes below).

| Tool | Description | Parameters |
|---|---|---|
| `sonicwall_list_tenants` | List the tenants visible to this API Key (only needed for disambiguation when the account has multiple tenants) | None |
| `sonicwall_list_devices` | List all firewalls managed by this NSM tenant | None |
| `sonicwall_list_vulnerable_devices` | List devices with known vulnerabilities (tenant-level, no serial needed) | None |
| `sonicwall_get_device_status` | Get the real-time system status of a firewall | `serial` (required) |
| `sonicwall_get_security_services_status` | Get the status of a firewall's security services (Gateway AV/IPS, etc.) | `serial` (required) |
| `sonicwall_list_firewall_zones` | List the security zones configured on a firewall | `serial` (required) |
| `sonicwall_get_system_logs` | Get the NSM-collected logs of a firewall | `serial` (required), `limit` (optional) |
| `sonicwall_get_attack_analytics` | Get attack/threat analytics by dimension (initiator/target/port/country/action taken) | `serial` (required), `dimension` (required), `limit` (optional) |

All tools are read-only; there are no configuration-change or reboot tools.

## Quick Start

### Docker (recommended)

```bash
docker compose up --build
```

The server starts on `http://localhost:8080`.

### Local (uv)

```bash
uv sync
python -m sonicwall_mcp
```

## Health Check

```bash
curl http://localhost:8080/health
# {"status": "ok"}
```

No credentials are required for the health endpoint.

## Authentication

Every request to `/mcp` must include the following HTTP headers (provided by the
MCP caller/gateway):

| Header | Type | Required | Description |
|---|---|---|---|
| `X-Sonicwall-Msw-Api-Key` | string | Required | The customer's long-lived MySonicWall API Key (mysonicwall.com → My Workspace → User Groups → User list → Generate My API Key); it is recommended to generate it with a dedicated account whose NSM role is ReadOnly. |
| `X-Sonicwall-Tenant-Name` | string | Optional | Only needed when this API Key can see multiple tenants (e.g. an MSP account managing multiple customers); matched by name substring (case-insensitive); may be omitted when the account has only one tenant. |
| `X-Sonicwall-Nsm-Region` | string | Optional, default `uswest` | NSM SaaS is multi-region (corresponds to `nsm-<region>.sonicwall.com`; European customers use `eucentral`); fill in the region where the customer's NSM tenant actually resides. |

Missing the required header returns `401 Unauthorized`.

**Authentication mechanism**: after receiving the API Key, this service itself runs through the 3-step token exchange flow described in the official SonicWall documentation (`GET mysonicwall.com/api/hgms/get-cloud-tenants` → `POST api.mysonicwall.com/api/generate-cscaccesscode` → `POST nsm-<region>.sonicwall.com/api/manager/auth/sso`), then uses the obtained NSM bearer token to call the actual business endpoints - the gateway does not issue the token on its behalf. **Every call re-runs these 3 steps and never caches** (even though the official docs say the NSM token is valid for about 8 hours) - in multi-tenant scenarios, caching tokens can easily mix up accounts between concurrent requests, which is consistent with how the other vendors in the fleet (`datto-rmm-mcp`/`easydmarc-mcp`/`oitvoip-mcp`/`ingrammicro-mcp`) handle it.

## Environment Variables

| Variable | Default | Description |
|---|---|---|
| `MCP_HTTP_PORT` | `8080` | Listening port |
| `MCP_HTTP_HOST` | `0.0.0.0` | Listening host |

## MCP Endpoint

```
POST http://localhost:8080/mcp
```

Connect your MCP client with:
- Transport: `http` (Streamable HTTP / SSE)
- Headers: `X-Sonicwall-Msw-Api-Key` (required), `X-Sonicwall-Tenant-Name` / `X-Sonicwall-Nsm-Region` (optional)

## Test Example

```bash
curl -X POST http://localhost:8080/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "X-Sonicwall-Msw-Api-Key: <customer-mysonicwall-api-key>" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": { "name": "sonicwall_list_tenants", "arguments": {} }
  }'
```

> ⚠️ This is a public repository; do not write real customer API Keys into any committed file.
> The placeholders above are for illustration only.

## Known Gaps

- **What was actually verified live, and how.** No real MySonicWall account
  was available, so nothing here has succeeded end-to-end against real
  data — but running this server for real with a **dummy** API key did
  confirm something concrete, not assumed:
  - `GET https://mysonicwall.com/api/hgms/get-cloud-tenants` is real and
    reachable — but a bad key doesn't get a clean JSON 401. It gets a
    `302` redirect to an HTML login page (`login.aspx?ReturnUrl=...` →
    `/muir/login`, confirmed in this server's own httpx request log). This
    server explicitly detects that redirect and turns it into a clear
    `unauthorized` error — the first version of this code instead followed
    the redirect, parsed the login page as empty JSON, and silently
    returned `{}` as if the tenant list were genuinely empty. Fixed before
    this was pushed, but it's a sign this endpoint's failure modes are not
    fully mapped; a *real* invalid-but-well-formed key might behave
    differently than the placeholder string used here.
  - This means `api.mysonicwall.com` (official docs' stated host for this
    same call) was never itself tried — this build uses the bare
    `mysonicwall.com` host instead, per the next point.
- **A real documentation/implementation conflict, not resolved.**
  SonicWall's own official docs (`msw-api_guide`) state
  `get-cloud-tenants` lives at `api.mysonicwall.com`. A real (if young)
  working implementation — [`GTalksTech/sonicwall-mcp`](https://github.com/GTalksTech/sonicwall-mcp),
  created 2026-09-01 — carries an explicit code comment that the bare
  `mysonicwall.com` host is what actually works, reading like a real,
  empirically-hit bug fix rather than a typo. This build follows the
  working implementation's choice (bare host), since "docs say X, real
  code found Y" is exactly the kind of thing worth trusting the tested
  code over — but **neither side of this has been independently confirmed
  by us against a real account**. If every tool starts failing at the
  tenant-resolution step, this is the first thing to check.
- **Every response-shape detail beyond the top-level call chain comes from
  reading that same third-party implementation's source directly** (both
  `server.py` and `nsm_auth.py`), not from SonicWall's own docs, because
  the docs describe the call chain (host, method) but not field names like
  `productGroupID` (the tenant ID field), `cloudServices` (not `services`)
  holding a `serviceName: "NGMGMT"` entry (not matched by the "Network
  Security Manager" display name alone), or the bearer token arriving
  nested at `response["status"]["info"][0]["message"]` instead of a
  top-level `access_token` field. None of this is guessable from the
  official docs alone, and none of it has been independently verified by
  us — it's only as trustworthy as that one implementation's own testing.
- **`X-Device-ID` header for per-device scoping is unverified.** The
  reference implementation uses this header to proxy a manager-plane call
  through to one specific firewall by serial number; never exercised here
  against a real device.
- **`SYSEVENT_REQUIRED_PARAMS` (`limit=50&sort=time`) is a real but
  narrow finding.** The reference implementation's own comments say
  `/systemevent/*` endpoints return HTTP 422 naming whichever required
  parameter is missing, and that `limit` is typed as a boolean gate in
  SonicWall's own swagger rather than an actual row cap (one sample
  returned 3,140 rows despite `limit=50`) — meaning `sonicwall_get_system_logs`
  and `sonicwall_get_attack_analytics`'s `limit` parameter may not do what
  its name implies. Passed through as-documented rather than guessed at,
  but not independently confirmed.
- **MVP-scoped by explicit decision, not a full port.** The reference
  implementation exposes on the order of 50 distinct NSM endpoints
  (interfaces, NAT, VPN policies, SSL-VPN, wireless/SonicPoint status,
  HA/failover, DNS, NTP, local users, config-audit diff/history, Analyzer
  NG reports). This build covers device/tenant discovery, one live status
  read, one config read (zones), and attack analytics/logs — the subset
  that maps most directly to PRD-17726's stated ask. Any of the rest can
  be added on request, but each one should be treated with the same
  "borrowed, not verified" caution as everything above.
- **No write/config-push endpoints are wrapped, and none are planned** —
  this service is read-only by design, matching the reference
  implementation's own "ideally a ReadOnly NSM role" recommendation.
