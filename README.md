# Remote MCP Server on GCP: Cloud Functions with Google OAuth Login

Connect Claude directly to your GCP resource inventory. No local proxy. No
service account key file on your laptop. You paste a URL, you log in with
Google, and the tools work.

This is the OAuth port of [`gcp-serverless-mcp`](https://github.com/mamonaco1973/gcp-serverless-mcp), which kept the Cloud Function
private behind Cloud Run IAM and shipped a local proxy that signed OIDC
assertions with a **downloaded service account private key**. That key was the
single credential for all MCP access — no expiry, no rotation, no per-user
identity, sitting in plaintext in a folder.

This version deletes it.

| | Proxy build | This build |
|---|---|---|
| Client setup | Install proxy, edit JSON config, hold a key file | Paste one URL |
| Credential on disk | SA private key, never expires | None |
| Who is the caller? | Always the same service account | The actual human, via Google |
| Function exposure | Private (Cloud Run IAM) | Public; the code enforces auth |
| Auth enforced by | The platform | `mcp.py`, on every call |

| Piece | Where | Role |
|---|---|---|
| **gcp-oauth-mcp-func** | Cloud Functions, `us-central1` | 2nd gen, Python 3.11, 256 MB, 60 s, 0 to 10 instances; public (`allUsers` invoker); `oauth.py` (the broker), `mcp.py` (JSON-RPC and token checks) and `tools.py`; the name is fixed so its redirect URI stays the same |
| **gcp-oauth-mcp-sa** | IAM | the function's identity: `cloudasset.viewer`, `storage.objectViewer` and `datastore.user`, and access to the one secret |
| **Firestore** | `(default)`, native mode | collections `mcp_pending_auth` and `mcp_auth_codes`, each with a TTL policy on `expires_at` |
| **gcp-oauth-mcp-client-secret** | Secret Manager | your Google OAuth client secret |
| **Google OAuth client** | your project | a **Web application** client you create by hand; its redirect URI is the function's `/oauth/callback` |
| **gcp-oauth-mcp-src-&lt;hex&gt;** | Cloud Storage | the function's source zip |

## Architecture

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="architecture-dark.svg">
  <img alt="Claude reaches gcp-oauth-mcp-func, a public second-generation Cloud Function that is both an OAuth authorization server and the MCP server. Claude probes /mcp, discovers the authorization server, registers itself, and sends the user's browser through /authorize to Google sign-in with your OAuth client; the function exchanges the code for tokens and returns the Google access token to Claude. Login state lives in Firestore with TTL policies, and the OAuth client secret comes from Secret Manager. MCP calls on /mcp carry the access token, which the function checks with Google, pinning the audience to its own client ID, and the ten tools query Cloud Asset Inventory and Cloud Storage as the gcp-oauth-mcp-sa service account." src="architecture-light.svg">
</picture>

Regenerate with `python3 make_diagram.py`.

The function plays two roles at once: to Claude it *is* the OAuth
authorization server, and to Google it is an ordinary OAuth *client*.
Connecting goes like this:

1. **Probe:** `POST /mcp` with no token returns `401` and a
   `WWW-Authenticate` header pointing at the metadata.
2. **Discover:** `GET /.well-known/oauth-authorization-server` (RFC 8414).
3. **Register:** `POST /oauth/register` (RFC 7591) gives Claude a client ID.
4. **Log in:** `GET /authorize` sends the browser to `accounts.google.com`,
   which returns to `/oauth/callback`.
5. **Token:** `POST /oauth/token` returns the Google access token.
6. **Use:** `POST /mcp` with `Authorization: Bearer <access token>`.

### Why a broker, and not just "point Claude at Google"?

Two gaps, neither of them ours to fix upstream:

1. **claude.ai's redirect URI is dynamic.** It embeds the org ID —
   `https://claude.ai/api/organizations/<id>/mcp/callback`. Google requires an
   exact allow-list match, so you can never register it. The broker registers
   only its own fixed `/oauth/callback` and carries Claude's URL through
   Firestore.

2. **Google has no dynamic client registration** (RFC 7591). Without a
   `/oauth/register` endpoint, Claude has no `client_id` — and the user ends up
   pasting a client ID and secret by hand.

Roughly 300 lines of `oauth.py` is what stands between "paste a URL" and "paste
a client ID, a client secret, and hope."

---

## The tools

| Tool | Operation |
|---|---|
| `list_compute_instances` | All VMs with machine type, zone, status |
| `list_storage_buckets` | All GCS buckets with location and storage class |
| `count_resources_by_type` | Ranked inventory summary |
| `find_resources_by_label` | Resources matching a label key+value |
| `list_static_ip_addresses` | All static external IPs |
| `find_resources_by_type` | Resources of a specific asset type |
| `find_resources_by_region` | Resources in a region or zone |
| `describe_resource` | Full config detail for a named resource |
| `list_cloud_functions_detail` | Runtime, memory, URL, service account, env vars |
| `list_bucket_objects` | Objects in a bucket with size and last-modified |

Responses are pre-formatted plain text, not JSON — Cloud Asset Inventory returns
deeply nested proto structs, and the model narrates a text table far better than
it parses one.

## Prerequisites

- `gcloud`, `terraform`, `jq`, `curl` in PATH
- `credentials.json` (GCP service account key) in the repo root
- That service account needs: Cloud Functions Admin, Cloud Run Admin, Cloud
  Build Editor, Artifact Registry Admin, IAM Admin, Cloud Asset Viewer, Storage
  Admin, Service Account Admin, Project IAM Admin, Secret Manager Admin,
  Datastore Owner

If this is your first time following along, we recommend starting with this
video: [GCP + Terraform: Easy Setup](https://youtu.be/3spJpYX4f7I).

## Deploy

```bash
git clone https://github.com/mamonaco1973/gcp-oauth-mcp.git
cd gcp-oauth-mcp
```

### Step 1 — create the Google OAuth client (once, by hand)

Terraform cannot do this for you. `google_iap_client` requires an IAP brand, and
external brands can only be created in the console. This is the one manual step.

1. **APIs & Services → Credentials → Create Credentials → OAuth client ID**
2. Application type: **Web application**
3. Leave the redirect URI blank for now — the function does not exist yet.
4. Copy the client ID and secret.

```bash
export MCP_GOOGLE_CLIENT_ID="123456789-abc.apps.googleusercontent.com"
export MCP_GOOGLE_CLIENT_SECRET="GOCSPX-..."
```

`check_env.sh` hard-fails if either is missing — Google sign-in is not an
optional extra here, it *is* the authentication.

### Step 2 — apply

```bash
./apply.sh
```

When it finishes it prints the **authorized redirect URI**. Paste that back onto
the OAuth client in the console.

The function name is **not randomised**, so that URI is stable — you do this
once, not after every rebuild.

### Step 3 — publish the consent screen

Confirm the OAuth consent screen is **Published**, not "Testing". In Testing mode
refresh tokens expire after 7 days and only allow-listed users can sign in.

Publishing needs **no Google verification review**: `openid`, `email`, and
`profile` are all non-sensitive scopes. It is a button, not an audit.

`apply.sh` ends by running `validate.sh`, which checks the OAuth handshake
and that the tools are protected:

```
NOTE: OK   discovery (RFC 8414) (HTTP 200)
NOTE: OK   resource metadata (HTTP 200)
NOTE: OK   registration (RFC 7591) (HTTP 201)
NOTE: OK   /mcp rejects no token (HTTP 401)
NOTE: OK   /mcp rejects bad token (HTTP 401)

========================================================================
  Validation complete — handshake works, tools are protected.
========================================================================
  Connector URL: https://gcp-oauth-mcp-func-<hash>.a.run.app/mcp
========================================================================
```

## Try it

1. In Claude, open **Settings > Connectors > Add custom connector** and
   paste the **Connector URL**.
2. Claude discovers the authorization server, registers itself, and sends
   you to Google to sign in.
3. Ask, for example, "What's running in us-central1?" or "Count my
   resources by type."

## Teardown

```bash
./destroy.sh
```

## Security — what this does and does not do

**It authenticates. It does not authorize.**

Every authenticated Google user is authorized. There is no allow-list. Anyone
with a Google account who reaches the endpoint can read this project's resource
inventory. That is an acceptable trade in a demo and a bad one anywhere else —
to lock it down, filter on the `email` or `hd` claim in `mcp._resolve_user`.

Three things this build does get right, and they are worth understanding:

**The token's audience is pinned.** `_resolve_user` rejects any token whose `aud`
is not our own `client_id`. This is not optional paranoia: a naive `/userinfo`
check accepts a valid Google token minted for *any* application on the internet,
which would let an unrelated app's token call these tools.

**The client secret lives in Secret Manager**, not a plain environment variable —
because `list_cloud_functions_detail` prints function environment variables. A
plaintext secret would be readable through the very tools it protects.

**The function is public, and that is correct.** `allUsers` holds
`roles/run.invoker`. The OAuth endpoints cannot require a token — obtaining one
is their job — and Claude probes `/mcp` unauthenticated on purpose to read the
`WWW-Authenticate` header. An IAM check would break the handshake before any code
ran. So the door opens, and `mcp.py` enforces the token instead.

## Gotchas

- **Google access tokens last one hour**, and that is not configurable. The
  `refresh_token` grant is mandatory here (the Cognito equivalent could skip it —
  Cognito allows a 24-hour access token).
- **`prompt=consent` in `/authorize` is load-bearing.** Without it Google omits the
  refresh token for a user who already granted access, and the connector dies
  silently after an hour.
- **`redirect_uri_mismatch` at login** means the URI `apply.sh` printed is not on
  the OAuth client. It is the single most likely thing to go wrong.
- **Don't randomise the function name.** The URL derives from it, and that URL is
  the registered redirect URI.
