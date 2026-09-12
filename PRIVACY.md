# Privacy Policy — Vaulted Desktop

_Last updated: 12 September 2026_

Vaulted Desktop ("Vaulted") is a desktop client for HashiCorp Vault, published
by **IronMade**.

**Vaulted has no backend.** There is no account, no sign-up, and no service
operated by the publisher for the app to talk to. It connects only to the Vault
servers you configure and, for browser-based sign-in, to the identity provider
your organisation has configured on those servers.

## What the publisher collects

**Nothing.** No telemetry, no analytics, no crash reporting, no usage
statistics, no advertising identifiers. The application contains no analytics
SDK of any kind, and no data of any description is transmitted to IronMade.

## What Vaulted stores, and where

Everything stays on your computer.

| What | Where | Notes |
| --- | --- | --- |
| Server profiles — address, namespace, TLS settings, chosen auth method | A configuration file in your user profile | Never contains a token or any secret |
| Vault token | Windows Credential Manager (macOS Keychain on macOS) | Only if you tick "Remember in OS keychain"; deleted when you choose "Log out & forget token" or delete the profile |
| Secret values you view | Process memory only | Wiped when the session ends, locks, or the app closes. Never written to disk |
| Search index | Process memory only | Never written to disk; discarded when you sign out |
| Preferences — appearance, idle-lock timeout | The same local configuration file | Contains no personal data |
| Your licence | Held by Windows, not by Vaulted | Vaulted stores no licence key, serial or receipt of its own |

Uninstalling Vaulted removes the application. Remove a remembered token
beforehand with **Log out & forget token**, or delete the entry from Windows
Credential Manager afterwards.

## Where Vaulted connects

1. **The Vault servers you configure.** Your infrastructure, addressed by you.
   Vaulted sends your credentials or token there to authenticate, and requests
   the secrets you ask for. Connections use TLS with certificate verification
   on by default.
2. **Your identity provider**, if you sign in with OIDC/SSO. Your system
   browser opens the provider your Vault server nominates, and Vaulted listens
   briefly on a local address (`127.0.0.1`) to receive the redirect back. That
   listener accepts only that one response and is not reachable from outside
   your computer.
3. **The update service for your platform.** On Windows, updates are delivered
   by the Microsoft Store under its own terms. On macOS, Vaulted checks a
   public GitHub release feed. Update checks reveal nothing about you beyond
   what any download request reveals — they carry no identifier and no
   information about your servers or secrets.
4. **The Microsoft Store licence service**, on Windows only. Vaulted is sold
   as a 15-day trial followed by a one-time purchase, and it asks Windows
   whether this copy is trialling, purchased, or expired. That question goes
   to Microsoft, under their terms, and never to IronMade — we receive no
   notification that you installed it, tried it, or bought it beyond the sales
   figures Partner Center reports to any publisher. Buying opens Microsoft's
   own purchase flow; Vaulted never sees your payment details, and there is no
   account to create with us.

There are no other network connections.

## Secret material

Vaulted is a tool for handling credentials, so to be explicit: **the secrets
you read through Vaulted travel only between your computer and the Vault
servers you configured.** They are never sent to IronMade, never written to
disk by Vaulted, and never included in any report.

Two consequences worth knowing:

- **Copying a secret** puts it on your system clipboard. Vaulted clears it
  automatically after 30 seconds if it has not been replaced, but operating
  system clipboard history features (Windows `Win+V`, third-party clipboard
  managers) may retain it beyond that. That is outside Vaulted's control.
- **Deep content search** reads every secret in a mount to build a searchable
  index in memory. This produces one entry per secret in your Vault server's
  own audit log — kept by your organisation, not by us. Vaulted asks for
  explicit confirmation before doing it.

## Children

Vaulted is a tool for infrastructure administrators and is not directed at
children.

## Changes

Material changes to this policy will be published here with an updated date.
Because Vaulted collects nothing, changes are expected to be rare.

## Contact

Questions about privacy, or anything else: **support@ironmade.site**.

You can also open an issue at
<https://github.com/WhisKeySwitch/vaulted-releases/issues>, but GitHub issues
are public. Email is the right channel for anything you would rather not have
indexed, and in either case please do not include secrets, tokens or server
addresses.
