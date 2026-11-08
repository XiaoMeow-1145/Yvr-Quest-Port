# DeepSeek 早期记录的审核与纠错

来源：用户提供的 `E:\DSH工作区\yvr-port\DEEPSEEK-SESSION-HISTORY.md`，2026-11-08审核。原文保留作为历史证据；本表结合当前源码/原库、OpenXR头文件、后续交接与用户反馈判断，不直接复制原文第12节作为验证。

原记录SHA256：

```text
f76fcbdf601f2004be3c703753a736bc830d3a23954f3fbe2b7c80f7915f8469
```

## 保留的有效工作

原APK/Unity/ABI结构、YVR loader依赖systemcore、该现场DisabledRuntimeBroker、APEX forwardloader与JavaVM/context初始化、GIPA缓存调用路径、外部SDK动态类加载、普通安装与包可见性、ASCII构建树、未定义符号导致依赖加载失败，以及经过自检的部分读回方法，有源码/运行链依据。将这些保留为有适用范围的能力和查证步骤，详见 [bootstrap-dependencies.md](bootstrap-dependencies.md)、[openxr-input.md](openxr-input.md)、[unity-diagnostics.md](unity-diagnostics.md)。

## 纠正的结论与过度推断

| 原记录的位置/说法 | 判断与修正 | 依据/去向 |
| --- | --- | --- |
| §0/§4：完整跑进影厅并稳定72fps | 已有场景对象与XR帧循环，不能称影厅已经显示或完整功能正常 | 当时黑屏；后续以真实图像与用户反馈验收 |
| §4：两profile动作路径完全一致 | 抓握click/value就是反例，profile映射后还需逐action核对类型/路径 | BOOLEAN→FLOAT辅助action；[openxr-input.md](openxr-input.md) |
| §4：-6是API_VERSION_UNSUPPORTED | **-6是INITIALIZATION_FAILED，-4才是API_VERSION_UNSUPPORTED** | 当前 `inc/openxr/openxr.h` enum直接核对 |
| §8：-31是FUNCTION_UNSUPPORTED | **-31是REFERENCE_SPACE_UNSUPPORTED，-7才是FUNCTION_UNSUPPORTED** | 同上；原数值/命名错误不能沿用 |
| §4：runtime自己是1.1.61 | 所引日志API值 `0x100000000003d` 解码为 **1.0.61**；不要混淆version编码和文中标签 | XrVersion高16位major、次16位minor、低32位patch |
| §4：maxApiVersion必须固定XR_CURRENT_API_VERSION=1.1.65 | 该范围曾成功，不是所有设备唯一合法值；区分header版本、支持范围、negotiated版本与实际错误 | 验证初始化/协商及合理版本范围 |
| §5：data descriptor导致中央目录不可用 | descriptor是正常ZIP结构；失败也可能是错误base/不完整提取/目录定位等，原因需查证 | [bootstrap-dependencies.md](bootstrap-dependencies.md) |
| §5：取下一个PK header、跳过失败项，再testzip就证明完整 | 压缩数据也可含PK签名；新ZIP的CRC只能证实新包自洽，不能证明原条目完整 | 按EOS/descriptor/原CRC/文件集合验证；历史雕刻脚本是启发式 |
| §6：RGBA全0证明图像从未写入 | 全0只是正确采样时点的值，透明清屏、覆盖、错index/layer/时点均可能 | 自检、合法持有时点与调用序列；[unity-diagnostics.md](unity-diagnostics.md) |
| §6：renderer.isVisible和shader支持证明几何应被画 | 是初步状态，不能替代当前XR pass的眼数据、裁剪与实际draw | 对照原插件描述符 |
| §6/§8：Cube不出现、draw计数0排除其他因素 | 某些诊断未满足线程、方法、FBO或真实hook路径条件，排除力度不足 | 只保存观察，不把不可靠诊断当定论 |
| §7：NaN必然使全部对象被剔除 | 具体结果取决于比较/culling实现；也需检查诊断布局与上游语义 | 有限值/无警告不能证明正确 |
| §7：0xe/0xf是FOV数组 | **是XrPosef：quaternion xyzw+position xyz；0xd是米制IPD** | 原插件输出和build191眼位、VIEW修复/用户验收 |
| §3/§10：每次重启后必须禁用锁屏/边界、skip flags、常亮 | 旧测试现场的绕过措施，不是正常移植步骤 | 当前正常设置恢复；佩戴/唤醒/处理系统提示，使用BuildOnly |
| §3：account provider异常就重试启动 | 必须修复非关键账号/遥测边界，保持初始化后续；偶然成功不是修复 | 后续Android兼容与菜单验收 |
| §2：metadata“改成29.1” | 文件头仍是整数29；29.1是工具布局选择的历史表述，不能改原metadata header | 实际header检查；当前Dumper config已是别的配置 |
| §2：UTF8 BOM一定使Python源文件报错 | CPython文件读取识别UTF8 BOM；字符串exec/错误解码/拼接才可能引入U+FEFF错误 | 仍建议明确无BOMUTF8，区分运行方式 |
| §9：method=NULL一律被裁剪、所有XR回调一定在渲染线程 | 也可能查错重载/namespace；线程由调用点决定，必须现场确认 | 对照MethodInfo/signature和实际线程 |
| §11：改getResolutionScale返回1即可 | 原函数返回引用地址，不能换成float寄存器返回 | 原库0x6580C反汇编 `x0=this+0xbc; ret` |
| §8：眼尺寸精确等于面板11/12 | 接近并含取整；1680/1832非精确11/12；不能据这个关系证明全部渲染尺寸正确 | 用runtime/Unity实际分配，不靠猜比例 |
| §1：不要求二维应用，TCXRMode2d可放弃 | 后来用户明确要求并验收巨幕内二维App | 纳入Shizuku、列表/图标/输入/焦点完整流程 |
| §12：每眼2K就概括完成画质 | 明确每眼2384×2496、应用纹理2560×1440、菜单960×524；120Hz不等于所有内容120FPS | [color-quality.md](color-quality.md) |

