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

## Idle lock and the stored token

**The threat.** Unauthorised use of the application by someone who is not its
owner — a person who reaches an unlocked machine and uses Vaulted, or the
screen, to obtain secrets they should not have.

**What is done about it.** After a configurable idle period (15 minutes by
default) the session locks: the in-memory token is zeroized, background
renewal stops, secret material is cleared from the window, and every Vault
command is refused. Unlocking requires OS user-presence verification — Windows
Hello, or Touch ID with a fallback to the account password on macOS — after
which the token is restored from the credential store. Vault is not contacted
for credentials, so unlocking is a presence check rather than a
re-authentication.

Separately and unconditionally, **reading a remembered token from the OS
credential store requires the same presence check** — when opening a server
profile, not only when unlocking. This is deliberately not a setting. Locking
would otherwise be trivially bypassable: the stored credential would remain one
click away through the profile switcher. Users who prefer not to be prompted
can decline to remember the token at sign-in.

Renewal stopping while locked is also deliberate: an unattended machine should
not keep asking Vault to extend a credential on behalf of someone who is not
there. When the token's own lifetime runs out, the lock escalates to a full
sign-out.

**What this does *not* defend against:**

- **An attacker who can read the process's memory** — a debugger, a memory
  dump, or malware running as the same user. Anything the application holds is
  available to them, and it was available before the lock too. Idle lock
  shortens the window in which token material is resident; it does not create
  a boundary against inspection of the running process.
- **A machine with no presence verifier.** Where the OS offers no enrolled
  fingerprint, face, or PIN there is nothing to unlock with, so the idle
  deadline signs out completely instead of locking. On such a machine the
  stored token also remains ungated — withholding it would remove the
  remembered-token feature and offer nothing in its place. The application
  states which behaviour applies. In practice this is a Windows case: macOS
  verifies through `LocalAuthentication` with a policy that falls back to
  the account password, so a Mac without Touch ID still has a verifier and
  still locks.

## The entitlement gate is commerce, not security

Vaulted is sold as a 15-day trial followed by a one-time purchase, and that
introduces a gate. It is worth being precise about what the gate is and is not,
because a paywall inside a credentials tool deserves more scepticism than one
inside a game.

**It is not a security boundary.** An entitlement that can be checked offline
can be defeated by anyone willing to patch a binary, and no claim to the
contrary is made here. The gate exists so the honest path is the easy one. It
is enforced in Rust rather than the webview only so that the trivial bypass —
deleting a conditional in devtools — is unavailable.

**No security control is conditioned on it.** TLS verification, presence
checks (Windows Hello / Touch ID), zeroization of secret material, clipboard
auto-clear and the
warnings shown before audit-relevant actions behave identically whether the
app is trialling, licensed or expired. None of them is a paid feature, and a
test asserts it structurally: the security-critical modules are read and must
not mention entitlement at all. That test exists because this is the promise
most likely to erode the day someone goes looking for something to put behind
the paywall.

**Doubt never locks.** The platform's store is the licence authority — the
Microsoft Store on Windows, the Mac App Store for the macOS App Store build,
through StoreKit. If it cannot be reached — a Store outage, a managed device,
a corrupted cache — the app treats itself as licensed and carries on. Expiry is concluded only from a
licence read successfully that says so. Failing closed would mean a Store
problem locking a paying customer out of their own tooling, which is a worse
outcome than the revenue it would protect.

**Locking leaves the process safer, not worse.** When the trial ends, the Vault
session is signed out and the token zeroized through the same path the idle
lock uses. Signing out, deleting a profile and its keychain entry, clearing the
clipboard and quitting cleanly all keep working while locked. A billing state
must never stand between a user and ending a session safely.

**Nothing is transmitted to the publisher.** The licence check is a local call
to the platform's store API — the Windows Store licence API, or StoreKit on
the Mac App Store build. It reaches Microsoft or Apple under their terms,
never IronMade, and the privacy policy's central claim — no backend, no
account, nothing collected — is unchanged by the introduction of commerce.
That was the reason for choosing store-managed licensing over licence keys
and a validation server, on both platforms.

**On the Mac App Store the trial is started by the user.** Apple's mechanism
for a time-limited trial in a non-subscription app is a $0 in-app purchase
("15-day Trial"), so until the user asks for it there is neither a trial nor
a licence. That state is shown as a welcome screen stating the trial's
length, what stops working when it ends and that the price is a one-time
purchase — before the trial begins, as App Store guideline 3.1.1 requires.
The trial's clock is Apple's timestamp on that transaction plus 15 days,
computed at every read; the app stores no clock of its own, so reinstalling
or clearing data does not restart it. Purchases can be restored under the
same Apple ID.

