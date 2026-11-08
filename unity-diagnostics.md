# Unity/IL2CPP 和 GL 取证

只在普通日志/静态调用不足以确定故障时加载本节。诊断用于证明具体数据流，不能让测试改变本身成为“原程序为何失败”的证据。最终包关闭这些探针。

## 证据分层

| 观察 | 可以支持 | 不能单独证明 |
| --- | --- | --- |
| 正常wait/end frame、约72fps | XR调用链和节奏运行 | 场景已经呈现或几何绘制正常 |
| IMAX onSceneLoaded、271 renderer | 场景对象存在，记录的状态可查询 | 每个renderer进入当前眼的绘制pass |
| shader isSupported、普通camera矩阵有限 | 初步排除空材质/部分错误 | 所有shader pass兼容或XR眼矩阵正常 |
| 读回RGBA全0 | 被正确采样图像在该时点为0 | 图像从未写入、几何从未执行 |
| 强制绿clear后读回绿色 | 该修改下clear与读回通路有效 | 原路径全部图层/所有帧正常 |
| draw hook计数0 | 该hook没有观察到调用 | 所有GL调用路径都没有draw |

失败诊断本身先做正对照。早期Cube/RenderTexture/Draw计数测试涉及错误线程、缺失方法、不完整FBO或函数表绕过，不属于同等可靠的排除证据。不要在已解决影院重演这些实验；新应用有对应证据缺口才选择其中一项。

## GL 图像读回

VR系统screencap可能没有包含应用的immersive合成层。截图黑并不说明眼纹理黑，可从已确认的swapchain图像做有限诊断。

记录handle、实际width/height/format、arraySize/sampleCount、每个image id、acquire返回index、frame/subImage/imageArrayIndex与imageRect。不能固定texture[0]，也不能只读left layer就概括全部提交。

诊断在正确的GLES线程/context、正确frame和应用合法持有图像的时点进行。`xrReleaseSwapchainImage` 后图像已交还runtime；不能仅因为EndFrame方便就认为此时仍可任意修改/读取。选择实际渲染完且release前的观察点，或与runtime合同一致的诊断方法；注意GL同步与共享context。

二维纹理用匹配target的FBO attachment；GLES数组纹理按layer使用 `glFramebufferTextureLayer`，不要 `glFramebufferTexture2D(... GL_TEXTURE_2D_ARRAY ...)`。先检查FBO complete和glGetError、采样格式及MSAA是否需resolve，失败结果不当像素数据。

自检用独立64×64纹理/FBO清为已知 `(0.25,0.5,0.75)`，RGBA8读回应约64/128/191，验证采集路径。保存/恢复被改的read/draw FBO、buffer、纹理绑定/active unit、pack alignment和pixel-pack-buffer；只有selftest完整且目标attachment/index/时点正确，才能解释读回内容。

缩小/稀疏采样用于统计时，写明采样区域和算法；统计全0可能漏掉小几何。RGB/alpha分别检查，遇到透明黑、清屏、后续覆盖、错误layer/view、sRGB编码都需结合调用序列解释。读回和文件输出会扰动帧时间，只做有限帧，交付版移除。

## IL2CPP 离线信息

用原 `libil2cpp.so` 与对应 `global-metadata.dat` dump类型/字段/方法；header整数version不等于某个工具选择的IL2CPP细分布局版本。本例实际metadata magic为0xFAB11BAF、整数version29。

历史所说“改成29.1”应理解为选择适合该Unity版本的dumper解析布局，而不是将metadata文件头改成小数。当前磁盘Dumper config已不是29.1配置，不能将早期会话设置当作仍在生效。保留原输入，使用适配的工具版本或受控配置，检查registration/method/field结果合理。

本例历史CodeRegistration0x3FD6C40、MetadataRegistration0x4115898只是原库的定位线索，不能套到别的二进制。清楚区分dump注释的RVA、ELF VA/file offset和ASLR运行地址；用当前ELF结构验证映射。

## 调用与布局的可靠性

- `il2cpp_runtime_invoke` 的MethodInfo/实例/参数与返回必须匹配真实签名；同时检查exception。method为空先查程序集/image、namespace、方法名、参数数量、具体重载和strip，不认为任何NULL都等于裁剪。
- 值类型返回为boxed对象时，用 `il2cpp_object_unbox` 读payload；不要把对象指针当int/Vector3/Scene。引用类型实例化先 `il2cpp_object_new` 再调用存在的ctor；不能向NULL实例invoke构造器。
- List实例以 `il2cpp_object_get_class` 查具体封闭类型，再访问Count等成员；List不是Il2CppArray。数组长度只用相应数组结构/API，不把容器内存当数组。
- `out/ref`结构参数提供足够大小和正确对齐的原生存储，以匹配的dump布局读；初始化缓冲区/设置sentinel辅助确认写入，但不能靠猜偏移得到的浮点数下结论。
- Android UI线程、Unity游戏主线程和Gfx渲染线程不同。按具体操作的线程要求安排，attach IL2CPP仅解决运行时线程注册，并不许可跨线程操作Unity场景。通过可靠的主线程回调执行创建/SetActive/播放等操作；渲染描述符观察跟随实际render callback。
- 不将“此现场某些OpenXR回调在Thread-4”推广为全部OpenXR函数一定在渲染线程。release没有报线程异常，也不能证明操作可靠。

本影院后续视频诊断还遇到 `FindObjectOfType(System.Type)` 与generic/bool重载混淆，必须依据签名而不是仅方法名/参数数量选择。

## 直接看原插件交给 Unity 的描述符

managed Camera 的pose/FOV看似正常而XR眼数据异常时，观察 `updateRenderPasses/updateCullPasses` 的真实输出，包括eye pose、投影/culling矩阵、viewport、纹理分配与引用。

本例build191通过原库已有动态符号/PLT重定位，将两个函数转至可记录的wrapper，从原始输出确认眼位 `(0.1,1000,-1)`；build192纠正0xe/0xf含义后恢复原 `libyvrplugin.so`。历史诊断脚本 `work/codex/patch_descriptors.py` 是针对该ELF结构的实验，不是移植必执行的补丁。

如果采用这种观测，先确认符号/relocation、原函数ABI、wrapper转发和目标布局，再校验离线修改及设备加载。ASLR基址只用于运行地址，不能当文件偏移。保存可恢复的原库，观测完成恢复并核对hash；不要把改后的原插件留在交付中。

二进制补丁尤其检查返回ABI。本例 `getResolutionScale` RVA0x6580C反汇编把 `this+0xBC` 地址放入x0返回，是 `const float&`；写 `fmov s0,#1.0; ret` 不会实现等价返回，反而会毁坏引用。返回scalar/reference、输出指针、hidden struct返回各按具体调用点确认。

## NaN 的解释边界

NaN矩阵/平面提示上游输入、布局、时间、pose或诊断可能有问题。不能简单断言“NaN一定把所有物体判到视锥外”，结果取决于具体比较和culling实现。

本例把0xe/0xf写成 `{left,right,top,bottom,near,far,1}` 后警告消失，但下游将它按quaternion+position解释，position=(0.1,1000,1)，还可能经过坐标变换成为z=-1。有限值不等于语义正确；要核对布局/单位/坐标系、四元数有效性和空间关系。最终以真实VIEW眼位恢复图像和用户6DoF验收确认修复。
