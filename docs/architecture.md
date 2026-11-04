# Architecture

AzureVPN-CLI is a single MSIX-packaged Win32 console application. The package declares the `networkingVpnProvider` and `runFullTrust` capabilities so the executable can call `Windows.Networking.Vpn.VpnManagementAgent` directly.

The package exposes itself on PATH via `windows.appExecutionAlias` (`vpn.exe`), so installing from the Microsoft Store makes the `vpn` command available in any terminal without a separate installer.

For background on the API choices, see [internal-api-investigation.md](internal-api-investigation.md).

## Layout

```
manifests/
  AppxManifest.xml         Package identity, capabilities, and execution alias.

scripts/
  Build-Msix.ps1           Compiles vpn.exe with csc.exe and packs the MSIX.
  Install-Msix.ps1         Registers the local build (loose layout in dev mode, signed MSIX otherwise).
  Uninstall-Msix.ps1       Removes the registered package.

src/
  Program.cs               Entry point and command dispatch.
  VpnCommands.cs           Pure CLI command logic, takes IVpnAgent.
  IVpnAgent.cs             Interface, AgentResponse, VpnProfileInfo.
  VpnAgent.cs              VpnManagementAgent-backed implementation.
  vpn.csproj               Library used by tests; excludes Program.cs and VpnAgent.cs.

tests/
  CommandTests.cs          xUnit tests against a fake IVpnAgent.
  TestHelpers.cs           FakeAgent and Build helper.
```

The `vpn.csproj` is a class library so `dotnet test` can build the test project without requiring WinRT references. The packaged executable is built outside MSBuild, with `csc.exe` referencing `Windows.winmd` from the Windows 10 SDK.

## Build

```powershell
npm run build
# or
.\scripts\Build-Msix.ps1 -Version 1.0.0
```

Output:

```
out/
  vpn.exe                  Compiled binary.
  AzureVPN-CLI.msix        Unsigned package.
  package/                 Layout copy, suitable for `Add-AppxPackage -Register`.
```

## Local install

Loose layout register requires Developer Mode:

```powershell
.\scripts\Install-Msix.ps1 -RegisterLoose
```

After this the `vpn` command is available in any new terminal:

```powershell
vpn list
vpn import "$env:USERPROFILE\Desktop\my-vpn.AzureVpnProfile.xml"
vpn my-vpn connect
```

To remove:

```powershell
.\scripts\Uninstall-Msix.ps1
```

## Test

```powershell
npm test
# or
dotnet test tests/VpnCli.Tests.csproj -c Release
```

## Distribution

The Microsoft Store is the supported distribution channel. The Store re-signs the package with a Microsoft-rooted certificate and approves the `networkingVpnProvider` restricted capability as part of submission.

Once the Store listing is published, winget picks up the Store source automatically, so users can also install with:

```powershell
winget install TheWiseMonkey.AzureVPN-CLI
```

## Profile model

- Profiles must be imported into AzureVPN-CLI from a `.AzureVpnProfile.xml` file.
- Existing profiles created directly inside the Azure VPN Client app are not visible to AzureVPN-CLI unless their XML is exported and imported here.
- AzureVPN-CLI creates `VpnPlugInProfile` entries that target the Azure VPN Client plug-in package `Microsoft.AzureVpn_8wekyb3d8bbwe`. The actual VPN tunnel and authentication are still handled by the Azure VPN Client.
- A profile imported here is a separate entry from the one in the Azure VPN Client app, even with the same name: `vpn <name> status` reports only the entry created here, and says `Disconnected` when the tunnel was opened from the app. Keep only one of the two entries connected at a time.

## Azure VPN Client compatibility

Checked against Azure VPN Client 4.0.8.0 (October 2026) with an Entra ID (AAD), OpenVPN/TCP profile:

- The plug-in package family name is unchanged.
- The entry created by `vpn import` matches the one the app creates for the same profile: same plug-in, the exported XML stored as-is in `CustomConfiguration`, and `https://<fqdn>` server URIs.
- Profile XML options added in 4.x (device SSO, disabling certificate pinning) live inside that XML, so they pass through without changes here.
- For connections that do not start from its own UI, the plug-in gets the Entra ID token silently from its cached account, and shows a toast when the user has to sign in.
- `connect` and `disconnect` complete in about 2 seconds. DNS for the tunnel comes from an NRPT rule set by the plug-in, so `Get-DnsClientServerAddress` shows no DNS server on the VPN interface.
