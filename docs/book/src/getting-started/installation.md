# Installation

## Package Managers

### Homebrew (macOS / Linux)

```bash
brew install HuginSecurity/tap/hugin
```

### cargo-binstall (pre-built binary)

```bash
cargo binstall hugin-bin
```

### AUR (Arch Linux)

```bash
yay -S hugin-proxy
```

### Nix

```bash
nix run github:HuginSecurity/Hugin
```

### Debian / Ubuntu

Download the `.deb` from the latest [GitHub Release](https://github.com/HuginSecurity/Hugin/releases), then:

```bash
sudo dpkg -i hugin-desktop-linux-x86_64.deb
```

## Pre-built Binaries

Download from [GitHub Releases](https://github.com/HuginSecurity/Hugin/releases). Every release includes `.sha256` checksum files and `.sig` Ed25519 signatures.

Available artifacts per platform:

- **macOS (Apple Silicon)** -- `.dmg` desktop app, `.tar.gz` CLI binary
- **macOS (Intel)** -- `.dmg` desktop app, `.tar.gz` CLI binary
- **Linux x86_64** -- `.deb`, `.AppImage`, `.tar.gz`
- **Linux AArch64** -- `.deb`, `.AppImage`, `.tar.gz`
- **Windows x86_64** -- `.zip` (desktop and CLI)

Verify a downloaded binary:

```bash
hugin verify hugin-desktop-darwin-aarch64.dmg
```

## Building from Source

Requirements: Rust 1.75+ (stable), SQLite development headers.

```bash
git clone https://github.com/HuginSecurity/Hugin.git
cd hugin
cargo build --release --bin hugin
```

The binary is at `target/release/hugin`. Copy it to a directory on your `$PATH`.

Linux builds additionally require:

```bash
# Debian / Ubuntu
sudo apt install libwebkit2gtk-4.1-dev libgtk-3-dev libayatana-appindicator3-dev

# Fedora
sudo dnf install webkit2gtk4.1-devel gtk3-devel libappindicator-gtk3-devel

# Arch
sudo pacman -S webkit2gtk-4.1 gtk3 libayatana-appindicator
```

## Platform Notes

### macOS -- Gatekeeper

Hugin ships unsigned. If macOS shows "cannot be opened because the developer cannot be verified":

- **DMG:** Right-click the app, select "Open", then click "Open" in the dialog. Or:
  ```bash
  xattr -d com.apple.quarantine /Applications/Hugin.app
  ```
- **CLI tarball:** After extracting:
  ```bash
  xattr -d com.apple.quarantine /usr/local/bin/hugin
  ```
- **Homebrew:** No action needed. Homebrew handles quarantine automatically.

### Windows -- SmartScreen

Windows may show "Windows protected your PC" for unsigned binaries. Click "More info", then "Run anyway".

## Verify Installation

```bash
hugin --version
```
