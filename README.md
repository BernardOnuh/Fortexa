# Fortexa

<p align="center">
  <img src="public/fortexa-logo.jpeg" alt="Fortexa logo" width="200" />
</p>

<p align="center"><strong>Policy-Controlled Payment Firewall for Autonomous Agent Actions on Stellar</strong></p>

Fortexa is a **policy-controlled payment firewall for autonomous agent actions on Stellar**.
It sits between agent intent and economic execution, applies governance/risk checks, and keeps an auditable decision trail.

This document reflects the **current implementation** in this repository.

See [docs/SCF_TRANCHE_PLAN.md](docs/SCF_TRANCHE_PLAN.md) for the Stellar Community Fund (SCF) funding tranches and roadmap alignment.

---

## 1) ⚠️ Why This Matters

Agentic systems can now trigger real payments. That creates a new risk layer: high-speed model decisions can become high-impact economic actions.

Fortexa adds a control plane between intent and money movement:

- Policy checks before execution
- Risk scoring on suspicious behavior
- Human-approval gate for sensitive cases
- Wallet-native signed XDR flow
- Auditable evidence trail for every decision

In short: Fortexa is the safety layer for agentic payments.

---

## 2) 🚀 Jury Demo Flow (Fast Path)

If you only read one section, read this:

1. **Login with wallet** on `/login`.
2. **Evaluate action** in `/console`.
3. Receive decision: **`BLOCK` / `REQUIRE_APPROVAL` / `WARN` / `APPROVE`**.
4. For allowed flows, **build unsigned XDR → sign in wallet → submit signed XDR**.
5. Verify outcome with **Explorer link** and inspect evidence in `/activity` and `/ops`.

---

## 3) 🧭 Current Product Model

Fortexa currently runs with a strict wallet-bound model:

1. User logs in with wallet (`/login`).
2. Session is created with role (`operator` / `viewer`).
3. Session wallet is bound as execution source.
4. Actions are evaluated by policy + security engine.
5. Approved/warned decisions can proceed to signed-XDR payment flow.
6. Decision/audit evidence is stored and visible in `/activity` and `/ops`.

---

## 4) 🔐 Auth and Access Control

### 4.1 Wallet-only Login

Fortexa uses a challenge-signature login flow:

1. Client requests a one-time login challenge via `POST /api/auth/challenge` with the wallet public key (`G...`).
2. The server returns a short-lived challenge message bound to that wallet.
3. The wallet signs the challenge message (SEP-53 / Freighter `signMessage`).
4. Client posts `publicKey`, `challengeId`, and `signature` to `POST /api/auth/login`.
5. The server verifies the signature, enforces one-time challenge use + expiry, then issues `fortexa_session`.

Role is still resolved via allowlists:

- `FORTEXA_OPERATOR_WALLETS`
- `FORTEXA_VIEWER_WALLETS`

If both allowlists are empty, current behavior falls back to `operator` role for any valid wallet (recommended only for local/dev).

Session cookie: `fortexa_session` (HMAC-signed).

Challenge TTL: `FORTEXA_AUTH_CHALLENGE_TTL_SECONDS` (default `300`).

### 4.2 Role Permissions

- `operator`: full decision/policy/payment flow
- `viewer`: read-only experience on sensitive execution paths

### 4.3 Login Hardening

- Rate limiting
- Brute-force lockout (`FORTEXA_AUTH_MAX_ATTEMPTS`, `FORTEXA_AUTH_LOCK_MINUTES`)

> Note: MFA is removed from current implementation.

---

## 5) 👛 Wallet and Signing Model (Current)

Fortexa currently does **not perform server-side signing or private-key custody**.

- Session is wallet-bound at login.
- Execution source wallet is derived from session identity.
- Manual arbitrary wallet assignment in UI is removed.
- `/api/stellar/balance` auto-syncs missing wallet mapping from session when possible.

---

## 6) ⚙️ Decision and Payment Flow

### 6.1 Decisioning

- Policy engine: `src/lib/policy/engine.ts`
- Security analyzer: `src/lib/security/analyzer.ts`
- Decision engine: `src/lib/decision/engine.ts`

Decision outcomes:
- `BLOCK`
- `REQUIRE_APPROVAL`
- `WARN`
- `APPROVE`

`Human Approve & Re-run` applies only when prior result is `REQUIRE_APPROVAL`.

### 6.1a Policy Simulation (Pre-Save Safety Check)

Before committing a policy change, operators can dry-run the unsaved draft from the Policy editor (**Run simulation**). The draft is evaluated against the seeded demo scenarios — and, optionally, a small