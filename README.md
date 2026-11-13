# NetworkManager Fortinet SSL VPN — 2FA/OTP Fix

Fix for Fortinet SSL VPN two-factor authentication (2FA/OTP) and the blocking authentication dialog in NetworkManager on Linux Mint.

This project provides patches for `network-manager-fortisslvpn`, allowing Fortinet SSL VPN connections to work with dynamically requested OTP challenges while keeping the desktop usable during authentication.

## Features

- **Dynamic 2FA/OTP support** — Handles FortiGate authentication challenges whose secret identifiers end with `_2fa`.
- **Non-modal authentication dialog** — Allows switching to other applications while the OTP prompt is open.
- **Native NetworkManager integration** — Connect using the existing Linux desktop VPN interface.
- **No proprietary FortiClient required** — Uses NetworkManager and the open-source Fortinet SSL VPN integration.

## The Problem

On some Fortinet SSL VPN configurations, the gateway requests an additional OTP after the initial username/password authentication.

The NetworkManager Fortinet plugin may reject this challenge with an error similar to:

```text
Secret 'username__gateway_2fa' is not supported
```

Additionally, the desktop's external authentication interface may display a blocking modal dialog, preventing users from switching to an authenticator application or accessing their OTP.

## The Solution

This patch introduces three changes.

### 1. Dynamic OTP Challenge Support

**File:** `src/nm-fortisslvpn-service.c`

Recognizes dynamic authentication challenge identifiers ending in `_2fa` and handles them using the plugin's existing OTP secret mechanism.

This includes the necessary adjustments to secret retrieval and secret flags.

### 2. Non-Modal GTK Authentication Dialog

**File:** `auth-dialog/main.c`

Disables GTK window modality before running the authentication dialog:

```c
gtk_window_set_modal(GTK_WINDOW(dialog), FALSE);
```

This allows users to switch applications while the authentication prompt remains open.

### 3. Disable External Authentication UI

**File:** `nm-fortisslvpn-service.name.in`

Changes:

```ini
supports-external-ui-mode=true
```

to:

```ini
supports-external-ui-mode=false
```

This makes NetworkManager use the plugin's GTK authentication dialog instead of the external desktop authentication interface.

**Both GUI changes are required for the tested configuration.**

## Tested Environment

| Component | Version / Environment |
|---|---|
| Operating system | Linux Mint 22 |
| Ubuntu base | Ubuntu 24.04 LTS (Noble) |
| NetworkManager Fortinet plugin | `1.4.0-1build2` |
| Desktop environment | Cinnamon |
| Authentication | Username/password + dynamic OTP |
| VPN protocol | Fortinet SSL VPN |

### Verified Results

- Dynamic OTP challenge accepted
- OTP submitted successfully
- GTK authentication dialog displayed instead of the fullscreen modal dialog
- Switching between applications during authentication works
- VPN connection established and remains stable

**Status: Successfully tested on the environment above.**

Compatibility with other Linux distributions, desktop environments and FortiGate configurations has not yet been verified.

## Installation

### Option A — Prebuilt Debian Packages

Prebuilt `.deb` packages may be provided under [GitHub Releases](../../releases).

Download both packages:

```text
network-manager-fortisslvpn_*.deb
network-manager-fortisslvpn-gnome_*.deb
```

Install them together:

```bash
sudo apt install ./network-manager-fortisslvpn_*.deb \
                 ./network-manager-fortisslvpn-gnome_*.deb
```

Verify the authentication UI configuration:

```bash
grep 'supports-external-ui-mode' \
  /usr/lib/NetworkManager/VPN/nm-fortisslvpn-service.name
```

Expected output:

```ini
supports-external-ui-mode=false
```

The prebuilt packages are intended for the tested Ubuntu 24.04 / Linux Mint 22 environment and `amd64` architecture.

### Option B — Build from Source

Obtain the original `network-manager-fortisslvpn` source package corresponding to version `1.4.0-1build2`.

Apply the patch from this repository:

```bash
cd network-manager-fortisslvpn-1.4.0

git apply /path/to/fortisslvpn-otp-nonmodal.patch
```

Build the Debian packages:

```bash
dpkg-buildpackage -us -uc -b
```

Install the resulting packages using APT.

**Note:** The patch must be applied to a compatible source version. Building may require additional development packages and Debian build dependencies.

## VPN Configuration

Create or edit a Fortinet SSL VPN connection through NetworkManager.

Configure your VPN gateway, port and credentials according to your organization's requirements.

For OTP-based authentication, configure the token secret so that it is requested when connecting rather than stored permanently.

The tested configuration uses:

```ini
otp-flags=2
```

This corresponds to NetworkManager's `NOT_SAVED` secret flag.

When the FortiGate requests the second authentication factor, the plugin should display a GTK dialog where the OTP can be entered.

You should be able to switch to another application to retrieve the code without closing the authentication prompt.

## Known Limitations

- Tested only on Linux Mint 22 with Cinnamon.
- The patch targets the `1.4.0-1build2` source package.
- Other FortiGate authentication flows may behave differently.
- OTP expiration and gateway-side authentication timeouts are not modified.
- Future distribution package updates may overwrite locally installed patched packages.
- The GTK dialog is intentionally non-modal, which differs from the original plugin behavior.

## Security Considerations

This project does not bypass or disable two-factor authentication.

The FortiGate continues to validate credentials and OTP codes.

Do not include private VPN configurations, passwords, OTP codes, gateway addresses or organization-specific authentication details in bug reports.

Review third-party binary packages before installing them on production systems.

## Upstream and Credits

Based on the open-source NetworkManager Fortinet SSL VPN plugin and its integration with `openfortivpn`.

This is an independent community patch and is not an official Fortinet or NetworkManager release.

All credit for the original software belongs to its respective maintainers and contributors.

## Contributing

Issues, testing reports and pull requests are welcome.

Testing on additional Linux distributions, desktop environments and FortiGate configurations would be particularly useful.

When reporting an issue, please include your distribution, desktop environment, plugin version and relevant sanitized logs.

## License

This project provides modifications to NetworkManager-fortisslvpn, which is distributed under the **GNU General Public License, version 2 or later (GPL-2.0-or-later)**.

The patches affecting `src/nm-fortisslvpn-service.c`, `auth-dialog/main.c`, and `nm-fortisslvpn-service.name.in` are distributed under the same license.

The original project's copyright notices and license terms remain applicable.

Some upstream files under `shared/` are licensed under **LGPL-2.0-or-later**, with specific exceptions documented in the original `debian/copyright` file.

This project is an independent community modification and is not affiliated with Fortinet or the NetworkManager maintainers.

## Disclaimer

This project is provided as-is, without warranty.

It is not affiliated with, endorsed by or supported by Fortinet.