**Development and direct builds are licensed.** A build without store
identity has no licence to read, so it reports licensed rather than expired:
development builds, the archival MSIX attached to each release, and the
macOS **direct** build — which is free and distributed privately, so there is
nothing to gate. The macOS App Store build run outside the App Store (a copy
of the bundle with no App Store transaction) is likewise licensed. The
concession is deliberate: the stores are the only sales channels, and
locking developers out of their own builds would be absurd.

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
- **macOS builds are Developer ID-signed, hardened and notarized.** Release
  bundles are signed with a Developer ID Application certificate under the
  Hardened Runtime, submitted to Apple's notary service, and stapled, so
  Gatekeeper opens them without a bypass. Two signature layers protect a
  macOS build and neither depends on the other: **Apple's** covers first
  launch and tamper-evidence of the installed bundle (Gatekeeper checks it);
  **minisign** covers the update download (the updater checks it against the
  key embedded in the app). A build that fails either check is refused — the
  release pipeline will not publish an unnotarized bundle, and the updater
  will not install an unsigned one. No entitlements are granted: Touch ID,
  the keychain, and the OIDC loopback listener all work under the Hardened
  Runtime as-is.
- **There are two macOS builds, and they differ in ways a reader auditing
  one should know.** The **Mac App Store** build runs in the App Sandbox
  with the entitlements in `src-tauri/Entitlements.appstore.plist` — network
  client, a loopback *server* for the OIDC redirect only, and read access to
  user-chosen files for CA certificates — is signed with an Apple
  Distribution certificate inside a `.pkg` signed with a Mac Installer
  Distribution certificate, is updated by the App Store, and is gated by the
  StoreKit licence above. The **direct** build is not sandboxed, is Developer
  ID-signed and notarized, updates itself over the minisign-verified feed,
  and has no licence gate. The two keep separate profiles and keychain
  tokens: the App Store build is `site.ironmade.vaulted` to macOS, the
  direct build `io.space-whale.vault-client`, so they have different sandbox
  containers, keychain access groups and config paths.
  **Settings ⚙** states which build is running. A copy of the App Store
  `.pkg` is attached to each GitHub release as a record of what was
  submitted; installed outside the App Store it has no transaction to read
  and reports itself licensed — see "Development and direct builds are
  licensed" above.
- **The signing identity is an individual developer, not the organisation.**
  Gatekeeper's first-launch dialog and the notarization ticket name the
  developer personally. Moving to an organisation identity is a certificate
  swap in the maintainer's release credentials; the one visible consequence
  for users is a single macOS
  Keychain "Allow" prompt on the first launch of a build signed with a new
  identity, because keychain items are anchored to the app's code identity.
  Windows is not in this bullet — see below.

## Windows packaging and code signing

Windows ships as a **full-trust MSIX through the Microsoft Store**, which
changes three things worth stating explicitly.

**Microsoft signs the package.** Store submissions are re-signed at ingestion,
so Windows builds carry a real, chain-validating signature — there is no
Authenticode gap on Windows and no SmartScreen warning to talk users past. The
`.msix` attached to each GitHub release is the *pre-ingestion* build and is
deliberately unsigned; it exists as a record of what was submitted, not as a
download.

**The app runs full trust, not sandboxed.** An MSIX can run inside the
AppContainer sandbox, and for most applications that would be the safer
default. Vaulted declares `runFullTrust` because three of its features depend
on capabilities AppContainer removes:

| Capability | Used for | Sandboxed behaviour |
| --- | --- | --- |
| Loopback listener | receiving the OIDC redirect on `127.0.0.1` | blocked; SSO cannot complete |
| Classic Credential Manager | the `keyring` token store | unavailable; WinRT APIs only |
| Unrestricted file reads | user-supplied CA certificate paths | denied; private-CA users would have to disable TLS verification |

The honest trade-off: full trust means the process runs at medium integrity
with the user's own privileges, exactly like any ordinary desktop application,
and the sandbox is not available as a second line of defence. What it does
*not* mean is extra privilege — it is a normal user process, not an elevated
one. The verification that it is medium integrity and genuinely not an
AppContainer is automated in `src-tauri/msix/verify-package.ps1`, because
getting this wrong is invisible until a feature silently fails.

**Updates come from the Store**, not from the minisign-signed feed. The update
feed (`latest.json`) advertises macOS artifacts only, so no Windows client can
resolve a package it cannot install.

**No configuration migration is needed, and not for the reason you might
expect.** Filesystem virtualization — where a packaged app's writes are
redirected into its own private store — is an *AppContainer* behaviour. This
package runs full trust, so it does ordinary Win32 file I/O: it reads and
writes `%APPDATA%\io.space-whale.vault-client\`, exactly the path an
unpackaged build uses. Verified against the installed package, whose own
`LocalCache` tree is empty. Packaged and unpackaged builds therefore share one
set of profiles, and moving between them loses nothing.

This is a consequence of the full-trust decision rather than a property of
MSIX. If the app were ever moved into the AppContainer sandbox, configuration
*would* be virtualized, profiles would appear to vanish, and Credential
Manager tokens — which are not virtualized — would survive as orphans. That
migration would then have to be written.

## Reporting

Report suspected vulnerabilities privately to **support@ironmade.site**, not
through a public issue. A public report on a credentials tool is a disclosure
in itself, and until now this section asked for private reporting without
offering anywhere private to send it.

Please include what you did, what you expected, and what happened instead.
Do not include real tokens, secrets or server addresses — a redacted
reproduction is more useful than a real one, and safer for you.
