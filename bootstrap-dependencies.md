# 原包、系统镜像依赖与 Quest 启动

本节补足从原始 YVR APK 到“可安装、可进入真实 XR 会话”的前半程。静态检查先于重建；保持已验收的功能范围，6DoF 不因为原清单写3dof就被降级。

## 原包与平台基线

记录原包哈希与来源，原版和“已去系统 UID”版分别保存。检查：

- Application、实际 Unity Activity、MAIN/LAUNCHER/VR filter、provider/service、进程、exported、sharedUserId、权限和包可见性。
- ABI、min/target SDK、Unity版本、`libunity.so`/`libil2cpp.so`/YVR plugin/OpenXR loader、播放器与容器库、资源和 metadata。
- ELF 导入/导出和 DT_NEEDED、字符串中的系统包/Java反射类、跨 APK 的动态类加载及 JNI 调用。

影院原包是 Unity2022.3.5f1/IL2CPP/arm64 的3D影厅，不是可以套二维窗口的普通Activity。原 `com.yvr.application.vr_flag=3dof` 不能证明内容只能3DoF；检查实际 pose 消费并保留6DoF。

本机 original `libyvrplugin.so` 与最终 decoded 中恢复的原插件 SHA256 都是：

```text
fab2c82b475e7a97849b923fa5b45d7b40711d0d96f3c720018048182b62a989
```

这支持“黑屏修复主要在 shim 纠正数据、最终恢复原插件”的案例结论。新包不能凭同名库复用地址。

## 工具链和构建树

本机已有 apktool2.10.0、uber-apk-signer1.3.0、JDK17、NDKr26d、Python3.12；先检查已存在工具，不重下整个NDK。arm64的编译/API level按原程序和设备能力选择；链接 `-Wl,--no-undefined`，ELF soname/导出/依赖及设备实际加载一起验证。

`DllNotFoundException: yvrplugin` 可能来自它的 `libopenxr_loader.so` 依赖缺符号，并非 yvrplugin 文件不存在。查看完整 dlerror 和依赖链，修复真正未定义符号；不要反复重命名库。

使用独立、纯ASCII工作树。保留原始资料、解包树和每次补丁依据，但避免无限保存数百个大APK。已有证据足够时再构建一个针对性候选；除用户要求清理外，不主动删除原/参考材料。

Windows细节：

- Windows PowerShell5.1 的文本重定向可能破坏 native stdout 的二进制输出。截图等用设备端写文件再 `adb pull`，或使用确实保持 bytes 的 subprocess/流；不能推断所有 PowerShell 版本都一样。
- PowerShell5.1 `Set-Content -Encoding UTF8` 默认带BOM，可用明确无BOM的UTF8写入。CPython直接执行UTF8 BOM源文件通常可识别；报 `U+FEFF` 时检查是通过字符串 exec/错误解码/拼接而引入，别把“Python不支持BOM”当作定律。
- 当前嵌入式Python有 `python312._pth`，可能忽略PYTHONPATH；外部校验依赖可用受控 `sys.path.insert` 加入，再运行工具。`dis.py` 遮蔽标准库问题仍需在子目录运行。
- 用日志缓冲区快照或可靠输出流；停止 Start-Job 可能丢未flush数据。日志带build/PID/时间，并保留已有崩溃内容；确需清空logcat时先保存证据。

## Quest 清单入口与普通安装

影院移除 `sharedUserId=android.uid.system` 后以项目签名构建普通安装包。同步检查外部SDK签名访问；移除属性不等于获得系统UID/特权。

为实际 Unity 启动Activity的 intent-filter 加 `com.oculus.intent.category.VR`，保留 MAIN/LAUNCHER、正确 exported 和 Unity lifecycle。案例还有 application metadata `com.oculus.vr.focusaware=true` 与 `com.oculus.supportedDevices=quest|quest2|questpro|quest3`；是本机有效清单示例，设备名称与平台要求应查目标版本，不随意添加未识别字符串。

VR category 告诉系统如何启动，不能补齐 runtime、pose 或系统服务。用实际 Unity Activity 启动并验证 immersive transition、进程及 XR session，而不是看到VR入口就认为画面成立。

## SDKAPP：不只是一个可有可无的服务

