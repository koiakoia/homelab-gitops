# External Access

Keycloak serves as the authentication backbone for both internal and external access to Project Sentinel services. This page documents the external access architecture and how Keycloak integrates with Cloudflare Access and oauth2-proxy.

## Two-Tier Access Architecture

```
EXTERNAL (*.${DOMAIN})                    INTERNAL (*.${INTERNAL_DOMAIN})
========================                   ============================

Internet User                              LAN / Tailscale User
     |                                          |
Cloudflare DNS (CNAME)                     Split DNS / dnsmasq
     |                                          |
Cloudflare Access                          ${PROXY_IP}
  (Keycloak IdP)                                |
     |                                     Traefik :443
Cloudflare Tunnel                               |
     |                                     OKD Router
cloudflared (pangolin-proxy)                    |
     |                                     Service :8080
Traefik :443
     |
  +--+-- oauth2-proxy ForwardAuth?
  |      (gitlab, matrix only)
  |
OKD Router
     |
Service :8080
```

## Cloudflare Access Integration

### How It Works

Cloudflare Access acts as a pre-authentication layer for externally published services. Keycloak is configured as the Identity Provider for Cloudflare Access via the `cloudflare-access` OIDC client.

**Flow**:

1. User visits `gitlab.${DOMAIN}` or `auth.${DOMAIN}` (any CF Access-protected domain)
2. Cloudflare Access checks for a valid CF Access JWT cookie
3. If no valid cookie, user is redirected to Cloudflare's login page
4. Cloudflare presents Keycloak as the IdP option
5. User authenticates with Keycloak at `auth.${DOMAIN}`
6. Keycloak returns an OIDC token to Cloudflare
7. Cloudflare sets a CF Access JWT cookie and proxies the request

**Keycloak client**: `cloudflare-access`

The `cloudflare-access` client in Keycloak is a standard OIDC confidential client. Cloudflare Access is configured with:

- **Application ID**: `3e540434-...`
- **IdP type**: OpenID Connect
- **Authorization URL**: `https://auth.${DOMAIN}/realms/sentinel/protocol/openid-connect/auth`
- **Token URL**: `https://auth.${DOMAIN}/realms/sentinel/protocol/openid-connect/token`

### Externally Published Services

| External URL | Backend | CF Access | oauth2-proxy |
|-------------|---------|-----------|--------------|
| `gitlab.${DOMAIN}` | GitLab (${GITLAB_IP}) | Yes | Yes |
| `auth.${DOMAIN}` | Keycloak (OKD) | Yes | No |
| `social.${DOMAIN}` | Mattermost (OKD) | Yes | No |
| `matrix.${DOMAIN}` | Matrix/Synapse (OKD) | Yes | Yes |
| `www.${DOMAIN}` | Haists Website (OKD) | No [VERIFY] | No |

## oauth2-proxy ForwardAuth

### Architecture

oauth2-proxy v7.7.1 runs on pangolin-proxy alongside Traefik. It provides a second layer of authentication via Traefik's ForwardAuth middleware for select external services.

**Why both CF Access and oauth2-proxy?** Cloudflare Access provides perimeter authentication, but oauth2-proxy adds service-level OIDC authentication that issues cookies scoped to `.${DOMAIN}`. This ensures the user has a valid Keycloak session that downstream services can consume, not just a Cloudflare Access JWT.

### Configuration

| Property | Value |
|----------|-------|
| **Keycloak client** | `oauth2-proxy` |
| **Issuer** | `https://auth.${DOMAIN}/realms/sentinel` |
| **Cookie name** | `_oauth2_proxy` |
| **Cookie domain** | `.${DOMAIN}` |
| **Cookie expiry** | 10 hours |
| **Cookie refresh** | 30 minutes |
| **Cookie flags** | `Secure`, `HttpOnly`, `SameSite=Lax` |
| **Client credentials** | Vault `secret/keycloak/oauth2-proxy` |
| **Cookie secret** | Vault `secret/pangolin/oauth2-proxy` |

