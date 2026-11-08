# OpenXR、眼位与手柄输入

## Runtime 与 shim 的边界

原 YVR loader 可能依赖 `org.khronos.openxr.runtime_broker` 或硬编码 `com.yvr.systemcore`。本案例 Quest 的 broker 实现是 DisabledRuntimeBroker，查询得到 null；增加 `<queries>` 不能让它提供不存在的 runtime。

确认设备当前 OpenXR 实现、可加载路径及 linker namespace，再选择标准 loader 或最小 shim。案例 shim 从本机 Meta runtime 加载、协商 `xrNegotiateLoaderRuntimeInterface`，以 runtime 的 GIPA 转发标准接口。Android loader 初始化需要有效 JavaVM 和 Application context，使用 `xrInitializeLoaderKHR`；不能把“库 dlopen 成功”当作 XR 初始化完成。

案例曾验证的路径：

```text
/apex/com.meta.xr/priv-app/VrDriver/VrDriver.apk!/lib/arm64-v8a/libopenxr_forwardloader.so
/apex/com.meta.xr/priv-app/VrDriver/VrDriver.apk!/lib/arm64-v8a/libvrapiimpl.so
```

这是该 Horizon OS 现场的内部路径，版本更新后应重新查证，不硬编码为通用 Quest 合约。加载自己的 loader 时避免再次加载同名 shim 造成递归。

该现场还可从 `/apex/com.meta.xr/active_runtime.aarch64.json` 确认forwardloader目标，其后经RIPC/VrRuntimeService加载真正实现。APK里看到库文件不表示应用linker namespace一定能加载；记录dlopen/初始化/协商各层的真实错误。

初始化成功后协商支持的接口/API范围，验证输出size/version/GIPA；先有有效JavaVM/context再走该现场forwardloader初始化和协商。原插件没有检查xrCreateInstance失败，随后通过NULL函数指针调用GraphicsRequirements导致SIGSEGV；应在上游失败边界终止或提供正确实现，不只修最后崩溃指令。

错误码以实际头文件为准：`-4=XR_ERROR_API_VERSION_UNSUPPORTED`、`-6=XR_ERROR_INITIALIZATION_FAILED`、`-7=XR_ERROR_FUNCTION_UNSUPPORTED`、`-31=XR_ERROR_REFERENCE_SPACE_UNSUPPORTED`。原记录将-6/-31命名错了。所引runtime version `0x100000000003d`为1.0.61，不是1.1.61；header的XR_CURRENT_API_VERSION不等于实际runtime版本。

处理 `XR_YVR_*` 时先列出实际调用。只过滤扩展名不能补足函数语义；只有必要接口确实实现时才能向应用宣称支持。未知接口明确报告 unsupported，或先查明调用者所需行为。

私有 ABI 从头文件或 ARM64 调用点确认：整数、指针和 float 参数不走同一寄存器类；用通用整数参数 stub 接收 float 刷新率会得到错误值。特别注意 getter 的输出类型、位宽、数组容量/count、合法枚举及状态 getter/setter的一致性。不能返回 `XR_SUCCESS` 而不填写输出。

标准 API 的 `type/next`、两次枚举容量、错误传播和资源销毁要保持。拦截既放在导出函数，也放在 GIPA；仅替换一个入口容易让动态调用绕过修复。

frame/session按真实状态推进，检查同步、预测显示时间、acquire/wait/release与EndFrame对应图层。有限尺寸、合法四元数和non-NULL函数指针是必要数据检查，不替代图像验收。需要读回或managed诊断时读 [unity-diagnostics.md](unity-diagnostics.md)。

## 黑屏：先检查空间与输入给渲染器的数据

按实际失败边界检查：

1. 进程、原生库加载、XR instance/session、状态事件与 begin session。
2. wait/begin/end frame 返回、shouldRender、swapchain acquire/wait/release 及提交的 projection/quad 图层。
3. 原插件交给 Unity 的 eye pose、FOV、参考空间、投影/裁剪矩阵及实际纹理内容。

Unity 只有清屏且不提交几何，可以是有效但错误的眼位，不能据此断言 shader、URP、layer mask 或 NaN。先读数据，尤其检查私有属性是不是被错误解释。