影院代码通过 `createPackageContext("com.yvr.sdkshare", 3).getClassLoader()` 和 `DexClassLoader(pm.getApplicationInfo(...).sourceDir, ...)` 加载外部代码。`UnityMainActivityShare` 接收生命周期/触摸；`CinemaPlayManager`、`SystemSettings`、`homesdk.AppManager` 和 `YVRCinemaPlayProvider` 都在SDK包中。完整包还可能有多DEX、资源和JNI，不能只抄几个类或把compile stub当实际实现。

检查动态依赖时区分：

| 现象 | 首先检查 |
| --- | --- |
| pm里没有支持包 | 文件来源、签名/安装结果、正确用户 |
| pm有包但应用NameNotFound | `<queries>`、目标SDK、包名/用户与实际查询 |
| 找到包但ClassNotFound | 全部DEX、准确namespace、ClassLoader及父加载器 |
| 类能加载但功能异常 | 资源context、初始化、回调、方法签名、所需系统服务 |

已安装的SDK仍可能对调用方不可见。影院 `<queries>` 明确包含 `com.yvr.sdkshare`；二维App查询再按需求声明MAIN/LAUNCHER查询。包可见性只让应用查询已存在包，不能创造缺失服务或授权特权权限。

最终SDK独立保留，签名源和验收见 [cinema-build210.md](cinema-build210.md)。未签名 `sdkshare_new.apk` 保留为提取参考，不能再交付。

## 从系统镜像提取缺失 APK

先检查镜像是否有效，零字节分片不是可用分区。识别Android sparse、dynamic super分区与ext4/EROFS等实际布局，优先恢复/导出文件系统或用可用原设备提取完整APK。`super_2.img` 是本例存在的主要素材，不表示所有super文件都是可直接扫描的文件系统。

文件系统方法不可用时，ZIP雕刻是恢复候选的最后办法。本例搜 `com/yvr/sdkshare` 等DEX字符串定位候选，以镜像偏移 `0x5170D000` 附近的local-header序列恢复653项，输出28,087,705字节的 `sdkshare_new.apk`。偏移只适用于原镜像；本机 `rebuild.py` 实际从已有 `sdkshare.apk` blob中 BASE0x25000 重建，两个偏移属于不同文件坐标。

**纠正历史恢复算法**：data descriptor 是合法 ZIP 功能，不会天然使中央目录不可用。不能把“下一个PK local-header”直接当压缩数据终点，因为压缩/存储数据内部也能出现同样字节；历史脚本会跳过解压失败项和重复名，属于启发式恢复，不是可靠通用ZIP parser。

雕刻时验证 local header 的flag/method/name/extra/大小和边界：

- 无descriptor时按已校验compressed size读，支持的method才解码。
- deflate带descriptor时，用raw deflate decoder的end-of-stream定位，再读有/无签名及ZIP64的descriptor，核对CRC32、compressed/uncompressed size。
- stored且未知长度，不能靠deflateEOF定位；需可信目录或一致的descriptor/候选边界验证，证据不够标为未恢复。
- CRC32与原记录匹配，且候选包含有效 AndroidManifest/resources/all DEX/必要native/assets，包名版本和关键类语义吻合。对失效项逐条记录，不能静默跳过后宣称完整。

重建ZIP会失去原APK签名。`zipfile.testzip()`只检查新ZIP记录的CRC：如果写入时重新计算了错误/残缺数据的CRC，也能通过。还需检查原CRC、文件集合、DEX/资源解析和功能；随后apktool修改、对齐、重新签名、安装验证。

## 启动阻挡与真正兼容错误

头显未佩戴、手柄睡眠、SensorLock/Guardian系统对话框、失焦或暂停可阻挡XR。先让用户正常佩戴/唤醒并处理系统提示，查看前台窗口和session状态。重启不是每次启动的必要条件。

原记录批量禁用锁屏、设置guardian_pause/skip flags及常亮，仅属旧现场测试变更；不作为安装或重启后默认步骤。不要force-stop guardian/vrshell等系统包，历史现场因此出现重连故障。现有正常设置保持；确有系统异常时根据日志处理，而不是反复改测试开关。

若崩溃来自 `YvrDataLog.getAccountInfo` 的缺失provider/NPE，修复非关键账号/遥测边界；重试直到偶然成功不能解决初始化不确定性。MEDIA/服务初始化继续保留必需回调，详见 [android-media-container.md](android-media-container.md)。
