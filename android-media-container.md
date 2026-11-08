# Android 兼容、媒体与二维应用容器

## Android 依赖与 SDK 支持包

清单和 Java/smali 中检查 YVR 特有 provider/service、系统设置、账号、遥测、设备型号与 SDK 进程。Android Exception 可能中断 Unity 初始化后续，即使 native XR 正常也会出现“菜单按键没功能”。

为非关键接口做明确兼容降级：保持返回类型、回调和调用者预期，捕获该边界实际异常，不把全部异常吞掉。关键功能用 Quest 上存在的等价实现替换；返回空列表或 null 只是暂时降级，需要记录功能缺失。

本影院 `com.yvr.sdkshare` 仍需独立安装，用户实测卸载使部分功能异常、重装恢复。已经选择保留独立包，不自动合并。其他 YVR 应用是否需要它，由调用和实测决定。

提取系统 APK 可能有 `sharedUserId=android.uid.system` 或签名权限，在新设备无原平台签名时不能原样安装。案例支持包移除这个 sharedUserId、重建并签名后有效。逐项检查依赖所需权限；移除 sharedUserId 不会凭空获得系统权限。需要证书一致的相互签名访问，使用项目确定的签名方案。

案例 `sdkshare_new.apk` 是镜像 local-header 重建的未签名参考包，曾被误交付而安装校验失败。正确交付来自 `sdkshare_signed/sdkshare_patched-aligned-debugSigned.apk`，签名/对齐验证及实机安装均通过。ZIP 能打开、文件复制成功、哈希与源文件相同都不能证明有有效 APK 签名。

设备型号若控制资源选择，返回原应用确实支持的型号是该案例的兼容措施；不要把全部 Quest 设备系统身份改成 YVR。

## 媒体列表与播放权限

先区分“没有记录”“记录有但打不开”“能解码但没首帧”。记录 Android 版本、target SDK、实际授予权限、MediaStore 查询和解码日志。媒体缓存/列表不是权限验证。

本影院 target SDK32，在 Android14 上仍需声明和授予 `READ_EXTERNAL_STORAGE`；限制 `maxSdkVersion=32` 会使运行时拿不到它，导致 IJK `Permission denied`。这个结论不是给所有 target SDK 的应用添加旧权限：新 target 的图像/视频/音频、选择照片访问等应按实际 Android 权限模型处理。

MediaStore 分页用 query Bundle 的 sort/limit/offset，避免依赖厂商 SQL 后缀。保留原请求语义：负 limit 不限制数量、空排序列使用 `_id` 等有证据的默认，Cursor 关闭与异常回调完整。媒体监听若原 YVR 管理器不存在，可先降级通知，保留重新查询路径；记录自动刷新是否实现。

播放验收同时检查首帧、声音和暂停继续。一次自制 H264 小视频通过后，应使用用户真实片源验证；最终影院用户实测 HEVC 1920×816 正常。不要把首帧日志当作整个文件、全部格式或 DRM 播放能力的证明。

## 封面生成与 smali 验证

复用应用原异步 worker/JSON 回调。Android `ThumbnailUtils.createVideoThumbnail(File, Size, CancellationSignal)` 可在支持的设备上生成有界封面，本案例默认480×270、最大边请求限制2048。

缓存键包含 canonical path、文件大小、mtime、请求尺寸，放应用自身 external-files/thumbnails。先写临时 JPEG，流关闭且内容完整后再发布最终文件；失败需返回应用可处理的结果并清理临时文件。读取实际 JSON 路径、文件存在性和 UI 展示，避免只检查函数进入日志。

smali 构造对象时，`new-instance` 的未初始化引用不能在 catch handler 中与已初始化引用合并。本案例 FileOutputStream 导致 VerifyError；用临时寄存器构造完成后再赋给用于 cleanup 的变量。修改 `.locals`、catch 范围和所有分支后，验证设备 ART 加载与实际代码路径。

## 二维应用：列表只是容器的入口

先澄清产品目标：从影院跳到系统应用，还是保持影院运行并在巨幕内显示、操作该应用。应用列表和启动 Intent 不会自动实现巨幕嵌入。

本案例原 `tcxrservice/YvrXR` 负责 displayPanel、SurfaceTexture、副屏启动和输入；Quest 没有该服务。恢复旧类或只补包查询不能提供服务。

已验收替代结构：

```text
影院 Unity 巨幕 / 菜单
        ↕ 保持原 Unity 回调与尺寸协议
ContainerController → QuestBridge → Shizuku UserService (shell UID2000)
        ↕ Surface / 目标 displayId
可信 VirtualDisplay ← 目标应用窗口
        ↕ SurfaceTexture / OES
共享 EGL context → Unity 巨幕纹理
```

Shizuku 需要安装、启动、影院授权及 UserService 绑定。先用小范围测试确认该 OS 的 shell 创建副屏、启动和注入能力，再整合进正式 UI。Shizuku 不保证每个 Quest 系统版本都可访问同样隐藏 API。

### 显示与焦点

- binder 中先验证调用方，再 `clearCallingIdentity`，系统服务操作完成后在 finally 恢复，避免原影院 UID 覆盖 shell 身份。服务需限制可调用者、显示尺寸和可操作应用，不能把任意系统指令开放给任何包。
- 虚拟副屏保持自己的焦点，避免夺走影院主屏焦点；案例包含 `OWN_FOCUS`、`STEAL_TOP_FOCUS_DISABLED`，其完整 flags 为 `1|8|64|256|1024|2048|4096|16384|65536`。其他版本查实际定义与权限，不直接复制魔数。
- 启动明确 component 到已创建 displayId；用 dumpsys activity/display 验证目标窗口和影院焦点。accepted/Success 不足以证明位于副屏。
- 处理绑定失败、binder death、Surface/VirtualDisplay/context 清理及重连。独立共享 EGL context 释放不能 `eglTerminate` Unity 仍使用的显示。

### 输入与坐标

触摸 MotionEvent 设置 `TOOL_TYPE_FINGER`、`SOURCE_TOUCHSCREEN`、`deviceId=-1` 和正确 displayId；按下/移动/释放保持 pointerId 与 downTime。案例 UNKNOWN toolType 即使 inject 返回 accepted=true 也不触发应用点击。

滚动鼠标、键盘、返回键分别使用相应 source/type，不把所有事件都当手指。射线坐标经过 Unity 菜单比例、巨幕纹理和虚拟屏幕尺寸映射；创建/resize 回调必须更新 Unity 的实际窗口宽高。测试边角点击、连续滑动和返回，不能只测中央一个按钮。

### 列表和图标

查询设备可见、可启动的目标 component，明确 package visibility 清单要求，保留原 Unity JSON 字段/回调。白图标的“Application Name”可能是 Android query/provider 异常导致旧默认模板被显示。

影院三个旧图标入口均改到 `QuestApps.icon`，生成256×256图标，名称/列表用 PackageManager；只改其中一个入口仍会调用不存在的 YVR provider。不要用硬编码应用清单掩盖查询错误。

## 服务生命周期与验收范围

非 root 头显重启后需要重新启动 Shizuku；服务重启后关闭重开影院或实现已验证的重连。本影院的普通影院/本地视频不依赖 Shizuku，巨幕二维应用依赖。

测试授权拒绝/服务不在线、应用启动和输入、退出应用后影院恢复。断开 ADB 长期运行、无线启动、新 OS 版本和更多目标应用要分别记录；当前影院交付未全面验证这些场景。禁止将这份案例扩大为“任意 App 都可嵌入”的承诺。
