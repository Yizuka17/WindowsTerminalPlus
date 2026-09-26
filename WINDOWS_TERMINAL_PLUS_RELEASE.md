# WindowsTerminalPlus 编译与发布流程

本文是 WindowsTerminalPlus 的正式发布清单。任何对话、自动化任务或维护者在构建、安装或发布新版本前，都应先完整阅读本文。目标是保证 Git 提交、标签、MSIX、签名、本机安装和 GitHub Release 指向同一份源码。

## 发布原则

- 不从包含未提交源码改动的工作区发布。
- 产品功能、测试基线修复和发布元数据尽量拆成独立提交。
- `artifacts/`、解包目录、测试日志、PFX 和私钥不得提交。
- 发布者固定为 `CN=17yizuka`，包身份固定为 `WindowsTerminalPlus`。
- 正式标签格式为 `windows-terminal-plus-vX.Y.Z`，清单版本格式为 `X.Y.Z.0`。
- 任何源码修正都必须重新构建、重新签名并重新验证；不得复用修正前的 MSIX。

## 1. 发布前检查

仓库通常位于 `C:\zako\WindowsTerminalPlus`，远程应为：

```text
fork   https://github.com/Yizuka17/WindowsTerminalPlus.git
origin https://github.com/microsoft/terminal.git
```

执行：

```powershell
Set-Location C:\zako\WindowsTerminalPlus
git status --short
git branch --show-current
git remote -v
git log -8 --oneline --decorate
git diff --check
```

逐个检查改动，确认没有用户文件、证书私钥、令牌、绝对个人路径或生成文件。发布提交完成后，`git status --short` 必须为空。

## 2. 版本号与发布内容

在 `src/cascadia/CascadiaPackage/Package-Plus.appxmanifest` 中递增四段版本号，例如：

```xml
Version="1.0.10.0"
```

同步准备中英文发布说明，至少包含：

- 用户可见的新功能和修复；
- 已知限制；
- 测试结果及环境性失败；
- 签名者 `CN=17yizuka`；
- MSIX 和 CER 的 SHA-256。

## 3. 编译环境

进入 Visual Studio x64 开发环境：

```powershell
. 'C:\Program Files\Microsoft Visual Studio\18\BuildTools\Common7\Tools\Launch-VsDevShell.ps1' `
  -Arch amd64 -HostArch amd64 -SkipAutomaticLocation -NoLogo
```

如果此路径不存在，先定位已安装的 Visual Studio Build Tools；不要在文档或源码中写入临时机器专用路径。

## 4. Debug 构建与测试

完整 Debug 构建：

```powershell
msbuild.exe .\OpenConsole.slnx /m `
  /p:Configuration=Debug `
  /p:Platform=x64 `
  /p:WindowsTerminalBranding=Plus `
  /p:AppxBundle=Never `
  /p:AppxPackageSigningEnabled=false `
  /v:minimal
```

运行单元测试：

```powershell
.\build\scripts\Run-Tests.ps1 '*.Unit.Tests.dll' x64 Debug
```

若 Debug 断言弹窗出现，应记录具体 DLL、测试名、文件和行号并修正根因，不要通过“忽略”断言把失败伪装成通过。与功能无关但合理的测试基线修复，应单独提交。

## 5. Release 构建与功能测试

```powershell
msbuild.exe .\OpenConsole.slnx /m `
  /p:Configuration=Release `
  /p:Platform=x64 `
  /p:WindowsTerminalBranding=Plus `
  /p:AppxBundle=Never `
  /p:AppxPackageSigningEnabled=false `
  /v:minimal
```

至少运行：

```powershell
.\bin\x64\Release\te.exe .\bin\x64\Release\winconpty.Feature.Tests.dll /logOutput:Low
.\bin\x64\Release\te.exe .\bin\x64\Release\ConHost.Feature.Tests.dll /logOutput:Low
```

UIA 测试依赖桌面语言、显示缩放、窗口位置和 WinAppDriver。若存在环境性失败，必须在发布说明中列出，不得笼统写成“全部通过”。

## 6. 提交待发布源码

再次检查差异：

```powershell
git diff --check
git status --short
git diff --stat
git diff
```

按逻辑拆分提交，例如：

```powershell
git add <测试基线修复文件>
git commit -m "Harden localized and UIA tests"

git add <产品功能与资源文件>
git commit -m "Add single-block paste mode and touch Enter action"

git add src/cascadia/CascadiaPackage/Package-Plus.appxmanifest
git commit -m "Prepare WindowsTerminalPlus 1.0.10"
```

提交后必须确认 `git status --short` 输出为空。后续签名包必须来自这个确定的 `HEAD`。

## 7. 二进制和 MSIX 签名

自签名证书的私钥仅保存在维护者证书存储中，不得导出或提交 PFX。查找证书和 Windows SDK：

```powershell
$releaseCert = Get-ChildItem Cert:\CurrentUser\My |
  Where-Object { $_.Subject -eq 'CN=17yizuka' -and $_.HasPrivateKey } |
  Sort-Object NotAfter -Descending |
  Select-Object -First 1

$windowsSdk = Get-ChildItem 'C:\Program Files (x86)\Windows Kits\10\bin' -Directory |
  Sort-Object Name -Descending |
  Where-Object { Test-Path (Join-Path $_.FullName 'x64\makeappx.exe') } |
  Select-Object -First 1

