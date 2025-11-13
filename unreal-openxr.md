# Unreal Engine OpenXR 技术报告

## 1. 模块拓扑与职责
- **核心加载与XR系统**：`FOpenXRHMDModule` 负责发现/加载 OpenXR runtime、选择可用 API 版本、启用扩展并创建 `FOpenXRHMD` 实例，同时联动 AR 模块（`IOpenXRARModule`）一同初始化（`Engine/Plugins/Runtime/OpenXR/Source/OpenXRHMD/Private/OpenXRHMDModule.cpp:66-97`）。PreInit 阶段会短暂创建实例用于 Oculus 音频 GUID 查询，并在配置允许时销毁复用（同文件:99-137）。
- **渲染桥接**：模块根据当前 RHI 选择 D3D11/D3D12/Vulkan/OpenGL/OpenGLES 专用 `FOpenXRRenderBridge`，每个桥接对象都实现获取图形绑定和 swapchain 构造逻辑（`OpenXRHMDModule.cpp:380-415` 与 `OpenXRHMD_RenderBridge.cpp`）。
- **OpenXRHMD 核心**：`FOpenXRHMD` 实现 `IXRTrackingSystem`，负责 session 生命周期、视图配置、帧管线、跟踪空间与设备管理。构造函数会登记 `/user/head` 空间，并让所有 `IOpenXRExtensionPlugin` 绑定委托与 RHI 需求（`OpenXRHMD.cpp:1708-1719`）。
- **输入子系统**：`OpenXRInput` 读取 Enhanced Input 配置，将 `UInputAction`/`UInputMappingContext` 转换为 OpenXR ActionSet、Action，利用 `IOpenXRHMD` 向设备列表添加 Pose/Haptics 动作并允许扩展插件在创建 Action/ActionSet 时插入链表节点（`OpenXRInput.cpp:82-190`、`239-247`）。
- **AR 子系统**：`OpenXRAR` 通过 `SetTrackingSystem` 获得 `IOpenXRHMD`，遍历扩展插件获取自定义 Anchor/Capture 能力，并在开始/停止/暂停 AR Session 时广播给插件（`OpenXRAR.cpp:40-136`）。
- **扩展插件生态**：任何模块可实现 `IOpenXRExtensionPlugin` 注册为 `OpenXRExtension` modular feature，参与 loader 替换、API Layer 插入、扩展声明、Session/Frame 各阶段回调，以及可选的 Anchor/Capture/ARPin/HMD 特性支持（`IOpenXRExtensionPlugin.h:122-205`）。例如 HandTracking 模块声明 `XR_EXT_hand_tracking`、缓存系统属性并在 `OnCreateSession` 中构建手部跟踪器（`OpenXRHandTracking.cpp`）。

## 2. 实例与扩展协商
1. **Loader 选择**：模块优先尝试 `GetDefaultLoader`，按平台加载 Epic 自带 loader 或动态库，并在 Android 上执行 `xrInitializeLoaderKHR` 绑定 JVM 环境（`OpenXRHMDModule.cpp:418-479`）。
2. **扩展过滤**：`EnableExtensions` 校验插件声明的必选/可选扩展，仅当必选全部可用才保持插件（同文件:482-520）。
3. **API 版本回退**：`TryCreateInstance` 会根据项目设置顺序尝试 1.1→1.0，失败时打印可选扩展/层列表。若遇到 `XR_ERROR_EXTENSION_NOT_PRESENT`，会依据配置的“问题 Layer”组合禁用部分扩展并重试，必要时还会临时剔除依赖被禁用扩展的插件（同文件:900-1132）。
4. **系统发现**：`GetSystemId` 调用 `xrGetSystem` 之前允许插件扩展 `XrSystemGetInfo`（同文件:1169-1183），并缓存 `XrSystemId` 供 HMD 实例复用。