**Critical**: The oauth2-proxy issuer MUST be `auth.${DOMAIN}` (the external Keycloak URL), NOT `auth.${INTERNAL_DOMAIN}`. This is because:

1. oauth2-proxy redirects users to the issuer URL for login
2. External users can only reach `auth.${DOMAIN}` (via Cloudflare)
3. Using the same issuer as the apps means the Keycloak session cookie is shared, avoiding double login

### Traefik Middleware

The ForwardAuth middleware is defined in `sentinel-middlewares.yml` on pangolin-proxy:

```
Middleware: oauth2-forward-auth
Forward URL: http://oauth2-proxy:4180/
```

**Important**: The ForwardAuth URL must point to the root path (`/`), NOT `/oauth2/auth`. The root path returns a 302 redirect for unauthenticated users (which Traefik can follow), while `/oauth2/auth` returns a 401 (which Traefik cannot redirect).

### Protected Services and Bypass Routes

**gitlab.${DOMAIN}** -- ForwardAuth enabled with bypass paths:

- `/api/` -- GitLab API (uses PAT auth)
- `/-/health` -- Health checks
- `/users/auth/` -- OmniAuth callbacks
- `/oauth/` -- OAuth token endpoint
- `/jwt/auth` -- JWT authentication (Harbor integration)
- `/robots.txt` -- Search engine robots

**matrix.${DOMAIN}** -- ForwardAuth enabled with bypass paths:

- `/_matrix/` -- Matrix client-server and federation API
- `/_synapse/` -- Synapse admin API
- `/.well-known/` -- Matrix server discovery

### IaC File Locations

oauth2-proxy configuration files in `sentinel-iac-work/pangolin/`:

- `docker-compose.yml` -- oauth2-proxy container definition
- `dynamic/sentinel-middlewares.yml` -- Traefik ForwardAuth middleware
- `dynamic/sentinel-external-services.yml` -- External route definitions with middleware references
- `.env.oauth2-proxy.template` -- Environment variable template

## DNS Requirements

**Critical rule**: `*.${INTERNAL_DOMAIN}` domains must NEVER have Cloudflare DNS CNAME records. These domains are resolved via Tailscale split DNS or LAN dnsmasq to `${PROXY_IP}`. Adding Cloudflare CNAMEs would override Tailscale split DNS resolution and break internal access for all services.

External domains (`*.${DOMAIN}` without the `208` prefix) use Cloudflare DNS CNAMEs pointing to the Cloudflare Tunnel.

## Session Sharing

The following session sharing model prevents users from having to authenticate multiple times:

1. **Keycloak session** -- created when user logs in at `auth.${DOMAIN}`, stored as a cookie on `auth.${DOMAIN}`
2. **oauth2-proxy cookie** -- `_oauth2_proxy` on `.${DOMAIN}`, created after oauth2-proxy validates the Keycloak token
3. **Cloudflare Access JWT** -- set by Cloudflare after CF Access authentication

Because oauth2-proxy uses `auth.${DOMAIN}` as its issuer, and the user already has a Keycloak session from the CF Access login, oauth2-proxy can silently obtain a token without showing another login prompt. The result is a single login experience even though three authentication layers are traversed.

## Keycloak Issuer URLs

Different services use different Keycloak issuer URLs depending on network reachability:

| Context | Issuer URL | Why |
|---------|-----------|-----|
| External services (oauth2-proxy, CF Access) | `https://auth.${DOMAIN}/realms/sentinel` | Users must reach login page via internet |
| Internal OKD services (Grafana, ArgoCD, etc.) | `https://auth.${INTERNAL_DOMAIN}/realms/sentinel` | Pods resolve via internal DNS |
| MAS (Matrix Authentication Service) | `https://auth.${INTERNAL_DOMAIN}/realms/sentinel` | Runs in air-gapped OKD, cannot reach Cloudflare |
| Redirect URIs | Both internal and external hostnames | Users may access from either network |