if (-not $releaseCert) { throw 'CN=17yizuka signing certificate with private key was not found.' }
if (-not $windowsSdk) { throw 'Windows SDK makeappx.exe was not found.' }
```

设置本次版本和路径。示例中的版本必须替换为清单里的实际版本：

```powershell
$releaseVersion = '1.0.10.0'
$unsignedMsix = Resolve-Path ".\src\cascadia\CascadiaPackage\AppPackages\CascadiaPackage_${releaseVersion}_x64_Test\CascadiaPackage_${releaseVersion}_x64.msix"
$releaseDir = Join-Path (Resolve-Path .\artifacts) "WindowsTerminalPlus-${releaseVersion}-x64"
$unpackedDir = Join-Path $releaseDir 'unpacked'
$signedMsix = Join-Path $releaseDir "WindowsTerminalPlus_${releaseVersion}_x64.msix"
$certificateFile = Join-Path $releaseDir 'WindowsTerminalPlus-17yizuka-CodeSigning.cer'
$makeAppx = Join-Path $windowsSdk.FullName 'x64\makeappx.exe'
$signTool = Join-Path $windowsSdk.FullName 'x64\signtool.exe'

if (Test-Path -LiteralPath $releaseDir) { throw "Release directory already exists: $releaseDir" }
New-Item -ItemType Directory -Path $releaseDir | Out-Null
```

解包、逐个签署包内 EXE/DLL、重新打包，再签署 MSIX：

```powershell
& $makeAppx unpack /p $unsignedMsix /d $unpackedDir /o
if ($LASTEXITCODE -ne 0) { throw 'makeappx unpack failed.' }

$payloads = Get-ChildItem -LiteralPath $unpackedDir -Recurse -File |
  Where-Object Extension -in '.exe', '.dll'

foreach ($payload in $payloads) {
  & $signTool sign /sha1 $releaseCert.Thumbprint /fd SHA256 $payload.FullName
  if ($LASTEXITCODE -ne 0) { throw "Signing failed: $($payload.FullName)" }
}

& $makeAppx pack /d $unpackedDir /p $signedMsix /o
if ($LASTEXITCODE -ne 0) { throw 'makeappx pack failed.' }

& $signTool sign /sha1 $releaseCert.Thumbprint /fd SHA256 $signedMsix
if ($LASTEXITCODE -ne 0) { throw 'MSIX signing failed.' }

Export-Certificate -Cert $releaseCert -FilePath $certificateFile -Force | Out-Null
```

## 8. 签名和哈希验证

```powershell
& $signTool verify /pa /v $signedMsix
if ($LASTEXITCODE -ne 0) { throw 'MSIX signature verification failed.' }

$invalidPayloads = $payloads | ForEach-Object {
  $signature = Get-AuthenticodeSignature -LiteralPath $_.FullName
  if ($signature.Status -ne 'Valid' -or $signature.SignerCertificate.Subject -ne 'CN=17yizuka') {
    $_.FullName
  }
}
if ($invalidPayloads) { throw "Unsigned or incorrectly signed payloads:`n$($invalidPayloads -join "`n")" }

Get-AuthenticodeSignature -LiteralPath $signedMsix |
  Select-Object Status, @{Name='Signer';Expression={$_.SignerCertificate.Subject}}
Get-FileHash -LiteralPath $signedMsix -Algorithm SHA256
Get-FileHash -LiteralPath $certificateFile -Algorithm SHA256
```

## 9. 本机升级与人工冒烟测试

```powershell
Add-AppxPackage -Path $signedMsix -ForceApplicationShutdown -ErrorAction Stop
Get-AppxPackage WindowsTerminalPlus |
  Select-Object Name, Version, Publisher, Architecture, Status, InstallLocation
```

至少人工验证：

- 开始菜单图标、产品名和启动；
- 普通与管理员资源管理器菜单，管理员入口显示正确发布者；
- Win+X/默认终端行为符合当前安装模式；
- 触摸横向选择、越界滚动选择、点击取消选择；
- 长按菜单的复制、粘贴、回车和系统剪贴板；
- 多行“可编辑文本块”粘贴与原版 WT 兼容粘贴两种模式；
- 自动、深色、浅色外观及两套独立配色；
- 英文、简体中文、繁体中文新增文案不为空且不截断。

若冒烟测试失败，修正源码并从 Debug/Release 构建重新开始。不要只替换已解包目录中的文件。

## 10. 标签、推送与 GitHub Release

冒烟测试通过后，在构建该包的准确提交上创建标签：

```powershell
$releaseTag = 'windows-terminal-plus-v1.0.10'
git tag -a $releaseTag -m 'WindowsTerminalPlus 1.0.10'
git push fork HEAD:main
git push fork HEAD:codex/windows-terminal-plus-touch
git push fork $releaseTag
```

随后在 `Yizuka17/WindowsTerminalPlus` 创建 GitHub Release：

- 标签：`windows-terminal-plus-vX.Y.Z`
- 标题：`WindowsTerminalPlus X.Y.Z`
- 上传签名 MSIX 和 CER；
- 发布说明写入功能、修复、测试结果、已知限制和两个 SHA-256；
- 发布后重新打开 Releases 页面，确认新版本为 Latest 且资源可下载。

如果创建了 Pull Request，必须把 PR 链接附加到对应 Codex 任务。直接推送和 Release 不需要伪造 PR。

## 11. 最终对账

发布完成后同时核对：

```powershell
git status --short
git show --stat --oneline $releaseTag
git ls-remote --tags fork "refs/tags/$releaseTag"
Get-AppxPackage WindowsTerminalPlus | Select-Object Name, Version, Publisher, Status
```

最终状态应满足：

- 工作区干净；
- 本地 `HEAD`、远程 `main` 和发布标签指向同一提交；
- 清单版本、本机安装版本、MSIX 文件名和 Release 标题一致；
- MSIX 及所有包内 EXE/DLL 均由 `CN=17yizuka` 有效签署；
- GitHub Release 中的哈希与本地产物一致；
- `artifacts/` 和 `unpacked/` 仍未被 Git 跟踪。
