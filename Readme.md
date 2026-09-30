# Home Lab PKI + IAM Stack: step-ca + Traefik + Keycloak → CipherTrust Manager OIDC

A self-contained Docker Compose stack that stands up a private certificate authority, a reverse proxy that automates certificate issuance, and an OIDC identity provider — then wires that identity provider into **Thales CipherTrust Manager (CTM)** so you can log into the CTM web GUI via SSO instead of local accounts.

```
                         ┌─────────────────────────────────────────┐
                         │              Docker network              │
                         │                                           │
   ACME (HTTP-01) ┌──────┤  step-ca  ◄──── issues certs for ────┐   │
                   │      │ (private CA)                        │   │
                   ▼      └─────────────────────────────────────┼───┘
            ┌────────────┐                                      │
   :80/:443 │  Traefik   │◄─────────────────────────────────────┘
            │(reverse    │
            │ proxy/TLS) │
            └─────┬──────┘
                   │ Host: keycloak.home.arpa
                   ▼
            ┌────────────┐          OIDC discovery / auth code flow
            │  Keycloak  │◄────────────────────────────────────────┐
            │ (OIDC IdP) │                                          │
            └────────────┘                                          │
                                                              ┌──────┴──────┐
                                                              │ CipherTrust │
                                                              │  Manager    │
                                                              │  (web GUI)  │
                                                              └─────────────┘
```

## What this gives you

- **step-ca** — a private, self-hosted certificate authority (Smallstep), auto-initialized on first boot, serving certificates over ACME.
- **Traefik** — reverse proxy and TLS terminator. Automatically requests and renews a certificate for Keycloak from step-ca via ACME (HTTP-01 challenge), and routes traffic to backend containers by Docker labels.
- **Keycloak** — OIDC identity provider, running in dev mode (`start-dev`), reachable at `https://keycloak.home.arpa`.
- A working **OIDC connection from CipherTrust Manager to Keycloak**, so CTM's login page offers Keycloak as an identity provider alongside (or instead of) local/domain accounts.

## Prerequisites

- Docker and Docker Compose v2
- A CipherTrust Manager instance with an SSL certificate validation fix for internally-signed (private CA) OIDC identity providers. **CTM historically rejected privately-signed IdP certificates for OIDC connections outright** (`failed to parse JSON` / `NCERRInvalidParamValue` errors), regardless of whether the CA was externally hosted or CTM's own local CA, and regardless of correct certificate chains — this was a product-level limitation, not a configuration problem. Confirm your CTM build includes the fix before expecting this to work over HTTPS.
- DNS resolution for `keycloak.home.arpa` (and your CTM hostname, e.g. `ciphertrust.home.arpa`) pointing at the Docker host — e.g. static entries on your router or local DNS server. `home.arpa` is the [RFC 8375](https://www.rfc-editor.org/rfc/rfc8375) reserved TLD for residential home networks; avoid `.local` (mDNS conflicts).
- Ports 80 and 443 free on the Docker host.

## Quick start

1. **Clone and configure secrets.** Copy `.env.example` to `.env` and set your own values — do not use the defaults in production or leave them in version control:
   ```
   cp .env.example .env
   ```

2. **Bring up step-ca first** so it can self-initialize and generate its root CA before anything else depends on it:
   ```
   docker compose up -d step-ca
   docker compose logs -f step-ca   # wait for it to report ready, then Ctrl-C
   ```

3. **Extract step-ca's root certificate** — you'll need this to trust the CA elsewhere (browsers, CipherTrust Manager):
   ```
   docker exec step-ca cat /home/step/certs/root_ca.crt > stepca-root.crt
   ```

4. **Bring up the rest of the stack:**
   ```
   docker compose up -d
   docker compose logs -f traefik keycloak
   ```
   Confirm Traefik successfully issues a certificate for Keycloak via ACME (no errors in the logs), and that Keycloak reaches a running state (not restart-looping).

5. **Verify the certificate:**
   ```
   curl -vk https://keycloak.home.arpa 2>&1 | grep -A2 "issuer:"
   ```
   Should show your step-ca CA name as the issuer.

6. **Create a Keycloak realm and OIDC client for CTM.** Log into `https://keycloak.home.arpa/admin` with the admin credentials from your `.env`:
   - Create a new realm (e.g. `ciphertrust`).
   - Create a client (e.g. `ciphertrust`), OpenID Connect, **Client authentication ON**, **Standard flow** (Authorization Code) enabled.
   - Set **Valid redirect URIs** to your CTM callback, e.g. `https://ciphertrust.home.arpa/api/v1/auth/oidc-callback`.
   - Copy the client secret from the client's **Credentials** tab.
   - Note the realm's discovery URL: `https://keycloak.home.arpa/realms/<realm>/.well-known/openid-configuration`.

7. **Trust step-ca's root in CipherTrust Manager.** In CTM: **Trusted Certificate Authorities → Add Trusted CA**, upload `stepca-root.crt`, and scope it to the service(s) that cover Access Management / OIDC connections (naming varies by CTM version — look for something like *User Management* or *Platform*). Adding a Trusted CA may require a service/microservice restart across the cluster before it takes effect — check for a restart prompt or trigger one manually if the connection still fails afterward.

8. **Create the OIDC connection in CTM**, using:
   - **Discovery URI**: the realm's discovery URL from step 6
   - **Client ID** / **Client Secret**: from step 6
   - **Redirect URI**: matching what you set in Keycloak
   - **Flow Type**: Authorization Code

9. **Test it.** Browse to CTM's login page using its actual configured hostname (not its IP address — CTM validates the redirect URI against the exact landing page URL), select the Keycloak identity provider, and log in.

## Files

| File | Purpose |
|---|---|
| `docker-compose.yml` | The full stack: step-ca, Traefik, Keycloak |
| `.env.example` | Template for secrets (passwords, admin credentials) — copy to `.env` |
| `stepca-root.crt` | step-ca's root certificate, extracted after first boot (generated, not committed) |

## Notes and gotchas

- **step-ca's default ACME certificate lifetime is short (often 24 hours).** Traefik renews it automatically as long as it stays running — this is expected behavior, not a fault.
- **CTM validates the browser's current URL against the OIDC connection's registered redirect URI before offering that identity provider at all** — accessing CTM's GUI via its IP address instead of its proper hostname will produce a "no matching allowed redirect URI" error before you even reach the login form.
- **`start-dev` mode** is used for Keycloak here, which is appropriate for a lab/test environment. For anything production-facing, build an optimized image (`kc.sh build`) and use `start` instead.
- **A diagnostic `ports: - "8080:8080"` mapping is present on the `keycloak` service but commented out.** Uncommenting it exposes Keycloak directly over plain HTTP, bypassing Traefik/TLS entirely — useful for isolating whether a failure is TLS/trust-related or a Keycloak/realm/client config problem (e.g. testing an OIDC connection over HTTP first). Leave it commented out for normal use; re-comment it (and `docker compose up -d`) once you're done diagnosing, so Keycloak is only reachable through Traefik/HTTPS again.
- **Destroying and rebuilding the stack** (`docker compose down -v`) wipes step-ca's CA (so `stepca-root.crt` must be re-extracted and re-trusted everywhere) and Keycloak's realm data (so the realm/client must be recreated). Compose config changes alone (e.g. editing labels or environment variables) only need `docker compose up -d` — no teardown required.

## License

Add your license of choice here.
