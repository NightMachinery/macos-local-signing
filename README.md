# macos-local-signing

Sign the apps you build for yourself on macOS with a stable, self-signed
certificate, so that **macOS privacy permissions survive rebuilds**, and
without macOS asking for your password while it signs.

## Why

An app signed ad hoc (what most local builds get) is identified by the hash
of that exact build. macOS keys privacy grants such as Microphone,
Accessibility and Full Disk Access to that identity, so every rebuild looks
like a new app and asks again. Signed with a certificate, the app is
identified by its bundle id and the certificate instead, and neither changes
between builds.

The usual way to make such a certificate puts its key in the login keychain.
Then macOS asks for your login password each time `codesign` uses the key:
once per signed file, which is dozens for an Electron or Tauri app, and an
unattended build cannot answer at all. This tool keeps the certificate and key
as a file, readable only by you, and signs from a throwaway keychain whose
password it knows. Nothing prompts.

It does not make the app look like it comes from an identified developer.
Gatekeeper and System Settings still say "unidentified developer"; only an
Apple-issued Developer ID changes that.

## Install

```sh
git clone https://github.com/NightMachinery/macos-local-signing
ln -s "$PWD/macos-local-signing/macos-local-signing" ~/.local/bin/   # any directory on PATH
```

It needs nothing beyond macOS: `/bin/zsh`, `security`, `codesign` and the
system `openssl`.

## Use

Create the identity once per Mac, **at its screen**: trusting the certificate
for code signing shows one password dialog, which macOS refuses over ssh.

```sh
macos-local-signing ensure "My App Local Signing"
```

If an identity of that name is already in your login keychain (made the
old way), `ensure` moves it into the file store instead of making a new one,
so the permissions your app already holds stay. `import NAME` does only that.

Then sign in one of three ways:

```sh
# codesign directly
macos-local-signing codesign "My App Local Signing" --force --deep My.app

# electron-builder (CSC_LINK, CSC_KEY_PASSWORD, CSC_NAME)
eval "$(macos-local-signing env "My App Local Signing" electron-builder)"

# Tauri (APPLE_CERTIFICATE, APPLE_CERTIFICATE_PASSWORD, APPLE_SIGNING_IDENTITY)
eval "$(macos-local-signing env "My App Local Signing" tauri)"
```

`macos-local-signing ready NAME` exits 0 when the identity is usable, for
build scripts that fall back to ad hoc without it. `dir NAME` prints where
it is kept: `~/.local/state/macos-local-signing/<name>/` (`identity.p12`,
`password`, `cert.pem`), or under `$MACOS_LOCAL_SIGNING_HOME`.

The first time you switch an installed app from ad hoc to the certificate, its
identity changes, so grant its permissions once more. After that they stay.
