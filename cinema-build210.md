# 本机已验收案例：YVR 影院 build210

案例日期：2026-11-08。应用 `com.yvr.launcher2d` 2.9.12，Unity2022.3.5f1，arm64，目标 Quest3S。本文是查证入口，不表示头显当前始终联网或文件永远在这些路径。

## 权威资料与保留文件

| 内容 | 本机路径 |
| --- | --- |
| 最新入口 | `E:\DSH工作区\AGENTS.md` |
| 最新技术交接 | `E:\yvrwork\HANDOFF_2026-11-08.md` |
| DeepSeek 原交接，只作历史证据 | `E:\yvrwork\work\codex\HANDOFF.deepseek.original.md` |
| 用户补充的完整早期记录 | `E:\DSH工作区\yvr-port\DEEPSEEK-SESSION-HISTORY.md`，解读见 [history-corrections.md](history-corrections.md) |
| 原始镜像/原APK资料 | `E:\yvr1文件夹` |
| 解包构建树 | `E:\yvrwork\decoded` |
| loader shim | `E:\yvrwork\shim\loader_shim.c` |
| 眼位/抓握/角度/颜色/刷新率 | 同目录 `codex_eye_properties.h`、`codex_grip.h`、`codex_pose.h`、`codex_layer_color.h`、`codex_refresh.h` |
| 容器 Java/C 与构建 | `E:\yvrwork\work\codex\container` 的 `src`、`quest_egl.c`、`build_cinema.py`、`patch_app_texture.py` |
| 原 IL2CPP dump | `E:\DSH工作区\yvr-port\work\il2cpp\dump.cs` |
| 已签名影院 | `E:\yvrwork\signed\build210-aligned-debugSigned.apk` |
| 已签名 SDK 支持包 | `E:\yvrwork\sdkshare_signed\sdkshare_patched-aligned-debugSigned.apk` |
| Shizuku 原包 | `E:\yvrwork\work\codex\container\Shizuku.apk` |
| 交付副本和新增源码 ZIP | `E:\codex\2026-11-08\e-dsh-agents-md-android-6\outputs` |

build210 SHA256：

```text
4c8bc0b75387aa0d44e06696fc4ea87b9a0e581c03170376c98ccdd756a02c70
```

签名 SDK 支持包 SHA256：

```text
bc4beb20289c97364b4b7fed497b0575fa3280b05015360538fe9a21de0a72cd
```

Shizuku13.6.0 SHA256：

```text
6e273ab0e991c4e79bc8b1bbb9b9dd739ccac1a8712a541a214078886b7b790f
```

用户实测影院与本地视频正常；SDK 独立保留；巨幕二维应用通过 Shizuku；菜单/应用颜色、眼尺寸、2560×1440及120Hz已验收。非 root 重启后需重启 Shizuku，服务重启后重开影院。

## 构建和校验命令

在 PowerShell 中运行。先查工具路径及目标设备，构建树保持ASCII。本机工具：Python在 `E:\DSH工作区\yvr-port\tools\py\python.exe`，JDK17、NDKr26d、apktool/uber signer在 `E:\yvrwork`。Python 可执行文件路径含中文不会把构建树变成中文路径。

```powershell
Set-Location -LiteralPath 'E:\yvrwork\work\codex\container'
& 'E:\DSH工作区\yvr-port\tools\py\python.exe' -X utf8 '.\build_cinema.py'
if ($LASTEXITCODE -ne 0) { throw '容器构建失败' }
& 'E:\yvrwork\run.ps1' -Name build211 -BuildOnly
if ($LASTEXITCODE -ne 0) { throw 'APK构建失败' }
& 'E:\yvrwork\jdk\jdk-17.0.21+9\bin\java.exe' -jar 'E:\yvrwork\tools\uber-apk-signer.jar' -a 'E:\yvrwork\signed\build211-aligned-debugSigned.apk' --onlyVerify --verbose
if ($LASTEXITCODE -ne 0) { throw 'APK校验失败' }
```

`build211` 只是新候选命名示例，先检查避免覆盖已有产物。C 两个库均 `-Wl,--no-undefined`；诊断构建才加 `-Diagnostics`，最终交付不加。`E:\yvrwork\dis.py` 会遮蔽 Python 标准库 `dis`，运行分析脚本应在 `work\codex` 子目录。

本机 `run.ps1` 仍含旧自动安装分支 `guardian_pause=1`，只用 `-BuildOnly`。正常设备 guardian_pause=0、锁屏enabled、stayon=false；不要为了测试恢复过时 skip flags。构建不会要求改设备设置。

