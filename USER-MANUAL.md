# Vaulted — User Manual

A task-oriented guide to every feature. Throughout, remember the core rule:
**the app never sends your secrets to the webview it doesn't need to, secret
values are masked by default, and copying a secret routes through the backend
and auto-clears the clipboard.** See [SECURITY.md](./SECURITY.md) for why.

---

## 1. Server profiles

A **profile** is a saved connection to one Vault cluster.

- **Add a server** — click **+ Add** in the Servers sidebar and fill in:
  - **Name** — a label (e.g. `prod-cluster`).
  - **Address** — the Vault URL (e.g. `https://vault.example.com:8200`).
    Prefilled with `https://`; a common mistake is pointing `https://` at a
    plain-HTTP dev server — the app detects that and tells you to use
    `http://`.
  - **Namespace** *(optional)* — applied to every request (Vault Enterprise).
  - **CA certificate** *(optional)* — a custom CA to trust. **Choose file…**
    opens the system file dialog; the certificate itself (not its path) is
    stored with the profile, so moving or deleting the original file later
    changes nothing. A profile saved by an older version that pointed at a
    file the app can no longer read shows *"could not be read — choose it
    again"*; it will not connect with a custom CA until you do.
  - **Skip TLS verification** — off by default. Turning it on shows a loud
    red warning and marks the profile; use it only against local dev servers.
  - **Default auth method** — which login form appears first for this profile.
- **Switch servers** — click a profile. This **ends the previous session**
  (its token is wiped from memory and its renewal timer cancelled) before
  connecting.
- **Edit / delete** — hover a profile row for **✎** (edit, pre-filled) and
  **✕** (delete). Deleting also removes that profile's remembered token from
  the OS keychain, and disconnects if it's the active one.

---

## 2. Signing in

After selecting a profile you'll see the login view with a tab per method.
The profile's default method is selected first; the others are one click away.

- **Token** — paste a Vault token (`hvs.…`).
- **Userpass** — username + password.
- **LDAP** — directory username + password.
- **AppRole** — `role_id` + `secret_id` (the secret_id is masked).
- **OIDC / SSO** — click **Sign in with browser**. Your default browser opens
  the provider; after you authenticate it redirects to a one-shot local
  listener and the app completes the login. Optional **role**; a collapsed
  **Advanced** field sets a non-default auth mount.

**Remember token in keychain** — tick this (requires a saved profile) to store
the resulting Vault token in the OS keychain (macOS Keychain / Windows
Credential Manager), keyed to the profile, so re-selecting it restores the
session. Only the token is stored — never your password or secret_id.

Credentials you type exist only briefly in the backend for the exchange and
are never persisted or logged.

---

## 3. Browsing KV v2 secrets

Selecting a KV v2 mount opens the explorer:

- **Mounts pane** (left) — every KV v2 mount your token can see, with a filter
  box when the list is long. If discovery is unavailable you can type a mount
  path manually.
- **Tree pane** (middle) — expand folders (loaded lazily, one level at a
  time). Folders you lack permission for show **⛔ no access** without breaking
  the rest of the tree. **+ New** creates a secret at the mount root; hovering
  a folder reveals a **+** to create inside it.
- **Secret pane** (right) — the selected secret's fields.

**Reading a secret**

- Each field is **masked** by default. Click **reveal** to show it, **copy**
  to put it on the clipboard without revealing it (the value is re-read in the
  backend, so it never passes through the page just to be copied). The
  clipboard auto-clears after ~30 seconds.
- **JSON view** shows the whole secret as JSON (reveals all values).
- **Version history** lists every version with created time and
  deleted/destroyed badges; click a version to view it.

**Editing**

- **Edit (new version)** opens the key/value editor (or a raw **JSON mode**).
  Non-string values you don't touch are preserved exactly.
- Saving uses **check-and-set**: if someone else changed the secret since you
  loaded it, the save is rejected as a conflict and you're prompted to reload
  and re-apply — no silent overwrite.
- **Create** uses `cas=0`, so creating over an existing path is refused.