## 3. Session 构建与资源准备
1. **视图与 Blend 配置**：`OnStereoStartup` 获取系统属性、枚举支持的 `XrViewConfigurationType` 并应用 cvar/运行时偏好，派生每眼推荐分辨率与最大像素密度，同时挑选 runtime 首选 `XrEnvironmentBlendMode`（`OpenXRHMD.cpp:2100-2174`）。
2. **Session 创建**：构建 `XrSessionCreateInfo`，将渲染桥的图形绑定和插件 `OnCreateSession` 串入 `next` 链，之后 `xrCreateSession` 成功后遍历插件执行 `PostCreateSession`（同文件:2177-2210）。
3. **参考空间**：枚举 reference space 并创建 `XR_REFERENCE_SPACE_TYPE_VIEW` 等空间对象。每个被跟踪的设备（HMD + 控制器）有对应 `FDeviceSpace`，Session 创建后立即 `CreateSpace`（同文件:2212-2478）。
4. **IOpenXRInputModule 协同**：`StartSession` 仅在 runtime 报 READY 状态后调用，先通知输入模块再执行 `xrBeginSession`，并允许插件在 `Begin.next` 链上附加结构体（同文件:2564-2584）。

## 4. Session 状态机与事件处理
- `OnStartGameFrame` 是状态驱动中心：轮询 `xrPollEvent`，在 READY/SYNCHRONIZED/IDLE/STOPPING/EXITING/LOSS_PENDING 等状态下切换本地 flag、调节帧率、触发 `StartSession`/`StopSession`，必要时调用 `DestroySession` 并在需要时 `RequestExitApp`（`OpenXRHMD.cpp:3596-3715`）。
- 参考空间变更、交互 profile 变化、可见性 mask 更新等事件也在此处转发给引擎/插件（`OpenXRHMD.cpp:3715-3758`）。
- 当 session 丢失或 runtime 要求退出时，模块会销毁 swapchain、空间与 Session 句柄，并重置 pipeline 状态/VRFocus 标志（`OpenXRHMD.cpp:2401-2442`、`3704-3715`）。

## 5. 多线程帧循环与管线
1. **三线程管线状态**：`FPipelinedFrameState` 在游戏、渲染、RHI 三份缓存之间轮转；`GetPipelinedFrameStateForThread` 根据调用线程返回正确的带锁访问器，保证 View/TrackingSpace/预测显示时间在各线程一致（`OpenXRHMD.cpp:1765-1797`）。
2. **WaitFrame（游戏线程）**：`OnBeginSimulation_GameThread` 中调用 `xrWaitFrame`，记录 `WaitCount`、预测显示时间与 `XrFrameState`，必要时重新创建 tracking space，并更新 `EnumerateViews`（`OpenXRHMD.cpp:3520-3576`）。
3. **渲染线程同步**：`OnBeginRendering_GameThread` 将游戏态数据移动到渲染态，同时根据 cvar 决定是在渲染线程还是更晚的 RHI 线程刷新设备 pose 以支持 Late Update，并设置哪些层需要提交（`OpenXRHMD.cpp` 中 `OnBeginRendering_GameThread` 段）。
4. **RHI 线程**：`OnBeginRendering_RHIThread` 通过 WaitCount 防止额外的 `xrBeginFrame`，必要时插入 `XR_TYPE_RHI_CONTEXT_EPIC`，触发插件 `OnBeginFrame_RHIThread`，然后才 `xrBeginFrame` 并在需要时等待/获取 swapchain 图像（`OpenXRHMD.cpp:3837-3945`）。`OnFinishRendering_RHIThread` 负责释放 swapchain 图像、拼装背景层/深度层/模拟面锁层/插件层，并最终 `xrEndFrame`（`OpenXRHMD.cpp:3948-4020` 及后续）。
5. **Swapchain/Layer 管理**：`OpenXRHMD_Layer`/`OpenXRHMD_Swapchain` 支持多种 RHI 的格式协商与 face-locked layer 模拟；`PipelinedLayerState` 记录当帧需要提交的原生/模拟层以及颜色缩放、深度测试、Alpha 反转等参数。
6. **帧内 Foveation 与 Late Update**：若 runtime + 项目开启 foveation，`FBFoveationImageGenerator` 在 RHI 线程设置当前 swapchain 索引；Late Update 则在渲染线程应用最新 HMD pose 并通过 `GraphBuilder` 将投影层复制到 RHI 状态（`OpenXRHMD.cpp:3308-3354` 一带）。