这些命令示范本机已存在工具和解包基础树的构建，不是直接处理任何原始APK的完整命令。新包前置流程、清单与外部SDK来源读 [bootstrap-dependencies.md](bootstrap-dependencies.md)。

```powershell
& 'D:\adb\adb.exe' devices -l
# 确认目标序列号后；该案例原现场为340YC10G8S0980
& 'D:\adb\adb.exe' -s '340YC10G8S0980' install -r -g 'E:\yvrwork\signed\build211-aligned-debugSigned.apk'
& 'D:\adb\adb.exe' -s '340YC10G8S0980' shell am start -a android.intent.action.MAIN -c com.oculus.intent.category.VR -n com.yvr.launcher2d/com.yvr.launcher2dsdk.UnityMainActivity
```

不要只凭启动输出宣称已运行。睡眠、未唤醒手柄或边界提示可能阻挡进入XR；取得PID/session/frame证据及头显操作验收。

## build210 的原生尺寸补丁

这是取消本影院应用合成层0.8尺寸缩放的版本专用补丁，不用于修复黑屏。

输入原库 `container/libil2cpp.before-build210.so` SHA256：

```text
85711d382d83937581e5150c22426e05749b4334715c822a32f7bab818624b11
```

当前 `decoded/lib/arm64-v8a/libil2cpp.so` SHA256：

```text
8b41f31952d2edc1af602146dbfd4f03325fe16e38c3b041efa7e939a6a0162c
```

| getter | 指令RVA | 文件偏移 | 原字节 → 新字节 |
| --- | --- | --- | --- |
| YVRAppCompositeLayer width | 0x3F5E06C | 0x3F5D06C | 0008211e → 1f2003d5 |
| YVRAppCompositeLayer height | 0x3F5E094 | 0x3F5D094 | 0008211e → 1f2003d5 |

两处 `fmul s0,s0,s1` 改 NOP，共8字节。原 `patch_app_texture.py` 校验 getter 指令结构、ELF地址映射及允许原/已补丁指令，再备份并记录 `app-texture-patch.json`，由 `build_cinema.py` 调用。该脚本并非以完整原库哈希防护所有版本，移植另一 APK 前必须重新定位，不能直接复用。

Skill 中独立的 `scripts/patch_exact_bytes.py` 可用以下 plan 演示原库到新输出的严格校验；不会自动修改 decoded，也不能对已经补丁的输入重复应用。

```json
{
  "original_sha256": "85711d382d83937581e5150c22426e05749b4334715c822a32f7bab818624b11",
  "patches": [
    {"offset": "0x3F5D06C", "original": "0008211e", "replacement": "1f2003d5"},
    {"offset": "0x3F5D094", "original": "0008211e", "replacement": "1f2003d5"}
  ]
}
```

## 证据索引

都在 `E:\yvrwork\work\codex\container`，按症状读取：

- 颜色：`color-analysis.txt`、`copy-color-analysis.txt`、`build207-color-before.txt`、`build208-color-after.txt`。
- 眼尺寸与性能：`build209-resolution.txt`、`build209-performance.txt`。
- 2K/120Hz：`build210-2k-120hz.txt`、`build210-display.txt`、`build210-performance.txt`。
- SDK签名/安装：`sdk-package-verification.txt`。
- 黑屏、媒体、菜单和震动的完整来龙去脉：最新交接的历史记录；原 DeepSeek 文中“0xe/0xf 是FOV”已证伪。

早期诊断位于 `E:\yvrwork\work\codex`：`patch_descriptors.py`、`loader_shim.build190.c`/`loader_shim.build192.c`、`libyvrplugin.build190.so`及build191/192记录。它们用于还原实验，不应直接重跑修改已验收解包树。审核纠错依据还包括当前OpenXR头文件、原库0x6580C的reference-return反汇编、metadata header29、原/最终yvrplugin哈希一致。

这些日志是对应版本的历史证据，不等于新构建已通过。源码ZIP保留新增兼容源码、smali、构建脚本和API许可；不包含全部Unity资源和原闭源库，重建仍需保留 decoded 基础树。Shizuku-API为MIT、Shizuku为Apache-2.0，交付继续附相应许可。

2026-11-08工作区清理已删除根目录历史 `buildN.apk`、除build210以外的签名构建及idsig、probe测试产物、未签名 `sdkshare_patched.apk`。保留上表的签名最终包、提取参考包、工具链、源码、日志及outputs历史交付；清单在 `work/codex/cleanup-2026-11-08.json`。旧打包脚本如 `prepare_outputs.py` 仍指向build204，属于历史脚本，不能直接重跑覆盖最终使用说明。
