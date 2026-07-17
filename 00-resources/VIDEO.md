#GCP #MCP #CloudFunctions #OAuth #ClaudeAI

*Connect Claude to Google Cloud — MCP Secured with Google Login*

Connect Claude directly to your Google Cloud project. No local proxy. No service account key file on your laptop. You paste one URL, log in with Google, and the tools work.

In this project we put ten Cloud Asset Inventory tools behind a single Cloud Function, and secure it with Google OAuth. The function is public, and it enforces the token itself — the login routes have to be reachable before a token exists, so authentication moves into the code. Claude discovers the login, registers itself, and sends you to Google. Nothing to configure but the URL.

This is the OAuth port of a serverless MCP server that used a local proxy signing OIDC assertions with a downloaded service account key — the single credential for all access, sitting in a folder. This version deletes it.

But here is the catch. Google does not implement dynamic client registration, so on its own an MCP client has no way to sign itself up. That is the roughly three hundred lines the function has to write by hand — and it is the exact gap AWS and Azure leave open too. No cloud closes it for you.

We use Cloud Asset Inventory as the example tool set, but the pattern works for any Cloud Function-backed MCP server.

WHAT YOU'LL LEARN
• Exposing Cloud Functions as MCP tools over a remote endpoint Claude connects to directly
• Why the function is public, and how authentication is enforced in code instead of by the platform
• Brokering Google OAuth for an MCP client — discovery, dynamic client registration, authorize, callback, token, and refresh
• Why Google does not serve dynamic client registration, and the one manual step that leaves you
• Validating the access token by pinning its audience to your own OAuth client
• The honest comparison — three clouds, and the same RFC none of them will implement for you

INFRASTRUCTURE DEPLOYED
• One Cloud Function 2nd Gen (Python 3.11) — the OAuth broker, the MCP endpoint, and ten tools, public with auth enforced in code
• Cloud Asset Inventory access via Application Default Credentials — no credentials in code
• Firestore for transient OAuth login state, swept on a short TTL
• The Google OAuth client secret held in Secret Manager, never a plaintext environment variable
• A Google OAuth client you create once by hand — the one piece Terraform cannot provision
• All provisioned with Terraform in a single apply, torn down with a single command

GitHub
https://github.com/mamonaco1973/gcp-oauth-mcp

README
https://github.com/mamonaco1973/gcp-oauth-mcp/blob/main/README.md

TIMESTAMPS
00:00 Introduction
00:34 Architecture
01:26 Securing MCP
02:28 Deploy It Yourself
