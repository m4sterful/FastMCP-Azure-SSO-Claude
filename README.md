# FastMCP-Azure-SSO-Claude
Below are key details on creating a secure SSO authentication with FastMCP for Claude. 

When building an MCP interface with claude these instructions can be provided to the AI to allow creation of a functional interface.

# FastMCP + Entra ID SSO: Implementation Notes

Key discoveries and non-obvious configuration requirements from building a FastMCP
server with Microsoft Entra ID SSO, deployed on cPanel/Passenger, consumed by Claude.ai.

---

## 1. Entra Access Token Version Must Be v2 (Manifest Edit Required)

**The single most impactful issue.** By default, Entra ID issues v1 access tokens.
FastMCP's `AzureProvider` validates tokens against the v2 JWKS endpoint
(`https://login.microsoftonline.com/{tenant}/v2.0/keys`). v1 and v2 tokens differ
in audience format and claim structure — v1 uses a legacy resource URI as the audience;
v2 uses the Application ID URI (`api://dandh-catalog`).

The fix is a one-line change to the **App Manifest**:

```json
"accessTokenAcceptedVersion": 2
```

**Where:** Azure Portal → Entra ID → App registrations → your app → **Manifest** →
find `accessTokenAcceptedVersion` (defaults to `null`, which means v1) → set to `2` → Save.

Without this, the OAuth popup completes and Claude shows the connector as green, but every
tool call fails with a token validation error. The symptom is indistinguishable from a
network or config issue, which makes it very hard to diagnose.

---

## 2. REQUIRED_SCOPES: Short Name Only, Not the Full api:// URI

In the config or .env, `REQUIRED_SCOPES` takes the scope's **short name only**:

```python
REQUIRED_SCOPES = ["Catalog.Read"]
```

Not the full Application ID URI form:

```python
# Wrong — causes scope validation failures on every tool call
REQUIRED_SCOPES = ["api://<your-API-GUID>/Catalog.Read"]
```

`AzureProvider` constructs the full scope URI internally. Passing the full string causes
scope validation to fail because the provider matches against the short name, not the
fully-qualified form.

---

## 3. allowed_client_redirect_uris: Claude's Exact Callback Must Be Listed

```python
auth_provider = AzureProvider(
    ...
    allowed_client_redirect_uris=[
        "https://claude.ai/api/mcp/auth_callback",
        "http://localhost:*",
    ],
    redirect_path="/auth/callback",
)
```

`https://claude.ai/api/mcp/auth_callback` is the URI Claude uses after the OAuth popup
completes. It must appear exactly as shown.

This is a **different URI** from the one registered in Entra:

- **Entra redirect URI** (`BASE_URL + "/auth/callback"`) — where Entra sends the
  authorization code after user sign-in. Registered under Authentication in the app reg.
- **`allowed_client_redirect_uris`** — where FastMCP's auth proxy sends the final token
  response back to Claude.ai. Configured on `AzureProvider` in code.

Both must be correct and consistent. `redirect_path` must match the Entra-registered URI
suffix exactly — even a trailing slash difference causes a redirect_uri mismatch error.

`http://localhost:*` is required for local development and testing.

---

## 4. Application ID URI: Set It Explicitly Under "Expose an API"

The Application ID URI (`api://<GUID>/Operation.read`) must be set explicitly under
**Expose an API** in the app registration. This is not set automatically on new
registrations and is required before you can define any scopes.

The format `api://<name>` is the standard for custom APIs. This URI becomes the audience
claim in v2 access tokens and is what `AzureProvider` validates `aud` against.

---

## 5. OAuth Discovery Is Automatic — Claude Finds It via Well-Known

FastMCP automatically serves these endpoints — no manual implementation required:

| Endpoint | Purpose |
|----------|---------|
| `/.well-known/oauth-protected-resource` | Resource metadata; first thing Claude fetches on connector add |
| `/.well-known/oauth-authorization-server` | Authorization server metadata |
| `/register` | Dynamic client registration (Claude registers itself here) |
| `{redirect_path}` | Token exchange callback (default: `/auth/callback`) |

When adding the connector in Claude, Claude hits `/.well-known/oauth-protected-resource`
immediately. If this endpoint isn't reachable, the connector fails silently with no useful
error message. Verify it manually before connecting:

