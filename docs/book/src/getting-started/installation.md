# Installation

## Download

Download the latest release from [hugin.nu/download](https://hugin.nu/download) or from [GitHub Releases](https://github.com/HuginSecurity/Hugin/releases).

Every release includes `.sha256` checksum files and `.sig` Ed25519 signatures.

### macOS

Available for both Apple Silicon (aarch64) and Intel (x86_64):

- **Desktop app:** `.dmg` -- mount and drag to Applications
- **CLI binary:** `.tar.gz` -- extract and copy `hugin` to a directory on your `$PATH`

### Linux

Available for x86_64 and aarch64:

- **Debian/Ubuntu:** `.deb` -- install with `sudo dpkg -i hugin-desktop-linux-x86_64.deb`
- **Other distros:** `.tar.gz` -- extract and copy `hugin` to `/usr/local/bin/`

### Windows

Available for x86_64:

- **Desktop + CLI:** `.zip` -- extract and run `hugin.exe`

## Verify a Download

Every release binary is Ed25519 signed. Verify with:

```bash
hugin verify hugin-desktop-darwin-aarch64.dmg
```

This checks the `.sig` file (expected at `<file>.sig` in the same directory) against the public key embedded in the binary.

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

### Windows -- SmartScreen

Windows may show "Windows protected your PC" for unsigned binaries. Click "More info", then "Run anyway".

## Verify Installation

```bash
hugin --version
```