## 后续对话补齐的失败与成功路径

这些不是DeepSeek原文的全部内容，但属于用户明确要求整合的移植经验：

- 抓握修好而菜单仍无响应：按键active/down正常后继续查Android初始化，不重猜菜单binding；修复后用户确认菜单可打开。
- 手柄定位正常但目测需前翻45°：测量grip/aim得到约-60°差，最终用真实aim方向，不按目测常数补旋转。
- 列表出现却无视频/不能播：区分媒体查询与READ_EXTERNAL_STORAGE；修复后用户确认真实片源画面/声音/暂停继续。
- 封面与震动：缩略图缓存/回调和smali构造器VerifyError分别解决；短振方案基于原固定1.5s请求，不能仅增加stop而忽略应用未发送stop。
- SDK整合：用户确认卸载后异常、装回恢复，最终要求独立保留；误交付未签名SDK导致校验失败，改签名源后安装成功。
- 二维App白图标“Application Name”：修复多个图标入口/包查询，创建完整Shizuku副屏容器，不能用返回[]假装功能可用。
- 二维发白、三维正常：按实际编码处理二维sRGB存储/RGBA view，保持立体层；用户验收颜色正常。
- 用户指定Q3S面板1.3倍、2K应用巨幕和120Hz：分别修改眼推荐尺寸、虚拟屏/合成层尺寸及刷新率，并验证真实分配/输入/播放，不能只改配置数值。

## 仍有限制的当前实现

审核 `shim/loader_shim.c` 发现 `yvr_stub_common` 注释声称未知getter返回unsupported，但实际通用分支用 `XR_SUCCESS`，部分按地址外观写0，部分数组/其他输出不写。这与注释不符，也不是可靠ABI验证。已经独立实现的眼位、主手柄/输入设备与typed刷新率接口要与它区分。

这些通用桩在影院已验收路径上未阻止用户所测功能，不能证明boundary/camera/passthrough/foveation/hand mesh等接口均有真实实现。Skill不能把当前shim描述为完备的通用YVR runtime。另一APK若调用这些接口，先确认签名/语义、真实平台替代和调用方fallback，逐项实现；不要复制“全部成功”或批量改成unsupported后假定仍兼容。

同样，现有单例session/eye/pose跟踪、固定容量表及资源清理的泛化能力未全面验证；长期断线运行、无线启动、更多第三方App和新OS版本仍分别需要测试。这些是实现范围说明，不能把历史脚本直接当作所有场景的模板。
