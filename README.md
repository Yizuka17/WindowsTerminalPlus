# WindowsTerminalPlus

[English](README.md) | [简体中文](README.zh-CN.md)

[![Latest release](https://img.shields.io/github/v/release/Yizuka17/WindowsTerminalPlus?display_name=tag&sort=semver)](https://github.com/Yizuka17/WindowsTerminalPlus/releases/latest)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Platform](https://img.shields.io/badge/platform-Windows%2011-0078D4.svg)](https://www.microsoft.com/windows/windows-11)

WindowsTerminalPlus is an independently built fork of [Microsoft Windows Terminal](https://github.com/microsoft/terminal), maintained by **17yizuka**. It focuses on touch-first terminal interaction, theme-aware profile colors, and practical Windows 11 integration.

> [!IMPORTANT]
> WindowsTerminalPlus is not an official Microsoft product and is not published or signed by Microsoft. Release packages are built from this repository and signed with the included self-signed `CN=17yizuka` certificate.

## Highlights

### Touch-optimized text interaction

- Swipe horizontally on terminal text to begin selection.
- Keep dragging in any direction to extend the selection, similar to holding the left mouse button.
- Drag beyond the top or bottom edge to scroll while extending the selection.
- Tap terminal text after selecting to clear the selection.
- Long-press selected text to open the copy menu.
- Long-press unselected text to open the shared context menu for actions such as paste.
- Normal vertical touch scrolling remains available.

### Independent dark and light appearance

Each profile can configure three separate settings:

- Appearance mode: Automatic, Dark, or Light.
- Dark color scheme.
- Light color scheme.

Dark and light modes also keep independent overrides for foreground, background, selection background, and cursor color. Automatic mode follows the application's current light/dark theme.

### Windows 11 integration

- Separate package identity: `WindowsTerminalPlus`.
- Stable command alias: `wtp.exe`.
- Formal non-development app icon and branding.
- Explorer context menu entries for both normal and elevated launch:
  - **Open in Terminal**
  - **Open in Terminal (Administrator)**
- The administrator command opens the selected directory and explicitly triggers UAC.

## Download and install

WindowsTerminalPlus currently ships as an x64 MSIX package.

Choose the installation mode before continuing:

- **Standalone use:** Microsoft Windows Terminal may remain installed, but launch Plus explicitly with `wtp.exe`. Win+X and other system terminal entry points may continue to open Microsoft's package.
- **Win+X/system terminal use:** uninstall Microsoft Windows Terminal first. Keeping both packages installed causes terminal-host registration and the `wt.exe` alias to compete, so Windows may continue routing system entry points to the Microsoft package.

1. Open the [latest release](https://github.com/Yizuka17/WindowsTerminalPlus/releases/latest).
2. Download both files:
   - `WindowsTerminalPlus_<version>_x64.msix`
   - `WindowsTerminalPlus-17yizuka-CodeSigning.cer`
3. Import the certificate into **Current User → Trusted People**.
4. Open the MSIX file and select **Install**.

The certificate and package can also be installed from PowerShell:

```powershell
Import-Certificate `
  -FilePath .\WindowsTerminalPlus-17yizuka-CodeSigning.cer `
  -CertStoreLocation Cert:\CurrentUser\TrustedPeople

Add-AppxPackage .\WindowsTerminalPlus_<version>_x64.msix
```

After installation, launch the app from the Start menu or run:

```powershell
wtp.exe
```

> [!CAUTION]
> Trusting a certificate allows packages signed by that certificate to install. Only use the certificate downloaded from this repository's release page and verify that the expected publisher is `CN=17yizuka`.

## Update and uninstall

Install a newer signed MSIX directly over the existing package to update it. Settings are retained by the package identity.

To uninstall:

```powershell
Get-AppxPackage WindowsTerminalPlus | Remove-AppxPackage
```

## Build from source

The project uses the same toolchain and dependencies as upstream Windows Terminal. See [upstream build guidance](doc/building.md) for prerequisites, including Visual Studio with C++ and WinUI workloads and the matching Windows SDK.

Example x64 Release build:

```powershell
. 'C:\Program Files\Microsoft Visual Studio\18\BuildTools\Common7\Tools\Launch-VsDevShell.ps1' `
  -Arch amd64 -HostArch amd64 -SkipAutomaticLocation -NoLogo

msbuild.exe .\OpenConsole.slnx /m `
  /p:Configuration=Release `
  /p:Platform=x64 `
  /p:WindowsTerminalBranding=Plus `
  /p:AppxBundle=Never `
  /p:AppxPackageSigningEnabled=false
```

The public release certificate does not contain the private signing key. Reproducing an identical signed release therefore requires your own code-signing certificate, but the source and unsigned build remain reproducible.

## Compatibility notes

- Windows 10 version 2004 (build 19041) or later is required; Windows 11 is the primary target.
- Side-by-side installation is supported only for independent use through `wtp.exe`; it is not a reliable configuration for system integration.
- To route Win+X, `wt.exe`, and Windows terminal-host entry points to Plus, uninstall Microsoft Windows Terminal before installing WindowsTerminalPlus. A Microsoft Store or Windows update may reinstall the official package and reclaim those registrations.
- Windows default-terminal and other system-owned behavior is ultimately controlled by Windows. WindowsTerminalPlus does not replace Microsoft-signed system files.
- The included certificate is self-signed. UAC and package installation identify the publisher as `17yizuka` only after the certificate is trusted.

## Upstream and attribution

WindowsTerminalPlus is derived from the open-source [Microsoft Terminal repository](https://github.com/microsoft/terminal). The terminal, console host, renderer, settings system, and most supporting components are the work of Microsoft and community contributors.

Product-specific modifications in this fork are maintained by **17yizuka**. Please report WindowsTerminalPlus-specific issues in this repository. For issues reproducible in unmodified Windows Terminal, use the upstream project.

## License

This repository remains licensed under the [MIT License](LICENSE). Microsoft trademarks and branding remain the property of Microsoft Corporation.
