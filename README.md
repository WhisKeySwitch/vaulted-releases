# Vaulted — downloads & updates

A cross-platform desktop client for [HashiCorp Vault](https://www.vaultproject.io/),
for macOS and Windows. Natively compiled — real Apple Silicon (`arm64`) binaries,
no Rosetta.

This is the **public distribution repo**: it hosts the installers, the
auto-update manifest (`latest.json`), and the user-facing docs. The source code
lives in a separate private repository.

## Download

Get the latest installer for your platform from the
[**Releases**](https://github.com/yaremenko2205/vaulted-releases/releases/latest)
page:

| Platform | File |
| --- | --- |
| macOS (Apple Silicon) | `Vaulted_<version>_aarch64.dmg` |
| macOS (Intel) | `Vaulted_<version>_x64.dmg` |
| Windows | `Vaulted_<version>_x64-setup.exe` or `_x64_en-US.msi` |

The app **auto-updates**: installed builds check this repo's `latest.json`,
verify each update's signature against a minisign public key baked into the app,
and self-install. Update *integrity* does not depend on OS code signing.

## macOS: "Vaulted is damaged and can't be opened"

Builds are **not yet Apple-notarized**, so on a Mac other than the one that
built it, Gatekeeper shows a "damaged" warning when the app arrives with a
download/AirDrop/USB *quarantine* flag. It is **not** damaged — remove the flag
once, in Terminal:

```bash
xattr -dr com.apple.quarantine /Applications/Vaulted.app
```

(If it still complains, use `sudo xattr -cr /Applications/Vaulted.app`.) Then
open it normally. The lasting fix is Developer ID signing + notarization,
planned for wider distribution.

## Docs

- [User manual](./USER-MANUAL.md)
- [Security model](./SECURITY.md)
