# 图层颜色、分辨率与刷新率

## 二维发白，三维正常

逐层检查 projection/quad、swapchain format、arraySize、渲染器写入和 compositor 读取的颜色空间。正常三维场景是缩小故障范围的证据，不应先修改全局 gamma、材质或视频亮度。

本影院的菜单/应用二维拷贝 shader 写入已显示编码的 RGB，`GL_RGBA8` 却被 Meta 按线性 RGB 读出后再次显示编码，因此发白。三维双层 array 的颜色本来正常。先确认实际 shader/拷贝数据和格式，不能仅因 arraySize1 就断言所有二维层都如此。

已验收的零逐帧复制方案：

1. 仅对证实采用这种编码的二维 RGBA8 层，以 `GL_SRGB8_ALPHA8` 建立 runtime swapchain 存储。
2. 通过 `GL_OES_texture_view` 返回同一存储的 `GL_RGBA8` view 给原渲染器，保持它写入编码 RGB 的方式。
3. Meta 采样 sRGB 存储时正确解码；原正常立体 projection arraySize2 保持 RGBA8。

技术前提：runtime 支持该 swapchain 格式、texture immutable、格式兼容、存在 GL_OES_texture_view 和函数入口、当前 EGL context 有效。本案例 candidate 还要求 faceCount1、sampleCount1、color-attachment。不能对多采样、非兼容格式或线性写入图层套用；前提失败时保留原路径并记录问题，避免半创建状态。

拦截 CreateSwapchain、EnumerateSwapchainImages 和 DestroySwapchain；处理只查询 count 与重复枚举，为每个 runtime image 创建/复用对应 view，保持 mip/层数、GL binding、alpha与清理。原 image 由 runtime 管理，兼容层只清理自己创建的 view。在有合适 context 时删除 GL 资源。

最终确认菜单文字/图标、巨幕二维应用、视频与三维场景同时正常。不以全局近似 gamma shader 掩盖单层色彩解释错误。

## 分开记录四种尺寸

| 对象 | build210 的实际值 | 调整路径 |
| --- | --- | --- |
| Quest 3S 物理面板单眼 | 1832×1920 | 硬件规格，用作用户要求的基准 |
| Unity 双眼渲染 | 每眼2384×2496，arraySize2 | xrEnumerateViewConfigurationViews 的建议尺寸及实际分配 |
| 二维应用虚拟屏幕 | 2560×1440，density320 | VirtualDisplay 配置 |
| 二维应用巨幕纹理 | 2560×1440 | 应用合成层真实尺寸/缩放和纹理分配 |
| 主菜单独立纹理 | 960×524 | 本轮没有调整 |

物理面板、推荐渲染尺寸、提交纹理是不同概念。眼分辨率提高不能让1080p应用源纹理产生更多细节；视频片源的清晰度也有自己的上限。

### 每眼分辨率

本案例 Meta 原建议1680×1760。用户要求“Quest 3S面板的1.3倍”，不是原推荐的1.3倍。1832×1.3=2381.6，向8像素对齐为2384；1920×1.3=2496。像素量约原推荐2.01倍。

在 `xrEnumerateViewConfigurationViews` 修改建议值并限制在 runtime max 内，随后检查 Unity 真实 swapchain 的 width/height/arraySize、渲染 viewport 和提交 imageRect。只扩大已分配 swapchain 不能证明 Unity 多渲染了像素。平台最大尺寸是合法上限，不是可持续性能预算。

其他头显先查其规格与 runtime 推荐。尊重用户选择的基准及倍率，避免把2384×2496写成所有 Quest 的默认。

### 应用巨幕与二进制补丁

原副屏1921×1081，巨幕纹理1536×864，是 YVRAppCompositeLayer 尺寸 getter 的0.8缩放。提高副屏后仍会被 getter 缩小。

build210 同步将副屏与应用纹理设2560×1440，并将 density240调320，以基本保持1080p时期的 UI 布局。发送创建后的尺寸回调同步 Unity windowWidth/windowHeight 和射线坐标。不能只改显示配置就宣称巨幕是2K。

取消缩放使用该版本 libil2cpp 两处 fmul 改 NOP；详细原哈希、RVA和偏移见 [cinema-build210.md](cinema-build210.md)。新 APK 必须重新定位 getter、检查常量来源和 ELF 映射。原库 hash 或指令不匹配，不按旧地址试写。

二进制写入工具只保证指定原字节被替换，不能验证 getter 属于目标功能。优先可维护的源码/接口修复；只有确实需要且能确认语义时使用补丁。

## 120Hz 与真正帧率

先枚举 runtime 支持刷新率，启用 `XR_FB_display_refresh_rate` 后用 typed float API 请求所需值，并读取当前值。支持率不存在时报告并选择符合用户意图的可用值，不把请求返回成功替换成虚假的120。

原 YVR 可能在 session begin 后再次请求72/90Hz。影院最终将三个 YVR enumerate/get/request 接口以正确 float ABI 转发并根据用户选择覆盖120，在 BeginSession 也请求。其他任务不要默认覆盖用户的节能或性能设置。

VirtualDisplay 自己也可有 requestedRefreshRate；build210 用 VirtualDisplayConfig 请求120，dumpsys 确认2560×1440/density320/refreshRate120。仍需查看 XR 实际帧率与耗时：120Hz 每帧预算约8.33ms。

区分头显刷新率、Unity 出帧、应用副屏更新和视频片源帧率。24/30/60fps 视频不会因为面板120Hz变为120fps视频。记录 steady-state、启动/切换掉帧、stale frames及温控。影院短时日志主要120/120或121/120，启动切换短暂掉帧，用户验收清晰流畅；这个结果不能推广成长期零掉帧或任意负载120FPS。
