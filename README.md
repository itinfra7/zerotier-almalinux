# zerotier-almalinux

ZeroTier One installation and upgrade support for AlmaLinux.

## Compatibility

- Target platform: AlmaLinux 8 or later
- Verified host: AlmaLinux 10.2, x86_64
- Package manager: DNF 4 with Python bindings
- Service manager: systemd
- Package version: latest available in the selected official RPM repository

On AlmaLinux 10.2, dependency resolution, RPM signature verification, and the extracted binary's version command were verified with `zerotier-one-1.16.2-1.el9.x86_64`. Package installation, daemon startup, network connectivity, and other AlmaLinux versions or architectures have not been tested.

## Technical Design

The script downloads the [official installer](https://install.zerotier.com/) on each run and adapts its RPM installation path for AlmaLinux. It checks official EL repositories from the highest available major version at or below the AlmaLinux major version, selecting the first whose latest package satisfies DNF dependencies for the detected architecture. Downloads use HTTPS, and RPM signature checks remain enabled.

The official installer already recognizes AlmaLinux, but uses `redhat/el/$releasever`. On AlmaLinux 10.2, DNF resolves `$releasever` to `10`. As of September 11, 2026, the [official EL repository listing](https://download.zerotier.com/redhat/el/) ends at EL 9, and the EL 10 metadata URL returns HTTP 404. This adapter selects a compatible repository without changing `/etc/os-release` or the system-wide release version.

An existing `[zerotier]` repository is disabled during dependency checks so a broken EL 10 URL does not block selection. Installation writes the selected URL to `/etc/yum.repos.d/zerotier.repo`.

## Requirements

- Root privileges
- Access to ZeroTier download servers and configured AlmaLinux repositories
- A running systemd for installation
- `/dev/net/tun` and permission to create TAP devices (`CAP_NET_ADMIN`)

For LXC and other containers, the host must provide TUN access and the required permissions. The installer checks TAP creation before installing ZeroTier.

## Install

For a one-line installation on an Internet-connected AlmaLinux host:

```bash
curl -fsSL https://github.com/itinfra7/zerotier-almalinux/releases/latest/download/zerotier-install-almalinux.sh | sudo bash
```

To run a checked-out copy instead, install its prerequisites and execute it as two separate commands:

```bash
sudo dnf install -y curl python3-dnf tar util-linux coreutils rpm
sudo bash ./zerotier-install-almalinux.sh
```

A normal run also installs missing prerequisites through DNF. It accepts the `curl` command provided by either `curl` or `curl-minimal`.

The installer enables and starts `zerotier-one.service`. Network membership is managed separately.

To select an EL repository explicitly:

```bash
sudo env ZT_EL_VERSION=9 bash ./zerotier-install-almalinux.sh
```

`ZT_EL_VERSION` defaults to `auto`. An explicit version still requires a compatible package and cannot downgrade the installed version.

## Upgrade

Check the available package, then rerun the installer:

```bash
sudo bash ./zerotier-install-almalinux.sh --check
sudo bash ./zerotier-install-almalinux.sh
```

`--check` leaves packages, repository configuration, signing keys, and the service unchanged. DNF metadata is stored temporarily and removed afterward. Missing prerequisites are reported without installing them. This mode does not require a running systemd, `tar`, or TUN access; RPM downloads and signature checks occur during installation.

A normal run restarts the service even when the package is already current, briefly interrupting connectivity. Node identity and joined networks are preserved, and the running version and existing node ID are checked after installation.

Before a package change, new RPMs are downloaded and verified. If existing state is present, it is backed up with the service stopped:

```text
/var/backups/zerotier-one/<timestamp>-<id>/state.tar.gz
/var/backups/zerotier-one/<timestamp>-<id>/previous-version
```

State backups are root-only and include secret keys. Previous RPMs are not included. Rollback is manual. If an upgrade fails after stopping a previously active service, the script attempts to start it again; packages and configuration are not automatically restored.

## Verify

```bash
sudo zerotier-cli info
sudo zerotier-cli listnetworks
systemctl is-enabled zerotier-one
systemctl status zerotier-one --no-pager
```

Check for `ONLINE` and `OK` on joined, authorized networks.

To join a network, replace `NETWORK_ID` with its actual ID:

```bash
sudo zerotier-cli join NETWORK_ID
```

Private networks also require member authorization in the controller.

## Service Management

```bash
sudo systemctl restart zerotier-one
sudo systemctl disable --now zerotier-one
sudo systemctl enable --now zerotier-one
```

## Troubleshooting

```bash
sudo journalctl -u zerotier-one -n 100 --no-pager
sudo zerotier-cli peers
```

If no compatible package is found, inspect the reported architecture and dependency errors. Unavailable candidate ZeroTier repositories are skipped; errors in configured AlmaLinux repositories must be resolved before installation can proceed.

If the service cannot start, inspect its journal and verify TUN access and TAP permissions, particularly in containers.

If the upstream patch locations change, installation stops. Update the adapter before retrying. The upstream script is authenticated through HTTPS; its PGP wrapper is parsed without independent PGP signature verification. RPM signature and TLS certificate checks remain enabled.

## License

[BSD-3-Clause](LICENSE). ZeroTier One itself is distributed under the licenses in the [ZeroTierOne repository](https://github.com/zerotier/ZeroTierOne).

## Credits

- [ZeroTier, Inc.](https://www.zerotier.com/) — [ZeroTierOne](https://github.com/zerotier/ZeroTierOne) and the [official installer](https://github.com/zerotier/install.zerotier.com)
- [AlmaLinux](https://almalinux.org/)
- AlmaLinux installer: [itinfra7 from GitHub](https://github.com/itinfra7)