**YVR 影院 2.9.12 的证据**：原插件读取的 `0xd` 是米制 IPD；`0xe/0xf` 是两眼 `XrPosef`，排列为 quaternion xyzw + position xyz，共 7 个 float。旧 shim 按 FOV 处理，Unity eye pose 变成 `(0.1,1000,-1)`，相机离场景约千米，形成“只清屏”。用真实眼睛相对头部姿态后恢复显示。其他版本必须确认编号和 ABI。

从 runtime 的 `XR_REFERENCE_SPACE_TYPE_VIEW` 中取得两眼相对头部的 pose；LOCAL/STAGE 世界位姿不能直接写入相对眼位，否则头部移动被应用两次。位置以米为单位，四元数顺序与坐标转换各做一次。FOV 是独立的 `XrFovf` 角度字段，单位弧度，不是 eye pose。

检查 locate 的位置/方向 valid flags、双眼数量与时间域。新实现优先使用帧循环的 `predictedDisplayTime`，或验证过的 OpenXR 时间转换；不要未经检查将任意时钟直接当 XrTime。案例眼位代码采用 CLOCK_MONOTONIC 纳秒并有默认 64mm 眼距，这是现场实现细节；默认值不能作为 tracking 成功的证据。

已解决的影院不要重演大量 descriptor/GL draw/NaN/强制3DoF/shader 改动。新应用若数据正常而帧链仍有反证，再有针对性检查这些层。

## 抓握：模拟量转布尔

原 YVR 只有按/不按；Quest Touch 抓握输入是 `/input/squeeze/value` 的 FLOAT，不能把 BOOLEAN action 直接绑定到该路径。

案例保留应用的原 BOOLEAN action，另外创建同 action set/subaction paths 的 FLOAT action，再将原 `/input/squeeze/click` 绑定转到它。`xrGetActionStateBoolean` 以 `isActive && value > 0.5f` 合成按下。0.5 是用户接受的该案例阈值，有其他需求时调整。

每次 `xrSyncActions` 更新 epoch；每手/subaction 保存当前值、changedSinceLastSync 和变化时间。同一 sync 周期重复查询不得再次改变 changed。检查 inactive、释放与左右手分离；不能只证明“一次按下有日志”。新增 action 必须在 action set attach 前创建。

映射 YVR interaction profile 到设备支持的 Touch profile，并核对全部 binding。右系统键可能由平台保留，不应代替左菜单输入。

## 手柄角度：分开 grip 与 aim

Quest grip pose 是握持框架，aim pose 是射线框架。位置正确而方向需人为前翻才能吻合时，在同一 base space 和同一 XrTime 对比两者。

本影院模型/射线期待 aim 方向但 grip 原点：保留 grip position，使用 runtime 实际 aim orientation。辅助 aim action/space 与原 grip space 成对，检查方向 valid flags，失败时保留原有效 pose；注意 session/space/action 的生命周期。

该现场相对四元数约 `(-0.5,0,0,0.866)`，约 -60°。用户目测45°不是需要写死的校正值。其他应用若分别消费模型和射线 pose，保留各自框架而不是全局替换。

## 菜单：输入正常后追踪初始化

沿 binding → action active/down/change → Unity 输入事件 → system focus/UI init → 菜单回调逐层确认。本案例左菜单输入已经正常，但 YVR-only Android API 抛异常中断初始化。

修复引起初始化中断的账号/遥测、provider、设备型号、媒体监听及应用查询接口；保留必要状态更新和回调。不要永久将 focus/init 写 true，或因菜单不响应反复改键值。

原私有输入设备 getter 也需要写出正确位宽。影院中的 currentInputDevice 写 uint64 枚举值1，primaryController 写 uint32 右手值1并保持 setter 状态；这些枚举仅对该插件版本有证据。

## 震动：区别持续时长与停止信号

记录原请求 amplitude、frequency、duration 和 stop 调用。移开控件仍振动可能是单次固定长脉冲，并非转发慢或停止丢失。

影院 YVRSetControllerVibration 原 API 仅频率/强度，插件硬编码 `1,500,000,000 ns`。悬停只发送正强度，没有移开 stop。最终只将 `duration == 1.5s` 且 `frequency in (0,1]` 的旧菜单请求转为 50ms，将归一化频率改成 `XR_FREQUENCY_UNSPECIFIED`；保留其他明确时长/Hz的请求。amplitude 为0转为真实 `xrStopHapticFeedback`。

不要对所有应用、所有震动请求统一裁剪50ms。测试多控件快速切换、左右手和释放，确认行为自然后关闭高频诊断日志。
