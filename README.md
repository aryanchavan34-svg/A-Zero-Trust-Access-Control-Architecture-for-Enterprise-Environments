# Zero Trust Architecture for Enterprise Security — Lab

Minor Project 2. A working, containerized Zero Trust environment: identity
verification (Keycloak + MFA), a policy enforcement point (Traefik +
forward-auth), RBAC-gated services, and micro-segmented networks — plus
insider/external threat simulations with documented results.

## Architecture

```
Internet ──> Traefik (PEP) ──> forward-auth ──> Keycloak (IdP, MFA/TOTP)
                 │
                 ├── /admin  ──> admin-app   (backend-admin network, internal)
                 ├── /dev    ──> dev-app     (backend-dev network, internal)
                 └── /       ──> public-app  (frontend network)
```

Every request hits Traefik first. Traefik forwards it to `forward-auth`,
which checks the session against Keycloak (OIDC). Unauthenticated requests
are redirected to log in; authenticated requests carry role claims that
gate access to `/admin` and `/dev`. `admin-app` and `dev-app` sit on
**separate, `internal: true` Docker networks** — they cannot reach each
other directly, and neither has direct internet egress. Traefik is the
only component bridging into both, simulating micro-segmentation with a
single, auditable enforcement point.

## Setup

```bash
cd zero-trust-lab
docker compose up -d
```

Then:
1. Open `http://localhost:8080` → Keycloak admin console (`admin` /
   `admin_change_me`) and confirm the `zero-trust-lab` realm imported with
   users `alice-admin`, `bob-dev`, `carol-guest` (all password
   `ChangeMe123!`, all forced to set up TOTP on first login).
2. Visit `http://localhost/admin`, `http://localhost/dev`, and
   `http://localhost/` and log in as each user to see access granted/denied.
3. Traefik dashboard: `http://localhost:8081`

## Testing (insider / external threat simulations)

Write these up in `tests/` as you run them — each as setup → attempt →
expected vs. actual → screenshot:

- **External threat**: try reaching `http://admin-app` or
  `http://dev-app` directly (bypassing Traefik) — should fail, since
  those networks are `internal` and not published.
- **Insider threat (privilege escalation)**: log in as `bob-dev`
  (developer role) and request `/admin` — should be denied.
- **Lateral movement**: `docker exec` into `admin-app` and try to curl
  `dev-app` — should fail, they don't share a network.
- **MFA bypass attempt**: try logging in with password only, TOTP
  disabled — Keycloak's `requiredActions: CONFIGURE_TOTP` should force
  enrollment before any session is issued.

## Honest scope notes (worth stating in your report)

- **RBAC enforcement here is intentionally lab-scale.** The
  `require-*-role` middlewares in `traefik/dynamic.yml` inject a header
  but don't yet cryptographically verify the JWT's role claim against
  it — for a stronger build, replace this with a small **Open Policy
  Agent (OPA)** sidecar that inspects the token's `realm_access.roles`
  claim and returns allow/deny. Naming this gap explicitly in your
  report (and proposing OPA as the fix) reads as more credible than
  pretending the shortcut is production-grade — this is exactly the kind
  of "suggest improvements" step the brief asks for.
- **TLS is omitted** for lab simplicity (`INSECURE_COOKIE=true`, plain
  HTTP). In a real deployment, every hop here needs TLS — call this out
  as a limitation too.
- Default secrets in this repo (`admin_change_me`, `ChangeMe123!`, etc.)
  are placeholders — rotate them before you screenshot/demo, and never
  commit real secrets to a public repo.

## Repo layout

```
zero-trust-lab/
├── docker-compose.yml
├── keycloak/realm-export.json     # roles, MFA policy, demo users
├── traefik/dynamic.yml            # PEP middlewares (auth + RBAC)
├── apps/{admin,dev,public}/       # sample backend services
└── tests/                         # threat-simulation writeups (add as you test)
```
