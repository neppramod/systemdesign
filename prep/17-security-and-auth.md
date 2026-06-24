# Topic 17: Security & Auth at Scale

> **Why this matters in a staff round:** Security shows up two ways. Sometimes it's the *whole*
> prompt ("design an auth system", "design Google Drive sharing"). More often it's a *cross-cut* —
> the interviewer says "okay, how do users actually authenticate to this?" or "how do you stop one
> tenant from reading another tenant's data?" and watches whether you have a real answer or hand-wave
> "we'll add auth." This doc gives you the vocabulary and the tradeoffs to do it crisply, and a couple
> of worked walkthroughs. You are **not** being tested as a security engineer — you're being tested on
> whether you design *secure-by-default* and can name the threat each control mitigates.

---

## Part A — AuthN vs AuthZ (get this straight first)

Two words people mush together. Say the distinction out loud early; it signals you know the space.

- **Authentication (AuthN)** — *who are you?* Proving identity. Password, OTP, passkey, client cert.
- **Authorization (AuthZ)** — *what are you allowed to do?* Given a known identity, can it perform this action on this resource? RBAC/ABAC/ReBAC live here.

They fail differently and they're enforced in different places. AuthN happens once per session/request at the edge or an identity provider; AuthZ happens on **every** protected operation, ideally close to the resource. A classic real-world bug is authenticating correctly and then forgetting the per-object authorization check (IDOR — "I'm logged in, so I just bump the `?docId=` and read someone else's doc").

> **Say this in the room:** "Authentication tells me the request is from Alice. Authorization tells me
> Alice can read *this specific* document. I need both, and the authorization check has to be on every
> object access, not just at login."

---

## Part B — Sessions vs Tokens

This is the bread-and-butter tradeoff. Almost every "design login" question routes through it.

### Session-based (server-side sessions)

1. User logs in with credentials.
2. Server creates a **session record** (`session_id → {userId, roles, expiry, ...}`) in a **session store**.
3. Server returns an opaque `session_id` in a cookie (`HttpOnly; Secure; SameSite`).
4. Every subsequent request carries the cookie; server looks up the session in the store to validate.

The session store is the crux. Options, in order of how I'd reach for them:
- **In-memory on the app server** — only works for a single node, or with **sticky sessions** at the LB. Sticky sessions break clean horizontal scaling and rolling deploys (node dies → sessions die). Avoid at scale.
- **Centralized store (Redis)** — the standard answer. Fast, TTL-native (set the key to expire = automatic session expiry), shared across all app nodes. This is what you say in an interview. Scale it like any Redis: replication + sharding by session_id.
- **Database** — durable but slower; fine if session volume is low.

**Why sessions are nice:** **revocation is trivial** — delete the row and the user is logged out *now*. The server is the source of truth, so you can invalidate, force-logout, or change permissions instantly.

**Why they hurt at scale:** every request hits the session store → it's on the hot path, a shared dependency, and a potential SPOF. You're trading statelessness for control.

### Token-based (JWT)

A **JWT** is a signed, self-describing token. Three base64url parts separated by dots:

```
header.payload.signature
{ "alg":"RS256","typ":"JWT" } . { "sub":"alice","role":"admin","exp":1718900000,"iss":"auth.acme" } . <sig>
```

- **Header** — algorithm + type.
- **Payload (claims)** — `sub` (subject/userId), `iss` (issuer), `aud` (audience), `exp` (expiry), `iat`, plus custom claims (roles, tenant). **Not encrypted** — base64, readable by anyone. Never put secrets in it.
- **Signature** — HMAC (`HS256`, shared secret) or asymmetric (`RS256`/`ES256`, signed with a private key, verified with the public key). Asymmetric is better at scale: the auth service holds the private key, every other service verifies with the public key and **never needs the secret**.

**The key property: stateless validation.** Any service can verify the signature and read the claims *without a round-trip to a store*. That's the whole appeal — no shared session dependency on the hot path, perfect for microservices and horizontal scale.

