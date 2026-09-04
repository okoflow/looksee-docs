[Documentation](index.md) / Security

# Security

This page describes how LookSee protects an instance and what an operator has
to take care of. It complements the
[security policy](https://github.com/okoflow/looksee/blob/main/SECURITY.md)
in the repository, which covers reporting vulnerabilities.

## Accounts and sessions

- One account exists: the owner, created on the setup page after the first
  start. There is no self-registration and no user management.
- Passwords are hashed with Argon2. The setup form requires at least eight
  characters with one digit and one capital letter.
- Signing in sets a `looksee_session` cookie: `HttpOnly`, `SameSite=Lax`,
  valid for seven days, and `Secure` when `AUTH_COOKIE_SECURE=true`. Signing
  out deletes the cookie; the token itself stays valid until it expires, so
  treat a leaked cookie as a reason to rotate `SECRET_KEY`.
- Every API route except `/health`, `/auth/*`, the interactive
  documentation, and the MediaMTX callback `/internal/media/auth` requires
  the cookie.
- Setup and sign-in accept 20 attempts per client address and 100 in total
  per minute; further attempts get `429` with a `Retry-After` header.

> [!WARNING]
> Until the owner account exists, the setup endpoint is open to anyone who can
> reach Studio or the API. Create the owner immediately after the first
> start, or start the stack with the ports firewalled and open them
> afterwards.

## Secrets

The API derives two keys from one root secret: a signing key for session
tokens and an encryption key for stored credentials. The root secret is
`SECRET_KEY` when set; otherwise the API generates one on first start and
stores it in `SECRET_KEY_FILE` on the `api_keys` volume with owner-only
permissions.

- Losing the secret signs every user out and makes stored credentials
  unreadable. Back up the `api_keys` volume or set `SECRET_KEY` explicitly.
- Credential payloads (bot tokens, webhook URLs, SMTP and MQTT passwords) are
  encrypted at rest and never returned by the API; the list shows a name and
  a non-secret summary. They are decrypted only when an action delivers.
- Rotating `SECRET_KEY` invalidates sessions and existing credential
  ciphertexts; re-enter credentials afterwards.

## Media access

Live video and webcam publishing go from the browser straight to MediaMTX,
but MediaMTX holds no credentials of its own: it asks the API to authorize
every connection through `/internal/media/auth`. Three kinds of callers
exist:

- Studio requests a grant from `POST /cameras/{camera_id}/media-access`
  with the session cookie and sends it to MediaMTX as a bearer token. A
  grant covers one camera and one action, `read` or `publish`, and expires
  after five minutes; publishing is granted only for Browser webcam cameras.
- The API and the inference service use the service user, `MTX_MEDIA_USER`
  with `MTX_MEDIA_PASSWORD`, to manage paths and to read camera streams. The
  password stays inside the stack and is never sent to a browser.
- The ffmpeg loop of a File camera publishes over RTSP from the MediaMTX
  container itself and is accepted from loopback only.

Consequences for a deployment:

- Treat `MTX_MEDIA_PASSWORD` as a backend secret: rotate it if it was ever
  exposed, and do not reuse it anywhere else.
- Keep MediaMTX ports (`8554`, `8889`, `8189/udp`) reachable only from the
  networks where cameras and Studio users are; a valid grant is still
  required, but the ports need not be public.
- The callback is served on the API port and answers only allow or deny.

## Network exposure

| Exposed to all interfaces | Loopback only |
| --- | --- |
| Studio `3000`, API `8000`, RTSP `8554`, WebRTC `8889` and `8189/udp` | PostgreSQL `5432`, Valkey `6379`, video storage `9000`, MediaMTX control API `9997` |

The MediaMTX control API accepts unauthenticated requests from loopback and
private address ranges; it is bound to loopback so only the API container
and the host reach it. Do not publish port `9997`.

The API's CORS policy allows `localhost` and `127.0.0.1` by default. Set
`CORS_ORIGIN_REGEX` to exactly the Studio origin in a deployment, and do not
widen it to every origin: the API accepts credentials with cross-origin
requests. Requests that change state and WebSocket connections whose
`Origin` header matches neither the pattern nor the API's own origin are
rejected with `403`.

## Actions and outbound traffic

Webhook, Telegram, Discord, Slack, MQTT, and email actions connect to the
addresses configured in the workflow or the credential. There is no allow
list, so a user who can edit workflows can make the API connect to internal
addresses. Only the owner can edit workflows; keep that in mind when giving
someone the owner password.

Snapshot images are served at `/snapshots/<file>` to any request with a
valid session token. They are stored unencrypted on the `api_snapshots`
volume.

## Recommended baseline

- Set private `POSTGRES_PASSWORD`, `MTX_MEDIA_PASSWORD`, and
  `STORAGE_PASSWORD` values; set `SECRET_KEY` or back up `api_keys`.
- Terminate TLS in a reverse proxy in front of Studio and the API, set
  `AUTH_COOKIE_SECURE=true`, and point the `RUNTIME_*` URLs at the public
  addresses.
- Restrict `CORS_ORIGIN_REGEX` to the Studio origin.
- Firewall the MediaMTX ports to camera and user networks.
- Keep the stack on a trusted network. Do not expose it, or the cameras
  behind it, to the public internet.

[Deployment](deployment.md) shows a reverse proxy configuration that applies
this baseline.