**Delete lifecycle**

- **Delete latest** / per-version **delete** — soft-delete (recoverable).
- **undelete** — restore a soft-deleted version.
- **destroy** — permanently remove a version's data (typed confirmation
  required).
- **Delete secret…** — remove the secret and all versions
  (`delete-metadata`, typed confirmation required).

---

## 4. Search

The tree pane's search box has a **mode** toggle (paths / contents) and a
**scope** toggle (this mount / all mounts). Search is usable even before you
pick a mount — it defaults to all-mounts then.

- **Path search** — find secrets by path/name. Typing auto-crawls the mount
  (LIST only — no secret values are read), with live progress and a cancel.
  Results rank leaf-name matches first; selecting one opens the secret.
- **All-mounts path search** — crawls every visible mount in turn; results are
  tagged with their mount, and opening one switches to that mount.
- **Content search** — searches *inside* secret values. Because this **reads
  every secret in the mount** (one audit-log entry per secret) and holds their
  values in app memory, it is gated behind a **consent dialog** that states
  the cost. Results show the matching secret's path and which **key names**
  matched and whether a **value** matched — **never the value text itself**.
  Opening a result shows the secret in the normal masked viewer.
- **All-mounts content search** — one consent dialog covers indexing every
  mount; results merge across mounts, tagged by mount.

The index lives only in memory and is dropped on logout or profile switch.
A footer shows how many paths/secrets are indexed and how stale the index is,
with a re-index control.

---

## 5. Advanced engines

Non-KV mounts are badged in the mounts pane and open their own panel.

**Transit** (⚙, "encryption as a service")

- Browse keys and read a key's metadata (type, versions, capabilities) — never
  the key material.
- **Encrypt** raw text (base64 is handled for you), **Decrypt** a ciphertext,
  **Rewrap** to the latest key version, and generate a **Data key**.
- Decrypted plaintext and plaintext data keys are secret material — masked,
  reveal/copy-with-auto-clear. Binary output is shown as base64.

**PKI** (🔏, certificate authority)

- View the CA certificate/chain, browse roles and their config, and list/read
  issued certificates by serial.
- **Issue** a certificate against a role (common name + optional TTL/SANs).
  The result includes the **private key**, which Vault returns *only once* —
  copy it immediately (masked, copy-with-auto-clear). Everything else (cert,
  CA, chain, serial) is public.
- **Revoke** a certificate by serial (confirmation required).

**Database** (🛢, dynamic credentials)

- Browse connections (non-secret config) and dynamic roles.
- **Generate credentials** for a role — a fresh username/password bound to a
  Vault **lease**. The password is masked/copy-with-auto-clear.
- The credential card shows a live **lease countdown** with **Renew** and
  **Revoke**. Note: closing the panel does *not* revoke the lease — it stays
  valid in Vault until its TTL or an explicit revoke.

---

## 6. Trial and purchase

Vaulted is free for **15 days**, with nothing held back — every feature in this
manual works during the trial, and there is no sign-up. After that it is a
one-time purchase: not a subscription, and updates are included. This applies
to the Microsoft Store and Mac App Store builds; the macOS build downloaded
directly is free.

- **On Windows** the trial starts the moment you install from the Store.
- **On the Mac App Store** you start it yourself: the first launch shows what
  the trial includes, what stops when it ends and that buying is a one-time
  purchase, with **Start free trial** and **Buy Vaulted** buttons. Starting
  the trial is a free purchase, so the App Store may ask for your Apple ID.
  The trial follows your Apple ID — it is 15 days per person, not per Mac,
  and reinstalling does not restart it. **Restore Purchases** brings a licence
  bought on another Mac to this one.
- A banner across the top shows the days remaining, and sharpens in the last
  few days. **Buy Vaulted** opens the store's purchase flow inside the app;
  once it completes the banner disappears immediately, with no restart.
- Dismissing the store dialog without buying changes nothing and is not
  treated as an error.
