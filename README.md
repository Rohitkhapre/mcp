# JumpServer MCP Server

## Configure JumpServer Environment File (.env)

```txt
# Bearer token to access the JumpServer Swagger Json API, optional
api_token=xxxxxxx 
jumpserver_url=http://jumpserverhost
```

This `.env` credential is used only once, at server startup, to fetch the API schema (which
tools exist) — it's a single service-account credential, not a per-developer one, and it does
not affect per-developer permissions at all (see "Per-Developer Tokens" below for that). Since
it's read once at boot, an expiring Bearer token here will eventually cause the container to
fail to start again after a restart once the token expires. Prefer a permanent Access Key
instead, from a low-privilege (e.g. read-only/auditor) account rather than an admin — this
credential only needs to read schema metadata, not perform real actions, and it sits in a
config file on disk:

```txt
# Access Key auth (preferred over api_token: doesn't expire)
access_key_id=xxxx-xxxx-xxxx
access_key_secret=xxxxxxxxxxxxxxxx
jumpserver_url=http://jumpserverhost
```

When both `api_token` and `access_key_id`/`access_key_secret` are set, Access Key auth takes
precedence.

## Start Docker Container

```bash
docker run -d -it -p 8099:8099 --env-file .env --name jms_mcp ghcr.io/jumpserver/mcp:latest
```

## Create JumpServer API Bearer Token for MCP Server

```shell

TOKEN=$(curl -s -X POST http://jumpserver_host/api/v1/authentication/auth/ \
  -H "Content-Type: application/json" \
  -d '{
    "username": "admin",
    "password": "xxxx"
  }' \
  --insecure | jq -r '.token')

echo "Your Bearer token: $TOKEN"

```


## MCP Server Configuration

```json
{
    "type": "sse",
    "url": "http://127.0.0.1:8099/sse",
    "headers": {
        "Authorization": "Bearer xxxxxxxx"
    }
}
```

## Per-Developer Tokens and Individual Activity Tracking

If your organization has multiple developers connecting through one MCP server, give each
developer their own JumpServer credential instead of sharing a single `api_token`. Every
credential a client sends when it connects is forwarded per-session, so every JumpServer API
call that developer makes is authenticated as *them* — which means it shows up under their own
account in JumpServer's audit/operation logs, not a shared service account.

Two credential types work this way; pick based on whether MFA is enforced on your accounts:

### Option A: Access Key (recommended when MFA is enabled)

JumpServer's username/password login endpoint (used to mint a Bearer token, see below) returns
`{"error": "mfa_required"}` once MFA is turned on for an account, and does not hand back a usable
token without completing an extra OTP challenge step. An Access Key sidesteps this entirely: it's
a separate, permanent credential type that JumpServer verifies by request signature rather than
by re-running the password/MFA login flow, and each developer generates their own from their own
account — no admin involvement, and MFA stays enabled.

1. Each developer creates their own Access Key in JumpServer: **Profile → Access Keys → Generate
   Access Key**. Save the `AccessKeyID` and `AccessKeySecret` immediately — JumpServer only shows
   the secret once.

2. Each developer configures their own MCP client with their own key pair:

   ```json
   {
       "type": "sse",
       "url": "http://127.0.0.1:8099/sse",
       "headers": {
           "X-JumpServer-Access-Key-Id": "<alice-own-key-id>",
           "X-JumpServer-Access-Key-Secret": "<alice-own-key-secret>"
       }
   }
   ```

   The MCP server signs every JumpServer API request on the developer's behalf using their key,
   so there's no token to refresh and nothing to keep valid across restarts.

### Option B: Bearer login token (only if MFA is not enforced)

1. Have each developer generate their own token using their own JumpServer username/password:

   ```shell
   TOKEN=$(curl -s -X POST http://jumpserver_host/api/v1/authentication/auth/ \
     -H "Content-Type: application/json" \
     -d '{
       "username": "alice",
       "password": "xxxx"
     }' \
     --insecure | jq -r '.token')
   ```

2. Each developer configures their own MCP client with their own token:

   ```json
   {
       "type": "sse",
       "url": "http://127.0.0.1:8099/sse",
       "headers": {
           "Authorization": "Bearer <alice-own-token>"
       }
   }
   ```

   This token expires and must be regenerated periodically; if MFA is enabled on the account,
   this endpoint will refuse to issue one at all. Prefer Option A when that's the case.

### Gating the MCP server itself

Whichever option you use, `Authorization` (and the two Access Key headers) stay reserved for
each developer's own JumpServer credential. To also gate who can reach the MCP server at all,
set a separate shared secret with `gateway_key` in `.env`, and have every client send it as a
distinct header:

```txt
gateway_key=some-shared-secret
```

```json
{
    "type": "sse",
    "url": "http://127.0.0.1:8099/sse",
    "headers": {
        "X-JumpServer-Access-Key-Id": "<alice-own-key-id>",
        "X-JumpServer-Access-Key-Secret": "<alice-own-key-secret>",
        "X-MCP-Gateway-Key": "some-shared-secret"
    }
}
```

`api_key` still works as before for deployments that don't need per-user attribution, but it
checks the same `Authorization` header used for Bearer tokens, so it can't be combined with
per-developer credentials — use `gateway_key` instead when you need both.
