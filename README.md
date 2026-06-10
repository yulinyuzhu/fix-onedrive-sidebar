# OneDrive 侧边栏无法打开 — 修复指南

## 症状

资源管理器左侧导航栏中 **OneDrive 条目点击无反应**，不弹出任何窗口或错误。托盘图标正常、OneDrive 进程运行中、已登录。

## 根因

OneDrive 左侧栏是一个 Shell Namespace Extension，依赖两组注册表：

| 注册表路径 | 作用 |
|---|---|
| `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer\Desktop\NameSpace\{018D5C66-4533-4307-9B53-224DE2ED1FE6}` | 声明「侧边栏显示此项」 |
| `HKLM\SOFTWARE\Classes\CLSID\{018D5C66-4533-4307-9B53-224DE2ED1FE6}` | 该项的完整处理程序（图标、Shell 处理器、目标文件夹） |

**两者缺一即无响应。** 常见缺失原因：
- OneDrive 被卸载后重装（卸载工具清理了 HKLM 级注册）
- Windows 更新导致注册表回退
- 注册表清理工具误删
- 安装不完整 / 权限不足

## 诊断步骤

### 1. 确认 OneDrive 进程和登录正常

```powershell
Get-Process -Name "OneDrive" -ErrorAction SilentlyContinue
```

无输出则 OneDrive 未运行，先执行：

```powershell
Start-Process "$env:LOCALAPPDATA\Microsoft\OneDrive\OneDrive.exe"
```

### 2. 检查 Desktop\NameSpace（步骤 a）

```powershell
Test-Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer\Desktop\NameSpace\{018D5C66-4533-4307-9B53-224DE2ED1FE6}"
```

返回 `False` 则缺失，跳至修复步骤 A。

### 3. 检查 CLSID 注册（步骤 b）

```powershell
Test-Path "HKLM:\SOFTWARE\Classes\CLSID\{018D5C66-4533-4307-9B53-224DE2ED1FE6}"
```

返回 `False` 则缺失，跳至修复步骤 B。

### 4. 若步骤 2、3 均返回 True 但问题依旧

检查 CLSID 的子键是否完整：

```powershell
$clsid = '{018D5C66-4533-4307-9B53-224DE2ED1FE6}'
'DefaultIcon', 'InProcServer32', 'Instance', 'Instance\InitPropertyBag', 'ShellFolder' | ForEach-Object {
    $path = "HKLM:\SOFTWARE\Classes\CLSID\$clsid\$_"
    "$_ : $(Test-Path $path)"
}
```

任何子键缺失则跳至修复步骤 B（脚本会自动补全缺失项）。

## 修复步骤

**以下所有操作需要管理员权限。**

### 修复步骤 A：补全 Desktop\NameSpace

```powershell
$clsid = '{018D5C66-4533-4307-9B53-224DE2ED1FE6}'
$nsPath = "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer\Desktop\NameSpace\$clsid"
New-Item -Path $nsPath -Force | Out-Null
Set-ItemProperty -Path $nsPath -Name "(default)" -Value "OneDrive" -Force
```

### 修复步骤 B：补全/重建 CLSID 注册

先获取当前用户的 OneDrive.exe 实际路径：

```powershell
$oneDriveExe = "$env:LOCALAPPDATA\Microsoft\OneDrive\OneDrive.exe"
```

然后执行以下完整注册（不依赖写死的用户名，自动适配）：

```powershell
$clsid = '{018D5C66-4533-4307-9B53-224DE2ED1FE6}'
$base = "HKLM:\SOFTWARE\Classes\CLSID\$clsid"

# 主键
New-Item -Path $base -Force | Out-Null
Set-ItemProperty -Path $base -Name "(default)" -Value "OneDrive" -Force

# DefaultIcon — 使用当前用户的 OneDrive.exe
$di = "$base\DefaultIcon"
New-Item -Path $di -Force | Out-Null
Set-ItemProperty -Path $di -Name "(default)" -Value "$oneDriveExe,5" -Force

# InProcServer32
$ips = "$base\InProcServer32"
New-Item -Path $ips -Force | Out-Null
Set-ItemProperty -Path $ips -Name "(default)" -Value "C:\WINDOWS\system32\shell32.dll" -Force

# Instance
$inst = "$base\Instance"
New-Item -Path $inst -Force | Out-Null
Set-ItemProperty -Path $inst -Name "CLSID" -Value "{0E5AAE11-A475-4c5b-AB00-C66DE400274E}" -Force

# Instance\InitPropertyBag
$ipb = "$inst\InitPropertyBag"
New-Item -Path $ipb -Force | Out-Null
New-ItemProperty -Path $ipb -Name "Attributes" -Value 17 -PropertyType DWord -Force | Out-Null
Set-ItemProperty -Path $ipb -Name "TargetKnownFolder" -Value "{A52BBA46-E9E1-435f-B3D9-28DAA648C0F6}" -Force

# ShellFolder
$sf = "$base\ShellFolder"
New-Item -Path $sf -Force | Out-Null
New-ItemProperty -Path $sf -Name "Attributes" -Value 4034920525 -PropertyType DWord -Force | Out-Null
New-ItemProperty -Path $sf -Name "FolderValueFlags" -Value 40 -PropertyType DWord -Force | Out-Null
```