```bash
curl https://yourdomain/mcp/.well-known/oauth-protected-resource
```

---

## 6. Proxying and IP Restriction: Use RewriteCond

Consider a subdomain of yourapp.company.com and an internal Phusion Passenger app on 127.0.0.1:8000. You can use the .htaccess details below to create a reverse proxy to your server for only Anthropic's IPs while not blocking letsencrypt requests:

```apache
RewriteEngine On

# IP allowlist — deny anything not matching Anthropic's IPs (currently 160.79.104.0/21)
RewriteCond %{REMOTE_ADDR} !^160\.79\.1(0[4-9]|1[01])\. [NC]
RewriteRule ^ - [F,L]

# Proxy to app (exclude acme-challenge for SSL renewal)
RewriteCond %{REQUEST_URI} !^/\.well-known/acme-challenge/
RewriteRule ^(.*)$ http://127.0.0.1:8000/$1 [P,L]
```
---

## 7. cPanel / Passenger Deployment

`passenger_wsgi.py` exposes the app to Passenger:

```python
import sys, os
sys.path.insert(0, os.path.dirname(__file__))

from server import mcp
application = mcp.http_app()
```

`mcp.http_app()` returns the underlying ASGI app. Passenger detects it as async and
handles it correctly — no additional WSGI adapter or wrapper needed.

The `sys.path.insert` is required so that `import config` and `import db` resolve
correctly from within the Passenger process, which doesn't inherit the working directory
the same way a direct `python server.py` invocation does.

**Restart after changes:** Passenger caches the application process. After any edit to
`server.py` or `config.py`, restart via the cPanel Python App UI or:

```bash
touch /home/<user>/<approot>/tmp/restart.txt
```

---

## 8. Transport: streamable-http Is Required for HTTP Deployment

```python
mcp.run(transport="streamable-http", host=config.MCP_SERVER_HOST, port=config.MCP_SERVER_PORT)
```

`stdio` transport only works for Claude Desktop (local subprocess). For any
HTTP-accessible deployment — cPanel, VPS, or cloud — `streamable-http` is required.
This is the transport Claude.ai uses for remote MCP connectors. The
`passenger_wsgi.py` entry point bypasses this call entirely (Passenger drives the
event loop directly via `mcp.http_app()`), but it's left here for local dev:

```bash
python server.py   # runs on MCP_SERVER_HOST:MCP_SERVER_PORT directly
```

---

## 9. Database: Force TCP on cPanel

```python
DB_HOST = "127.0.0.1"   # Not "localhost"
```

On cPanel, `"localhost"` causes MySQL to attempt a Unix domain socket connection.
The socket path inside a virtualenv Python process may not resolve correctly, causing
silent connection failures at startup. `127.0.0.1` forces TCP, which works reliably
regardless of the execution context.

---

## 10. Entra App Registration: Non-Obvious Checklist

| Item | Location | Required Value / Action |
|------|----------|------------------------|
| Access token version | Manifest → `accessTokenAcceptedVersion` | `2` |
| Application ID URI | Expose an API | e.g. `api://<guid>/app-action` |
| Redirect URI | Authentication → Web | `https://yourdomain/auth/callback` |
| Scope | Expose an API → Add a scope | Short name e.g. `Action.Read`; state: Enabled |
| API permission | API permissions → My APIs | Add delegated `Action.Read` |
| Admin consent | API permissions | Grant tenant-wide to suppress per-user prompts |
| Assignment required | Enterprise applications → Properties | Set to Yes + assign users/groups to restrict access |

---

## 11. Tools Require No Auth Arguments

Because `AzureProvider` validates the Bearer token at the HTTP transport layer before
any tool is invoked, individual tool functions need no token or credential parameters:

```python
@mcp.tool()
def catalog_summary() -> dict:
    """..."""
    return db.query_one("SELECT ...")
```

If token validation fails, FastMCP rejects the request before the tool function is
called. The tool layer is auth-transparent — keep it that way and avoid leaking auth
concerns into tool signatures.

Use `ToolError` for expected error conditions (not found, bad input) so FastMCP
returns a clean MCP-protocol error rather than an unhandled exception:

```python
from fastmcp.exceptions import ToolError

if not row:
    raise ToolError(f"Item '{item_number}' not found.")
```