## 6. 设备与输入跟踪
- **DeviceSpaces**：HMD + 每个注册的 Action 设备均保留 `FDeviceSpace`（含 `XrSpace`、Action/Path/Subaction）。`AddTrackedDevice` 允许输入模块把 Grip/Aim/Palm 等动作绑定到具体路径/子路径，在 Session 创建后会立即 `CreateSpace`（`OpenXRHMD.cpp:2433-2478`）。
- **姿态更新**：`UpdateDeviceLocations` 在拥有有效 TrackingSpace 且 WaitFrame 成功后，遍历 `DeviceSpaces` 调用 `xrLocateSpace` 缓存 `XrSpaceLocation`，可选择是否通知扩展插件用于更复杂的跟踪需求（`OpenXRHMD.cpp:1800-1860`）。
- **输入映射**：`OpenXRInput` 将 Enhanced Input 类型映射到 XR ActionType，支持 ActionSet 优先级（`OpenXRInput.cpp:48-190`）。控制器对象会为 Grip/Aim/Palm/Vibration 创建 Pose/Haptic 动作，并通过 `IOpenXRHMD::AddTrackedDevice` 获取 deviceId，以便在 `UpdateDeviceLocations` 阶段读取姿态（`OpenXRInput.cpp:198-247`）。

## 7. AR/Anchors/捕获集成
- `OpenXRAR` 实现 `IOpenXRARSystem`，其 `SetTrackingSystem`/`OnStartARSession`/`OnStopARSession` 会把 AR Session 生命周期分发给每个扩展插件，从而让平台/供应商插件在 ARPins、QR/Camera/SceneUnderstanding 等管线中注入自身实现（`OpenXRAR.cpp:40-136`）。
- 插件可返回 `IOpenXRCustomAnchorSupport`/`IOpenXRCustomCaptureSupport`，用于将 Unreal ARPin 映射到 runtime 空间、提供相机纹理/射线追踪、或接入硬件层持久化（`IOpenXRExtensionPlugin.h:17-103`）。

## 8. 可扩展性与平台特化
- 通过 `IOpenXRExtensionPlugin`，供应商（如 Oculus、Meta、Valve、visionOS、自研平台）可以：
  - 提供自定义 loader 或 API layer（`IOpenXRExtensionPlugin.h:153-205`）。
  - 声明额外渲染桥（比如 visionOS 的 `FOXRVisionOSRenderBridge`）。
  - 在 Session/Frame 生命周期各阶段插入结构体（`OnCreateInstance / OnGetSystem / OnCreateSession / OnBeginSession / OnWaitFrame / OnBeginFrame / OnEndFrame / UpdateCompositionLayers` 等）。
  - 暴露手部追踪、眼动追踪、Vive Tracker 或厂商特化的输入 profile，这些模块都位于 `Engine/Plugins/Runtime/OpenXR*` 目录下并通过上述接口与核心 OpenXRHMD 解耦。

## 9. visionOS 支持实现
- 通过平台插件 `Engine/Platforms/VisionOS/Plugins/Runtime/OpenXRVisionOS`，Unreal 在 visionOS 上自带一个“内建 runtime”，拦截所有 OpenXR API 入口（`xrGetInstanceProcAddr/xrCreateInstance` 等）并转发到 `FOXRVisionOSInstance/FOXRVisionOSSession`；插件在 `OXRVisionOS` 命名空间维护全局/实例函数表，实现一个嵌入式 OpenXR loader，使 visionOS 上缺失的系统 runtime 由 UE 自己托管（`Engine/Platforms/VisionOS/Plugins/Runtime/OpenXRVisionOS/Source/OXRVisionOS/Private/OXRVisionOS.cpp:58-200`）。
- Session 封装层 (`FOXRVisionOSSession`) 将 OpenXR 状态机映射到 Apple 的 `cp_layer_renderer` 与 ARKit `ar_session_create()`；在 World Tick 中轮询 layer renderer state，只有当 visionOS SceneKit 层进入 running 才将 Session 置为 READY 并允许 `xrBeginSession`，从而保证系统空间与 Unreal pipeline 同步（`OXRVisionOSSession.cpp:120-177`）。同一模块还内置了 gaze tracker、控制器姿态调整、与 Metal RHI 的同步栅栏（`OXRVisionOSSession.cpp:74-173`、`343-420`）。
- 渲染路径通过专属 `FOXRVisionOSRenderBridge`，它自定义 `XrGraphicsBindingOXRVisionOSEPIC`，让 `xrCreateSession` 时能够接入 Apple 的 `cp_layer` 纹理队列，并声明颜色域/DCI-P3 输出与 HDR 能力；所有 swapchain 创建都走 visionOS 专用分支（`OXRVisionOS_RenderBridge.cpp:14-48`、`OXRVisionOSSession.cpp:196-250`）。
- 该插件与主 OpenXR 模块协同方式是：平台扩展包 `OpenXR_VisionOS.uplugin` 仅在 VisionOS 目标上启用 OpenXRHMD/Input/AR 模块，确保其他平台的 loader 逻辑不受影响（`Engine/Platforms/VisionOS/Plugins/Runtime/OpenXR/OpenXR_VisionOS.uplugin:1-29`）。FOpenXRHMDModule 仍负责实例/扩展协商，但其 `GetCustomRenderBridge`/`GetCustomLoader` 会从 visionOS 插件拿到上述自定义实现以完成整合。
- 帧循环中，FoxrvSession 同时要管理苹果侧 ARKit 会话和 Unreal `PipelinedFrameState`：比如在 `OnWorldTickStart` 与 `OnFinishRendering_RHIThread` 里分别等待 visionOS layer renderer、在 RHI 线程彻底排空 GPU workload 后再回收 swapchain，避免 Vision Pro compositor 与 Unreal 互相阻塞（`OXRVisionOSSession.cpp:120-210`, `OXRVisionOSSession.cpp:420-520`）。

