# WindowsTerminalPlus

[English](README.md) | [简体中文](README.zh-CN.md)

[![最新版本](https://img.shields.io/github/v/release/Yizuka17/WindowsTerminalPlus?display_name=tag&sort=semver)](https://github.com/Yizuka17/WindowsTerminalPlus/releases/latest)
[![许可证](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![平台](https://img.shields.io/badge/platform-Windows%2011-0078D4.svg)](https://www.microsoft.com/windows/windows-11)

WindowsTerminalPlus 是基于 [Microsoft Windows Terminal](https://github.com/microsoft/terminal) 独立编译和维护的分支，由 **17yizuka** 开发。项目主要面向触控终端交互、深浅主题独立配色，以及更实用的 Windows 11 系统集成。

> [!IMPORTANT]
> WindowsTerminalPlus 不是微软官方产品，也不是由微软发布或签名的软件。Release 安装包由本仓库源码编译，并使用随 Release 提供的 `CN=17yizuka` 自签名证书签署。

## 主要特性

### 触控优化的文本交互

- 在终端文本上横向滑动即可开始选择。
- 进入选择状态后，只要手指不松开，就可以向任意方向拖动扩展选择，效果类似持续按住鼠标左键。
- 拖动超出终端上边界或下边界时，可以自动滚动并继续扩展选择。
- 完成选择后，单击终端文本区域即可取消选择。
- 长按已选择的文本可打开复制菜单。
- 未选择文本时长按，会打开同一套上下文菜单，可执行粘贴等操作。
- 原有的纵向触控滚动行为保持不变。

### 深色与浅色外观完全独立

每个配置文件都可以分别设置以下三个选项：

- 外观模式：自动、深色或浅色。
- 深色配色方案。
- 浅色配色方案。

深色和浅色模式还可以分别覆盖前景色、背景色、选择背景色和光标颜色，彼此不会共用。自动模式会跟随应用当前的深色或浅色主题。

### Windows 11 集成

- 独立的软件包标识：`WindowsTerminalPlus`。
- 稳定的命令别名：`wtp.exe`。
- 正式版非 Dev 图标和产品名称。
- 在文件资源管理器右键菜单中同时注册普通入口与管理员入口：
  - **在终端中打开**
  - **在终端中打开（管理员）**
- 管理员入口会继承当前目录并明确触发 UAC。

## 下载与安装

WindowsTerminalPlus 目前提供 x64 MSIX 安装包。

安装前请先确定使用方式：

- **仅独立使用：**可以保留微软 Windows Terminal，但需要通过 `wtp.exe` 明确启动 Plus；Win+X 等系统终端入口仍可能打开微软版本。
- **用于 Win+X/系统终端：**请先卸载微软 Windows Terminal。两个软件包同时存在时，终端宿主注册和 `wt.exe` 执行别名会发生竞争，Windows 可能继续把系统入口交给微软版本。

1. 打开[最新 Release](https://github.com/Yizuka17/WindowsTerminalPlus/releases/latest)。
2. 下载以下两个文件：
   - `WindowsTerminalPlus_<版本>_x64.msix`
   - `WindowsTerminalPlus-17yizuka-CodeSigning.cer`
3. 将证书导入到 **当前用户 → 受信任人**。
4. 打开 MSIX 文件并选择**安装**。

也可以使用 PowerShell 安装证书和软件包：

```powershell
Import-Certificate `
  -FilePath .\WindowsTerminalPlus-17yizuka-CodeSigning.cer `
  -CertStoreLocation Cert:\CurrentUser\TrustedPeople

Add-AppxPackage .\WindowsTerminalPlus_<版本>_x64.msix
```

安装完成后，可以从开始菜单启动，或者运行：

```powershell
wtp.exe
```

> [!CAUTION]
> 信任证书后，系统将允许安装由该证书签名的软件包。请只使用从本仓库 Release 页面下载的证书，并确认预期发布者为 `CN=17yizuka`。

## 更新与卸载

直接安装更高版本且签名一致的 MSIX 即可更新，软件包设置会保留。

卸载命令：

```powershell
Get-AppxPackage WindowsTerminalPlus | Remove-AppxPackage
```

## 从源码编译

本项目使用与上游 Windows Terminal 相同的工具链和依赖。请参阅[上游编译说明](doc/building.md)准备环境，包括带有 C++、WinUI 工作负载的 Visual Studio，以及匹配版本的 Windows SDK。

x64 Release 编译示例：

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

公开提供的 Release 证书不包含私钥，因此如需重新制作签名包，需要使用自己的代码签名证书；源码和未签名构建本身仍然可以复现。

维护者和自动化任务在安装或发布新版本前，必须遵循 [WindowsTerminalPlus 编译与发布流程](WINDOWS_TERMINAL_PLUS_RELEASE.md)。

## 兼容性说明

- 最低需要 Windows 10 版本 2004（内部版本 19041），主要目标平台为 Windows 11。
- 只有在通过 `wtp.exe` 独立使用时，才适合与微软 Windows Terminal 并存；并存不是可靠的系统接管配置。
- 如果要让 Win+X、`wt.exe` 和 Windows 终端宿主入口指向 Plus，应先卸载微软 Windows Terminal，再安装 WindowsTerminalPlus。Microsoft Store 或 Windows 更新可能重新安装官方版本并重新取得这些注册。
- Windows 默认终端及其他系统入口最终仍由 Windows 控制；WindowsTerminalPlus 不会替换微软签名的系统文件。
- 附带证书为自签名证书。只有在证书受信任后，UAC 和软件包安装界面才会将发布者识别为 `17yizuka`。

## 上游项目与归属

WindowsTerminalPlus 源自开源的 [Microsoft Terminal 仓库](https://github.com/microsoft/terminal)。终端、控制台宿主、渲染器、设置系统以及大部分基础组件均由微软和社区贡献者开发。

本分支中的产品特定改动由 **17yizuka** 维护。WindowsTerminalPlus 特有问题请在本仓库反馈；如果问题可以在未经修改的 Windows Terminal 中复现，请向上游项目报告。

## 许可证

本仓库继续采用 [MIT 许可证](LICENSE)。微软商标及相关品牌归 Microsoft Corporation 所有。
