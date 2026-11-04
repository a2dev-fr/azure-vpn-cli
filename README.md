# AzureVPN-CLI

[![CI](https://github.com/the-wise-monkey/azure-vpn-cli/actions/workflows/ci.yml/badge.svg)](https://github.com/the-wise-monkey/azure-vpn-cli/actions/workflows/ci.yml)

Command-line interface for Azure VPN profiles on Windows.

## Install

From the Microsoft Store: _coming soon_.

Or via winget once the Store listing is published:

```powershell
winget install TheWiseMonkey.AzureVPN-CLI
```

## Commands

```text
vpn import <profile.AzureVpnProfile.xml> [name]   Import a profile
vpn list                                          List your profiles
vpn <name> connect                                Connect a profile
vpn <name> disconnect                             Disconnect a profile
vpn <name> status                                 Show connection status
vpn export <name> [output.xml]                    Export a profile
vpn delete <name>                                 Remove a profile
```

## Examples

```powershell
vpn import "$env:USERPROFILE\Desktop\my-vpn.AzureVpnProfile.xml"
# Imported my-vpn

vpn list
#   my-vpn                               Disconnected

vpn my-vpn connect
# Connected

vpn my-vpn status
# Connected

vpn my-vpn disconnect
# Disconnected
```

## Requirements

- Windows 11. The package installs on Windows 10 19041 or newer, but Microsoft only supports the Azure VPN Client on Windows 11.
- [Azure VPN Client](https://apps.microsoft.com/detail/9np355qt2sqb) 4.x installed from the Microsoft Store. Tested with 4.0.8.0 (October 2026).

## Development

See [docs/architecture.md](docs/architecture.md) for build and packaging details.

## License

[MIT](LICENSE).