## 10. Session 生命周期梳理
1. **引擎启动**：`FOpenXRHMDModule::CreateTrackingSystem` 调用 `InitInstance`（若需要则先运行 `PreInit`），加载扩展插件并创建 `FOpenXRHMD` + `ARSystem`。
2. **选择图形/RHI**：RenderBridge 创建后记录 GPU LUID，确保后续 `GetGraphicsAdapterLuid` 可以和 runtime 要求匹配。
3. **用户进入XR**：`OnStereoStartup` 依据设备/项目配置建立 Session，并根据 `XR_TYPE_EVENT_DATA_SESSION_STATE_CHANGED` 等事件在游戏线程推动 `StartSession`。
4. **帧循环**：
   - 游戏线程：`xrWaitFrame` → 更新 TrackingSpace/ViewConfig → 调度渲染线程。
   - 渲染线程：转存帧状态、运行 Late Update、通知扩展层准备。若启用面锁层仿真，会把 emulated layer 合成到后台 swapchain。
   - RHI 线程：`xrBeginFrame` → Acquire swapchain → 触发插件 → 在 `OnFinishRendering_RHIThread` 收尾、拼装 layer header → `xrEndFrame`。
5. **暂停/摘头显**：Session 状态变为 IDLE/STOPPING 时限制帧率、清理 layer/空间；LOSS_PENDING/EXITING 时销毁 Session，并在配置允许下调用 `RequestExitApp`，必要时等待新的系统/Session 重建。

## 11. 关键实现特性总结
- **三段流水**：`WaitFrame`、`BeginFrame`、`EndFrame` 分别绑定于游戏/渲染/RHI，辅以 `WaitCount/BeginCount/EndCount` 保证调用配对，避免 SteamVR 上的 `xrWaitFrame` 饿死（`OpenXRHMD.cpp:3837-3945`）。
- **可编程 Layer 链**：借助 `next` 指针串联扩展结构（如 `XR_FB_composition_layer_alpha_blend`、`XR_COMPOSITION_LAYER_DEPTH_TEST_FB`），并允许插件添加自有 layer（`OpenXRHMD.cpp:3948-4020` 以及 `AddLayersToHeaders` 调用）。
- **设备焦点与应用生命周期**：当 runtime 触发 EXITING/LOSS_PENDING/INSTANCE_LOSS 事件时，模块会重置 VRFocus、释放 Session，并视 `xr.OpenXRExitAppOnRuntimeDrivenSessionExit` 决定是否退出应用（`OpenXRHMD.cpp:3655-3715`）。
- **输入/AR 解耦**：通过 `IOpenXRHMD` 暴露的少量函数（获取 Session、Space、DeviceId、Extension 列表等），输入与 AR 模块都不需要直接依赖 `FOpenXRHMD`，从而易于维护和平台扩展。

本报告覆盖了核心模块间的依赖关系、扩展机制以及 OpenXR Session 的完整生命周期，可作为团队在二次开发或平台适配时的参考。