### 修复步骤 C：重启 Explorer 使变更生效

```powershell
Stop-Process -Name "explorer" -Force
Start-Process "explorer.exe"
```

等待几秒桌面恢复，点击左侧栏 OneDrive 验证。

## 修复步骤速查（一步到位版）

将诊断和修复合并为一个脚本，执行前确认以管理员身份运行：

```powershell
$clsid = '{018D5C66-4533-4307-9B53-224DE2ED1FE6}'
$hklmNS = "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer\Desktop\NameSpace\$clsid"
$hklmCLS = "HKLM:\SOFTWARE\Classes\CLSID\$clsid"

# Step A: Desktop\NameSpace
if (-not (Test-Path $hklmNS)) {
    New-Item -Path $hklmNS -Force | Out-Null
    Set-ItemProperty -Path $hklmNS -Name "(default)" -Value "OneDrive" -Force
    Write-Host "[FIX] Added HKLM Desktop\NameSpace"
}

# Step B: CLSID
$odExe = "$env:LOCALAPPDATA\Microsoft\OneDrive\OneDrive.exe"
New-Item -Path $hklmCLS -Force | Out-Null
Set-ItemProperty -Path $hklmCLS -Name "(default)" -Value "OneDrive" -Force

New-Item -Path "$hklmCLS\DefaultIcon" -Force | Out-Null
Set-ItemProperty -Path "$hklmCLS\DefaultIcon" -Name "(default)" -Value "$odExe,5" -Force

New-Item -Path "$hklmCLS\InProcServer32" -Force | Out-Null
Set-ItemProperty -Path "$hklmCLS\InProcServer32" -Name "(default)" -Value "C:\WINDOWS\system32\shell32.dll" -Force

New-Item -Path "$hklmCLS\Instance" -Force | Out-Null
Set-ItemProperty -Path "$hklmCLS\Instance" -Name "CLSID" -Value "{0E5AAE11-A475-4c5b-AB00-C66DE400274E}" -Force

New-Item -Path "$hklmCLS\Instance\InitPropertyBag" -Force | Out-Null
New-ItemProperty -Path "$hklmCLS\Instance\InitPropertyBag" -Name "Attributes" -Value 17 -PropertyType DWord -Force | Out-Null
Set-ItemProperty -Path "$hklmCLS\Instance\InitPropertyBag" -Name "TargetKnownFolder" -Value "{A52BBA46-E9E1-435f-B3D9-28DAA648C0F6}" -Force

New-Item -Path "$hklmCLS\ShellFolder" -Force | Out-Null
New-ItemProperty -Path "$hklmCLS\ShellFolder" -Name "Attributes" -Value 4034920525 -PropertyType DWord -Force | Out-Null
New-ItemProperty -Path "$hklmCLS\ShellFolder" -Name "FolderValueFlags" -Value 40 -PropertyType DWord -Force | Out-Null

Write-Host "[FIX] CLSID registration complete"

# Step C: Restart Explorer
Stop-Process -Name "explorer" -Force
Start-Process "explorer.exe"
Write-Host "[DONE] Explorer restarted — sidebar should work now"
```

## 注册表结构参考

正常修复完成后，以下路径应存在（镜像 HKCU 中的同名条目）：

```
HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer\Desktop\NameSpace\
  {018D5C66-4533-4307-9B53-224DE2ED1FE6}             (default) = "OneDrive"

HKLM\SOFTWARE\Classes\CLSID\
  {018D5C66-4533-4307-9B53-224DE2ED1FE6}             (default) = "OneDrive"
    DefaultIcon                                       (default) = "<用户AppData路径>\OneDrive.exe,5"
    InProcServer32                                    (default) = "C:\WINDOWS\system32\shell32.dll"
    Instance                                          CLSID     = "{0E5AAE11-A475-4c5b-AB00-C66DE400274E}"
      InitPropertyBag                                 Attributes        = 17 (DWORD)
                                                      TargetKnownFolder = "{A52BBA46-E9E1-435f-B3D9-28DAA648C0F6}"
    ShellFolder                                       Attributes        = 4034920525 (DWORD)
                                                      FolderValueFlags  = 40 (DWORD)
```
