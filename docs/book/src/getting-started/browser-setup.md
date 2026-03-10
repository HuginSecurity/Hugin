# Browser Setup

Hugin intercepts HTTP/HTTPS traffic by acting as a proxy. You need to (1) point your browser at the proxy and (2) trust the Hugin CA certificate.

## Quick Setup (Mullvad Browser)

Click the **Mullvad Browser** button in the Hugin toolbar. Hugin automatically:

1. Creates a temporary Firefox profile with the proxy pre-configured
2. Imports the Hugin CA certificate
3. Disables certificate pinning, HSTS preload, and HTTPS-Only Mode
4. Disables the Mullvad Browser Extension (which would override proxy settings)
5. Launches the browser in an isolated profile

No manual configuration needed. Your normal Mullvad Browser profile is untouched.

**Requirement:** `certutil` must be installed (`brew install nss` on macOS).

## Manual Setup

### Step 1: Configure the Proxy

Set your browser to use `127.0.0.1` port `8080` as its HTTP and HTTPS proxy.

#### Firefox / Mullvad Browser

Settings > Network Settings > Manual Proxy Configuration:
- HTTP Proxy: `127.0.0.1`, Port: `8080`
- Check "Also use this proxy for HTTPS"

#### Mullvad Browser -- Extra Settings

Mullvad Browser requires additional `about:config` overrides:

- `security.enterprise_roots.enabled` = `true`
- `security.cert_pinning.enforcement_level` = `0`
- `network.stricttransportsecurity.preloadlist` = `false`
- `dom.security.https_only_mode` = `false`
- `dom.security.https_only_mode_ever_enabled` = `false`
- `dom.security.https_only_mode_pbm` = `false`
- `extensions.torlauncher.start_tor` = `false`
- `extensions.installDistroAddons` = `false`

The Mullvad Browser Extension uses `browser.proxy.onRequest` to route all traffic through a SOCKS proxy, overriding your manual settings. Disable it via Add-ons > Extensions, or set `extensions.installDistroAddons` to `false`.

Mullvad Browser resets `prefs.js` on each startup. Use `user.js` in the profile directory for persistent settings.

#### Chrome / Chromium

Chrome uses system proxy settings. Alternatively, launch with a flag:

```bash
chromium --proxy-server="http://127.0.0.1:8080"
```

#### macOS System Proxy

System Settings > Network > (your adapter) > Advanced > Proxies:
- HTTP Proxy: `127.0.0.1:8080`
- HTTPS Proxy: `127.0.0.1:8080`

### Step 2: Trust the CA Certificate

Hugin generates a CA certificate at `~/.config/hugin/Hugin-Proxy-CA.pem` on first run. Your browser or OS must trust this certificate for HTTPS interception.

You can also download it from `http://127.0.0.1:8081/api/ca.pem` while Hugin is running.

#### macOS (System Keychain)

```bash
sudo security add-trusted-cert -d -r trustRoot \
  -k /Library/Keychains/System.keychain \
  ~/.config/hugin/Hugin-Proxy-CA.pem
```

#### Linux (System-wide)

```bash
sudo cp ~/.config/hugin/Hugin-Proxy-CA.pem /usr/local/share/ca-certificates/hugin-ca.crt
sudo update-ca-certificates
```

#### Windows (Administrator)

```cmd
certutil -addstore Root %USERPROFILE%\.config\hugin\Hugin-Proxy-CA.pem
```

#### Firefox (Manual Import)

Firefox uses its own certificate store, separate from the OS:

1. Settings > Privacy & Security > Certificates > View Certificates
2. Authorities tab > Import
3. Select `~/.config/hugin/Hugin-Proxy-CA.pem`
4. Check "Trust this CA to identify websites"

Or via command line with `certutil` (from NSS tools):

```bash
certutil -A -n "Hugin Proxy CA" -t "CT,C,C" \
  -i ~/.config/hugin/Hugin-Proxy-CA.pem \
  -d "sql:$(find ~/Library/Application\ Support/Firefox/Profiles -name '*.default-release' | head -1)"
```

#### Chrome on Linux

```bash
certutil -d sql:$HOME/.pki/nssdb -A -t "C,," -n "Hugin CA" \
  -i ~/.config/hugin/Hugin-Proxy-CA.pem
```

## Verifying the Setup

1. Start Hugin
2. Open the configured browser
3. Visit any HTTPS site (e.g., `https://httpbin.org/get`)
4. Check the HTTP History tab in Hugin -- you should see the request appear
5. No certificate warnings should appear in the browser

If you see certificate errors:
- Verify the CA cert is imported and trusted
- For Mullvad Browser: confirm `security.cert_pinning.enforcement_level` = `0`
- For HSTS-protected sites: confirm `network.stricttransportsecurity.preloadlist` = `false`

## Scope

By default, Hugin captures all traffic through the proxy. To limit capture to specific targets, configure scope in the Scopes view within the UI, or edit `config.toml`:

```toml
[scope]
include_hosts = ["*.example.com", "api.target.com"]
exclude_hosts = ["fonts.googleapis.com", "*.analytics.com"]
```

## Mullvad VPN Compatibility

Hugin works alongside Mullvad VPN. Browser traffic routes through the Hugin proxy on localhost, then Hugin's outbound connections go through the VPN tunnel. Enable split tunneling in the Mullvad app if you experience connectivity issues.