- **When the trial ends**, Vaulted stops working until it is bought. Your Vault
  session is signed out and the token wiped from memory at that moment — the
  same thing an idle lock does — so nothing is being held. Your servers,
  settings and any remembered tokens are exactly where you left them.
- Signing out, deleting a server (which also removes its remembered token),
  changing settings, checking for updates and quitting all still work while the
  trial is over. A billing state never stands between you and ending a session
  cleanly.
- Nothing about your security changes with your entitlement: TLS verification,
  presence checks (Windows Hello / Touch ID), idle lock, zeroization and
  clipboard clearing behave identically before and after you buy. None of them is a paid feature.
- If the store cannot be reached, Vaulted carries on working. It only stops
  when it has been told plainly that the trial has ended, never because it
  could not ask.

## 7. Sessions and updates

**Two macOS builds.** Vaulted for Mac comes two ways: the **Mac App Store**
build (paid, 15-day trial, updated by the App Store) and the **direct** build
downloaded as a `.dmg` (free, updates itself). They are the same app, but
macOS keeps them apart: each has its own server profiles and remembered
tokens, so installing one next to the other does not move anything across
— add your servers again. **Settings ⚙** says which build you are running
at the bottom of the dialog.

- **Token renewal** — a renewable session auto-renews in the background at
  ~2/3 of its TTL. The header shows a live **countdown** and a state chip
  (auto-renews / won't auto-renew / renewal failed). When a session expires,
  the token is wiped and you're returned to the login view.
- **Log out** clears the session; **Log out & forget token** also removes the
  remembered token from the keychain.
- **Appearance** — **Settings ⚙** offers *Midnight* (the default dark glass),
  *Light*, and *Follow Windows*. Following Windows adopts both your light/dark
  setting and your accent colour, and tracks them while Vaulted is running —
  change either in Windows Settings and the app follows without a restart. Your
  choice is remembered.
  - A Windows accent can be any colour, so Vaulted picks the text on
    accent-coloured buttons for readability rather than assuming white. A pale
    accent gets dark text.
  - *Follow Windows* only appears where Windows exposes those settings.
- **Idle lock** — after 15 minutes without activity (configurable in
  **Settings ⚙**, including *Never*), Vaulted locks: the token is wiped from
  memory, secrets are cleared from the screen, and every Vault action is
  refused. A warning appears shortly before, and any mouse or keyboard input
  cancels it. Unlock with **Windows Hello** or **Touch ID** (a Mac without a
  sensor asks for your account password instead) — your Vault session is
  preserved, so you are not signing in again. Anything unsaved in the secret editor is
  discarded, which is what the warning is there to prevent.
  - **Renewal stops while locked**, deliberately: an unattended machine should
    not keep extending a Vault token. The locked screen counts down the token's
    remaining life, and once it runs out the lock becomes a full sign-out that
    does need a real login.
  - Search indexes **survive a lock** — unlocking does not mean re-crawling —
    but a content index still *building* when the lock hits is abandoned, and
    you are told so on return.
  - On Windows, where the machine has no fingerprint, face, or PIN set up,
    there is nothing to unlock with, so Vaulted **signs out** at the deadline
    instead of locking. The Settings dialog says which of the two your machine
    does. A Mac always locks: the account password is the fallback.
- **Opening a server with a remembered token asks for Windows Hello or Touch
  ID first.**
  This is not part of the idle-lock setting and cannot be switched off —
  setting the timeout to *Never* does not affect it. If you would rather not be
  asked, do not tick *Remember in OS keychain* when signing in; you will type
  your credentials each time instead.
- **Updates** — **Check for updates** (in the sidebar) looks for a newer
  signed release; a quiet check also runs at startup. When one is available a
  banner shows the version and notes. **Install & restart** downloads it,
  verifies its signature, and relaunches — this only happens when you click
  it, so an authenticated session is never interrupted unexpectedly. In the
  Mac App Store build, **Check for updates** opens the App Store's Updates
  page instead; the App Store delivers updates.

---

For the guarantees behind the masking, keychain, and update behavior, read
**[SECURITY.md](./SECURITY.md)**.