> **The catch you must name: revocation.** Because validation is stateless, a JWT is valid until it
> expires *no matter what*. Delete the user, change their role, detect a stolen token — the issued JWT
> keeps working. There's no row to delete. This is the single most important thing to say about JWTs.

**Solutions to the revocation problem (this is the deep-dive):**
- **Short-lived access token + long-lived refresh token.** The access token (e.g. 5–15 min) is what services validate statelessly. A separate **refresh token** (days/weeks) is presented to the auth service to mint a new access token. The refresh token *is* checked against a store, so you regain revocation — just at refresh boundaries, not per request. Damage window on a stolen access token = its short TTL.
- **Refresh-token rotation** — each refresh issues a new refresh token and invalidates the old one. If an old refresh token is reused, that's a theft signal → revoke the whole family. (Standard for SPAs/mobile.)
- **Token blacklist/denylist** — keep revoked token IDs (`jti`) in Redis until they'd expire anyway. Reintroduces a store lookup, but only for the revoke case, and the set is small (bounded by token TTL). Pragmatic hybrid.
- **Keep access tokens short** so blacklists stay tiny and the revocation window is acceptable without one.

### When to pick which

| Dimension | Server-side sessions | JWT (token) |
|---|---|---|
| State | Stateful (session store) | Stateless (self-contained) |
| Validation cost | Store lookup every request | Signature check, no lookup |
| Revocation | Instant (delete record) | Hard — TTL-bound; needs refresh/blacklist |
| Scale across services | Shared store = hot dependency/SPOF | No shared dependency; great for microservices |
| Payload visibility | Opaque id, data stays server-side | Claims readable by client (don't store secrets) |
| Size on the wire | Small cookie | Larger (claims travel every request) |
| Best fit | Single web app, need instant logout, regulated | Microservices, mobile, third-party APIs, scale-out |

> **Default I'd state:** "For a single web app where instant logout matters, I'd use server-side
> sessions in Redis. For a microservices backend or mobile/SPA clients, short-lived JWT access tokens
> + rotating refresh tokens, asymmetric signing so services verify with a public key." The mistake
> juniors make is treating JWTs as a free win — always volunteer the revocation tradeoff.

---

## Part C — OAuth 2.0 and OIDC

OAuth 2.0 is an **authorization** framework for **delegated access** ("let this app act on my behalf without my password"). **OpenID Connect (OIDC)** is a thin identity layer *on top of* OAuth 2.0 that adds **authentication** (the `id_token`). "Log in with Google" = OIDC.

**The roles:**
- **Resource Owner** — the user.
- **Client** — the app requesting access.
- **Authorization Server** — issues tokens (Google, Okta, Auth0, your own IdP).
- **Resource Server** — the API holding the protected data, validates the access token.

### Authorization Code flow with PKCE (the one to know)

PKCE ("pixie", Proof Key for Code Exchange) is mandatory for public clients (SPAs, mobile) and recommended for all. Step by step:

1. Client generates a random `code_verifier`, hashes it → `code_challenge`.
2. Client redirects user to the Authorization Server with `code_challenge` + requested **scopes**.
3. User authenticates and consents at the Authorization Server (the client never sees the password).
4. Auth Server redirects back with a short-lived **authorization code**.
5. Client exchanges the code **+ the original `code_verifier`** at the token endpoint.
6. Auth Server verifies `hash(code_verifier) == code_challenge`, then returns an **access token** (+ refresh token, + `id_token` if OIDC).

> **Why PKCE:** the authorization code travels through the browser/redirect and could be intercepted.
> Without PKCE, a stolen code can be redeemed by an attacker. With PKCE, redemption requires the
> `code_verifier` that never left the legitimate client. It replaces the client secret that public
> clients can't safely keep.

**Access token vs ID token** (people confuse these constantly):
- **Access token** — for the *Resource Server*. "Bearer this to call the API." Opaque or JWT, carries scopes. The API validates it.
- **ID token** — for the *Client*. An OIDC JWT asserting *who the user is* (`sub`, `email`, `name`). The client reads it to establish a session. **Never** send the ID token to an API as authorization.

**Scopes** = coarse-grained consent boundaries (`read:contacts`, `email`, `offline_access`). Not your full authorization model — they bound what the token *can* request; fine-grained AuthZ still happens at the resource.

> **Why you don't roll your own OAuth:** the flows have subtle, exploitable failure modes — redirect
> URI validation, state/nonce for CSRF, code replay, token leakage, PKCE. Mature IdPs have had these
> audited for a decade. "I'd use a managed IdP (Okta/Auth0/Cognito) or a battle-tested library, not
> hand-roll the token endpoint" is the senior answer.

### SSO and SAML (high level)

- **SSO** — authenticate once, access many apps. Backed by OIDC (modern) or **SAML** (enterprise legacy).
- **SAML** — XML-based, browser-redirect assertions between an **Identity Provider** (Okta, AD FS) and a **Service Provider** (your app). Enterprise B2B still runs on it. You don't need the XML details; know that it's the enterprise SSO standard, OIDC is its modern JSON/JWT successor, and "we'll support SAML for enterprise SSO" is the right thing to say for a B2B product.

---

## Part D — Authorization Models

Once you know *who*, decide *what they can do*. Three models, escalating in expressiveness.

- **RBAC (Role-Based)** — permissions attach to **roles**, users get roles. `admin`, `editor`, `viewer`. Simple, ubiquitous, easy to reason about and audit. Breaks down when you need per-resource or contextual rules → leads to "role explosion" (`editor_for_project_42`).
- **ABAC (Attribute-Based)** — decisions from **attributes** of subject, resource, action, and environment, evaluated by a policy. "Allow if `user.dept == doc.dept AND time is business hours AND request.ip is corporate`." Very flexible, expresses context (time, location, classification). Cost: policies get complex, hard to answer "who can access X?" (you'd have to evaluate every user).
- **ReBAC (Relationship-Based)** — permissions derive from a **graph of relationships** between subjects and objects. "Alice can view doc because she's an `editor` of the `folder` that `contains` it." This is **Google Zanzibar** and it's the right model for sharing/hierarchy problems (Drive, GitHub, Notion). Stores tuples `(object#relation@subject)` like `doc:readme#viewer@user:alice` and `folder:eng#editor@group:eng-team`, then answers "check(user:alice, viewer, doc:readme)" by traversing relations.

| | RBAC | ABAC | ReBAC (Zanzibar) |
|---|---|---|---|
| Decision basis | Roles | Attributes + policy rules | Relationship tuples / graph |
| Expressiveness | Low | High (contextual) | High (hierarchy/sharing) |
| "Who can access X?" | Easy | Hard (evaluate all) | Easy (reverse-index the graph) |
| Hierarchy/inheritance | Manual | Possible but verbose | Native (folder → file) |
| Failure mode | Role explosion | Policy complexity, audit | Tuple consistency, graph latency |
| Reach for it when | Most internal apps, admin tiers | Compliance/context rules | Sharing, multi-tenant, nested resources |

> **Interview move:** for a permissions/sharing prompt (Drive, Dropbox, GitHub), say "this is a
> relationship-based authorization problem — I'd model it Zanzibar-style with relation tuples and a
> check API, because inheritance through folders and group membership is the hard part, and RBAC would
> explode." That single sentence is a strong staff signal.

**Policy Enforcement Points (PEP) vs Policy Decision Points (PDP).** Separate *where you ask* from *where you decide*. The PEP (gateway, service middleware) intercepts the request and asks the PDP (a central authz service — OPA, a Zanzibar-style service) "allowed?". Centralizing the decision keeps policy consistent and auditable; the cost is a network hop on the hot path, mitigated by caching decisions and co-locating the PDP.

---

## Part E — API & Service-to-Service Security

- **API keys** — coarse identity for a *client/app* (not a user). Simple, but a bearer secret: if it leaks it's reusable. Scope them, rate-limit per key, support rotation, never embed in client-side code.
- **HMAC request signing** — client signs the request (method + path + body + timestamp) with a shared secret; server recomputes and compares. Proves authenticity **and integrity** (body wasn't tampered), and the secret never travels on the wire. Used by AWS SigV4, Stripe webhooks. Include a **timestamp + nonce** to block replay (see attacks).
- **mTLS (mutual TLS)** — both sides present certificates. The standard for **service-to-service** auth inside a mesh (Istio/Linkerd): every service has a cert from an internal CA, identity is the cert, traffic is encrypted. Great fit for zero-trust internal networks where "inside the firewall" is no longer a trust boundary.

> **Layering, out loud:** "External clients → OAuth/JWT at the gateway. Service-to-service → mTLS in
> the mesh. Webhooks/partners → HMAC-signed requests with timestamps. Each layer authenticates a
> different kind of caller."

---

## Part F — Encryption & Key Management

### In transit vs at rest

- **In transit — TLS.** Everything over the wire, *including internal hops* (don't assume the LAN is safe — that's the zero-trust point). TLS gives confidentiality + integrity + server authentication; mTLS adds client auth.
- **At rest — disk/DB/object-store encryption.** Protects stolen disks/backups. Often transparent (TDE, S3 SSE). Necessary for compliance but doesn't protect against an attacker who already has app-level access — so it's *defense in depth*, not the whole story.

### Symmetric vs asymmetric

- **Symmetric** (AES) — one shared key, fast, for bulk data. Problem: key distribution.
- **Asymmetric** (RSA/ECC) — public/private keypair, slow, for key exchange, signatures, and identity. TLS uses asymmetric to bootstrap then symmetric for the session (best of both).

### Key management

- **KMS** (AWS KMS, GCP KMS, Vault) — keys live in a hardened service / HSM and **never leave**; you call the KMS to encrypt/decrypt or to wrap keys. The app never holds the master key.
- **Envelope encryption** — encrypt data with a **data key (DEK)**; encrypt the DEK with a **key-encryption key (KEK)** held in KMS; store the encrypted DEK next to the data. To rotate, you re-wrap DEKs with a new KEK without re-encrypting terabytes of data. This is the answer to "how do you encrypt a huge dataset and still rotate keys cheaply."
- **Key rotation** — rotate on a schedule and on compromise; keep old keys long enough to decrypt old data; version your keys.
- **Secrets management (Vault)** — app credentials, DB passwords, API keys. Centralized, access-controlled, audited, ideally **dynamic short-lived secrets** (Vault mints a DB credential that expires) instead of long-lived static ones. Never commit secrets to git or bake them into images.

### End-to-end encryption (E2EE)

Only the endpoints hold keys; **the server stores ciphertext it cannot read** (Signal, WhatsApp, password managers). The design constraints are what get tested:
- Server **can't search or index** content → no server-side full-text search, no server-side content moderation, recommendations, or de-dup on plaintext.
- **Key distribution and recovery** are the hard problems — lose the key, lose the data; multi-device requires key sync.
- You trade features and operational convenience for the strongest confidentiality guarantee. Know *when* it's worth it (messaging, secrets) vs overkill (most apps want at-rest + in-transit + tight access control instead).

---

## Part G — Password Handling

If a login prompt appears, get this right; it's a cheap correctness signal.

- **Never store plaintext.** Never log it, never email it.
- **Hash with a slow, salted, memory-hard function: bcrypt, scrypt, or argon2** (argon2id is the modern pick). Per-user random **salt** (defeats rainbow tables; usually stored in the hash string itself). Bcrypt has a built-in salt + cost factor.
- **Why not SHA-256/MD5:** they're *fast*, which is exactly wrong — a GPU does billions/sec, so a leaked hash is brute-forced quickly. Password hashes must be deliberately slow/expensive (tunable work factor) so each guess costs the attacker real compute.
- **Pepper** (optional) — a secret added to all hashes, stored separately (in KMS, not the DB) so a DB leak alone isn't enough.
- Add **rate limiting + lockout/backoff** on login (Part H), and prefer MFA / passkeys (WebAuthn) where it matters.

> **One-liner:** "Argon2id with a per-user salt and a tuned work factor. SHA is for integrity, not
> passwords — it's too fast." That's the whole answer; don't overbuild it.

---

## Part H — Attacks to Design Against

You won't do a pentest in the room, but for each common attack you should name the **mechanism** and the **design-level mitigation**.

- **SQL injection** — untrusted input concatenated into a query. → **Parameterized queries / prepared statements / ORM**, least-privilege DB user, input validation. Never string-build SQL.
- **XSS (Cross-Site Scripting)** — attacker JS runs in a victim's browser via unescaped output. → **Context-aware output encoding**, **Content-Security-Policy**, `HttpOnly` cookies (so stolen-via-XSS JS can't read the session), sanitize rich input.
- **CSRF (Cross-Site Request Forgery)** — victim's browser is tricked into making an authenticated request using its cookies. → **CSRF tokens** (synchronizer/double-submit), **`SameSite` cookies**, check `Origin`/`Referer`. Note: token-in-header auth (Authorization: Bearer) is largely immune since the browser doesn't auto-attach it — a reason SPAs lean on tokens.
- **SSRF (Server-Side Request Forgery)** — attacker makes *your server* fetch an internal URL (e.g. the cloud metadata endpoint `169.254.169.254`). → **Allowlist outbound destinations**, block link-local/internal ranges, no raw user-supplied URLs to internal fetchers, use IMDSv2. Increasingly the scariest one in cloud designs — call it out.
- **Replay attacks** — attacker re-sends a valid captured request. → **Nonces + timestamps** in signed requests (reject old or already-seen nonces), short token TTLs, idempotency keys for mutations.
- **Credential stuffing / brute force** — → rate limiting, lockout/backoff, MFA, breached-password checks.

> **Tie to the edge doc:** rate limiting is your front-line abuse defense — per-IP, per-user, per-key
> token buckets at the gateway/CDN edge, plus WAF rules. It's not just for capacity; it blunts brute
> force, scraping, and credential stuffing before they reach your services.

---

## Part I — Cross-Cutting Principles (recite these)

- **Defense in depth** — layered controls; no single failure is catastrophic (edge WAF + authn + authz + at-rest encryption + audit).
- **Least privilege** — every identity (user, service, DB account, IAM role) gets the minimum it needs. Scope tokens, scope DB grants, scope IAM.
- **Secure by default** — deny by default and open explicitly; private buckets, closed ports, opt-in sharing. The default state must be the safe state.
- **Zero trust** — don't trust the network; authenticate and authorize every hop (→ mTLS internally).
- **Audit logging** — immutable, tamper-evident log of *who did what to what when*. Required for compliance and incident response. Don't log secrets/PII into it.
- **PII handling / GDPR** — data minimization (collect less), purpose limitation, encryption, **right to erasure** (design deletion in — hard with backups/derived data/E2EE), consent, breach notification.
- **Data residency** — some data must physically stay in a region (EU data in the EU). Affects sharding/replication topology — you may need per-region storage and routing. Mention it for any global system handling regulated data.

---

## Part J — Worked Walkthrough 1: Design an Authentication/Authorization System

**Requirements (drive it):** millions of users, web + mobile + third-party API clients, microservices backend, need fast logout for security, SSO for enterprise customers, p99 auth check < 10ms on the hot path.

1. **AuthN.** Username/password (argon2id + salt) plus social/enterprise login via **OIDC**; offer MFA/passkeys. An **Auth Service** owns credentials and is the OAuth2/OIDC Authorization Server (or wraps a managed IdP). Enterprise tenants → **SAML/OIDC SSO**.
2. **Tokens.** On login, issue a **short-lived (10 min) JWT access token** signed **asymmetrically (RS256)** + a **rotating refresh token** (stored, revocable). Every microservice validates the access token locally with the public key → no store lookup on the hot path → meets the latency target and scales horizontally.
3. **Revocation.** Refresh tokens are checked against Redis and rotated; reuse of an old one revokes the family. For emergency "log out now," a small `jti` **denylist in Redis** (TTL = access-token lifetime) keeps the blacklist bounded. Short access TTL keeps the worst-case exposure to minutes.
4. **AuthZ.** Roles in the JWT for coarse RBAC; fine-grained/per-resource decisions go to a central **authz service (PDP)** with **decision caching**; services act as PEPs. For sharing-style features, ReBAC (next walkthrough).
5. **Edge.** API gateway does TLS termination, token validation, and **rate limiting** (brute-force/abuse). Internal calls over **mTLS**.
6. **Secrets/keys.** Signing keys in **KMS/Vault**, rotated with key versioning (`kid` in the JWT header so verifiers pick the right public key during rotation).
7. **Failure modes (wrap-up):** auth service down → cached public keys still let services validate existing tokens (graceful degradation for reads); refresh path degrades → users keep working until access tokens expire. Redis denylist down → fail safe (treat as not-denied) or fail closed depending on risk appetite — state the choice.

## Part K — Worked Walkthrough 2: Design Google Drive Sharing Permissions

**The hard part is authorization, not storage.** Files/folders form a hierarchy; sharing is per-user, per-group, and inherited down the tree; you must answer "can Alice view this file?" in single-digit ms at huge scale, and "who has access to this file?" for the sharing UI.

1. **Model it ReBAC / Zanzibar-style.** Store **relation tuples**: `doc:readme#viewer@user:alice`, `folder:eng#editor@group:eng-team`, and a **hierarchy** tuple `doc:readme#parent@folder:eng`. Relations: `owner`, `editor`, `viewer`, with `editor` implying `viewer`, and access **inherited from parent folder**. Group membership is itself tuples (`group:eng-team#member@user:bob`).
2. **Check API.** `check(user, relation, object)` walks the relation graph: direct grant? via group membership? via parent folder (recurse up)? Returns allow/deny.
3. **Scale the reads.** This is read-heavy (every file open is a check). **Cache check results** aggressively; precompute/reverse-index for "who can access X." Zanzibar's real innovation is **Zookies / consistency tokens** — a snapshot token that guarantees a check reflects at least a given ACL version, so you never show a file to someone *after* their access was revoked (the "new enemy" problem). Mention it; it's the consistency tradeoff of the design.
4. **Enforcement.** The document service is the **PEP**; it calls the **permission service (PDP)** on every access. Never trust a client-supplied `docId` without a check (IDOR).
5. **Links / public sharing.** "Anyone with the link" = a capability token (signed, expirable, revocable), not an open object.
6. **Tradeoffs to name:** ReBAC over RBAC because folder inheritance + group sharing would cause role explosion; eventual consistency on permission propagation vs the strong-ish guarantee you need on *revocation* (hence consistency tokens); check latency mitigated by caching + denormalized reverse indexes.

> **Why this answer scores:** you (a) identified the real problem (authorization graph, not blob
> storage), (b) named the right model and a real system (Zanzibar), and (c) surfaced the subtle
> consistency issue (revocation / new-enemy) instead of hand-waving "we cache it."

---

### Self-check before the mock (answer these from memory)
- [ ] AuthN vs AuthZ in one sentence each — and where each is enforced.
- [ ] Sessions vs JWT: the core tradeoff, and the JWT revocation problem + its three fixes.
- [ ] Walk the OAuth2 authorization-code-with-PKCE flow; why PKCE exists.
- [ ] Access token vs ID token — who is each *for*?
- [ ] RBAC vs ABAC vs ReBAC — and which one for a Drive/GitHub sharing prompt, and why.
- [ ] Envelope encryption: why DEK + KEK makes key rotation cheap.
- [ ] Why hash passwords with argon2/bcrypt and not SHA-256?
- [ ] Name the mitigation for each: SQLi, XSS, CSRF, SSRF, replay.
- [ ] What does E2EE cost you at the design level?
- [ ] mTLS vs HMAC vs API key — which caller does each authenticate?
