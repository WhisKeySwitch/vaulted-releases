# Security Model

Vaulted handles secret material, so its security properties are design
constraints, not features. This document states the trust model and the known,
accepted limitations honestly.

## Trust model

**Rust is the security boundary.** All communication with Vault happens in the
Rust backend. The webview (React/TypeScript UI) never opens a connection to
Vault and never persists secrets — it calls Tauri commands and receives only
the display-safe data it needs. This keeps tokens and secret material out of
the browser context (no XSS-reachable secrets, no secrets in devtools/JS heap
dumps beyond what is transiently displayed).

**Tokens and secrets are minimized in memory.** The Vault token and secret
material are held only in the Rust process and wrapped so they are **zeroized**
on logout, session expiry, profile switch, and drop. Credentials submitted to
a login (password, secret_id) exist only for the duration of the exchange and
are then zeroized; only the resulting token may be persisted.

**Only the token is persisted, in the OS keychain.** When you choose to
remember a session, the Vault token is stored in the macOS Keychain / Windows
Credential Manager, keyed per profile. Non-sensitive profile config (name,
address, namespace, TLS settings, default auth method) lives in a plain config
file. **Passwords, secret_ids, and secret values are never written to disk.**

**TLS verification is on by default.** Connections verify certificates and
honor a custom CA. Disabling verification ("skip-verify") is an explicit,
per-profile opt-in that the UI marks with a loud warning.

**Secret display is consistent and guarded.** Every secret value in the app —
KV fields, transit plaintext, PKI private keys, database passwords — is shown
through one component that masks by default, reveals only on explicit action,
and copies via a backend clipboard command that **auto-clears after ~30
seconds**. For KV fields, "copy" re-reads the value in the backend so it need
not pass through the webview at all.

**Locked-down webview.** A restrictive Content Security Policy; developer tools
compiled out of release builds; the webview is granted only the minimal set of
plugin permissions it needs (notably, it has no direct clipboard permission —
clipboard writes go through the backend).

**Updates are signature-verified.** Auto-update bundles are signed with a
minisign key; the app verifies each update against a public key baked into the
binary and refuses anything that doesn't verify. This makes updates safe even
though the team builds are not (yet) OS-code-signed — integrity rests on the
minisign signature, not on the transport or OS signing. See
[RELEASING.md](./RELEASING.md) for the signing/release process and key
handling.

## Accepted limitations

These are known and deliberate, documented so they aren't surprises:

- **In-memory content index holds plaintext.** Deep content search builds an
  in-RAM index (tantivy) of secret values so they can be searched. Those
  values sit in process memory for the session and cannot be zeroized
  field-by-field (same class as any decoded JSON value, at larger scale). The
  index is never written to disk and is dropped on logout/profile switch. It
  is built only after an explicit consent dialog.
- **Deep content search generates audit-log volume.** Indexing reads every
  secret in a mount — one Vault audit-log entry per secret. This is why it is
  an explicit, consent-gated action rather than automatic. Path search (LIST
  only) does not read secret values.
- **Clipboard history.** Auto-clear wipes the live clipboard, but OS-level
  clipboard *history* features (macOS clipboard managers, Windows Win+V) may
  retain a copied secret beyond the clear. Be mindful when copying on a
  machine with clipboard history enabled.
- **Dynamic credentials outlive the panel.** A generated database credential's
  lease stays valid in Vault after you close the panel, until its TTL or an
  explicit revoke — this matches Vault's dynamic-secret semantics.
- **OS code signing / notarization is deferred.** Team builds are not
  Apple-notarized or Windows-Authenticode-signed yet, so first launch needs a
  Gatekeeper bypass (right-click → Open on macOS). Update integrity is
  independent of this (minisign). Notarization is planned for public
  distribution.

## Reporting

For a real deployment, report suspected vulnerabilities privately to the
maintainers rather than via public issues.
