# Unreal visionOS OpenXR 迁移方案

目标：把 `Engine/Platforms/VisionOS/Plugins/Runtime/OpenXRVisionOS` 中 Epic 自研的 visionOS OpenXR runtime 和扩展插件，抽离成独立的 C++ 库，既能被 Unreal 继续复用，也能为其它引擎或原生 app 提供 Vision Pro OpenXR 接入能力。

---

## 1. 现有模块盘点

| 模块 | 作用 | 关键文件 |
| --- | --- | --- |
| 自定义 OpenXR loader/runtime | 通过 `OXRVisionOS` 命名空间劫持 `xrGetInstanceProcAddr`、维护全局/实例函数表，实现“内嵌 runtime” | `OXRVisionOS.cpp` |
| Runtime 管理 | `FOXRVisionOSInstance` 负责扩展枚举、`xrCreateInstance`/`xrDestroyInstance`、系统属性缓存等 | `OXRVisionOSInstance.{h,cpp}` |
| Session/Space/Swapchain | `FOXRVisionOSSession` 映射 Session 状态机→ARKit、`cp_layer_renderer`，驱动 Metal swapchain；`FOXRVisionOSSpace`/`FOXRVisionOSSwapchain` 管理空间句柄与纹理队列 | `OXRVisionOSSession.{h,cpp}`, `OXRVisionOSSpace.*`, `OXRVisionOSSwapchain.*` |
| Render Bridge | `FOXRVisionOSRenderBridge` 暴露 `XrGraphicsBindingOXRVisionOSEPIC`、HDR 信息等 | `OXRVisionOS_RenderBridge.cpp` |
| Action/Input | `FOXRVisionOSAction(Set)`、`FOXRVisionOSController`、`FOXRVisionOSTracker` 处理控制器/手势 | `OXRVisionOSAction*.cpp`, `OXRVisionOSController.*`, `OXRVisionOSTracker.*` |
| 平台工具链 | `OXRVisionOSPlatformUtils.*`, `MetalRHIVisionOSBridge.*`, `OXRVisionOS_openxr_platform.h`（自定义 OpenXR 扩展定义） |

依赖：ARKit (`ar_session_create` 等)、`cp_layer_renderer` Swift 接口、Metal RHI、苹果专用 XrStructureType 常量。

---

## 2. 目标库结构建议

```
VisionOSXR/
 ├─ include/
 │   ├─ VisionOSXRRuntime.h         // 导出的 C API：xrGetInstanceProcAddr、启动/关闭
 │   ├─ VisionOSXRRenderer.h        // 渲染桥/HDR 数据
 │   └─ VisionOSXRPlatform.h        // OpenXR 扩展枚举、cp_layer/Metal 句柄
 ├─ src/core/                       // 从 OXRVisionOS.cpp/Instance.cpp 分离
 ├─ src/session/                    // Session + Space + Swapchain + Tracker
 ├─ src/render/                     // Render bridge，Metal 互操作
 ├─ src/input/                      // Action/Controller/HandTracking
 ├─ samples/                        // 纯 C++ / 其他引擎集成示例
 └─ CMakeLists.txt                  // 主构建脚本
```

关键点：
1. **库导出接口**：提供 `VisionOSXR_Bootstrap()`、`VisionOSXR_Shutdown()`、`VisionOSXR_GetInstanceProcAddr()`，供主机引擎注册 loader。
2. **可选编译宏**：`VISIONOSXR_WITH_UNREAL` 启用 IOpenXRExtensionPlugin 适配；默认只暴露纯 OpenXR runtime。
3. **资源拥有权**：Session/Swapchain 对象生命周期完全由库内部控制，通过回调（例如 `FVXRCustomHooks`）向宿主通知事件。
4. 剔除所有对Unreal的依赖

---

## 3. 迁移步骤

### 3.1 抽离核心 runtime
1. 复制 `OXRVisionOS.cpp` → `src/core/VisionOSXRRuntime.cpp`，删去 Unreal `IMPLEMENT_MODULE`、`IModularFeatures` 相关代码，改为普通 C API。
2. `GlobalFunctionMap/InstanceFunctionMap` 改成 `std::unordered_map<std::string, PFN_xrVoidFunction>`；提供初始化函数 `BuildFunctionMaps()` 在 `VisionOSXR_Bootstrap()` 中调用。
3. `FOXRVisionOS` 类拆成：
   - `VisionOSXRRuntime`（无 Unreal 依赖）。
   - `VisionOSXRUnrealExtension`（保留原 IOpenXRExtensionPlugin 实现，放在 bindings/unreal）。
4. 保留 `GetCustomLoader` 逻辑：返回 `VisionOSXR_GetInstanceProcAddr`。

### 3.2 Session/Space/Swapchain 独立
1. 将 `FOXRVisionOSSession` 及相关类移动到 `src/session`，替换 Unreal 类型：
   - `TSharedPtr` → `TSharedPtr`/`std::shared_ptr`（视引擎需求，可提供可编译宏）。
   - `FWorldDelegates`、`UE_LOG` 等封装为抽象回调 `IVXREngineHooks`，由宿主实现。例如：`OnWorldTickStart` 改为宿主在每帧调用 `VisionOSXR_TickWorld(double DeltaSeconds)`。
2. `cp_layer_renderer` 访问集中到 `VisionOSXRLayerHost` 类，负责：
   - 创建/销毁 layer renderer。
   - 暴露事件（paused/running/invalidated）。
   - 与 Metal 纹理队列交互。
3. 将 `MetalRHIVisionOSBridge` 依赖隔离在 `src/render`，并通过接口 `IVXRMetalBridge` 注入 RHI 命令队列/纹理句柄。

### 3.3 Render Bridge & Graphics Binding
1. `FOXRVisionOSRenderBridge` 改写为 `VisionOSXRRenderBridge`，不依赖 Unreal `FOpenXRRenderBridge`。
2. 对外导出 `VisionOSXR_GetGraphicsBinding(void* OutBinding)`，让宿主在调用 `xrCreateSession` 前拿到 `XrGraphicsBindingOXRVisionOSEPIC`。
3. HDR/色域查询改为 `VisionOSXR_QueryDisplayCaps(VXROutputCaps&)`。

### 3.4 输入与追踪
1. `FOXRVisionOSTracker`、`FOXRVisionOSController`、行动作类迁移至 `src/input`。
2. 将 Unreal `FTransform`/`FQuat` 替换为自定义数学类型或 glm，宿主（包括 Unreal）通过适配层转换。
3. 保持 OpenXR Action 接口不变，对外暴露 `VisionOSXR_CreateActionSet` 等包装函数，或继续在 Session 内部处理，再由 API 层提供 `xr*` 调用。

### 3.5 宿主接入（引擎 / 原生 App）
1. 在 `bindings/native` 提供最小示例：应用层加载库 → 调用 `VisionOSXR_Bootstrap()` → 按标准 OpenXR 初始化流程（xrCreateInstance → xrGetSystem 等）。
2. 提供宿主接口示例（如 `IVXREngineHooks`）描述如何在自己的帧循环中调用 `VisionOSXR::PumpWorldTick()`、如何把 Metal 设备/命令队列句柄传入。
3. 对于不同渲染器，`VisionOSXRRenderBridge` 需要导出 Acquire/Release/Present API，宿主在渲染线程使用。

---

## 4. 构建与依赖

| 组件 | 依赖 | 说明 |
| --- | --- | --- |
| Core runtime | C++17、OpenXR headers | 可直接使用 `Engine/Source/ThirdParty/OpenXR/include` 或官方 SDK |
| Session/ARKit | `ARKit.framework`, `RealityKit`, `Metal`, `MetalKit`, `CoreVideo` | 需要 ObjC++ 桥接层，可放在 `src/objc` |
| Swift layer renderer | `cp_layer_renderer` Swift 模块 | 保持原 `OXRVisionOSAppDelegate`、Swift 辅助文件，改为独立 package |
| 宿主接口 | 自定义 Hook（如帧循环、日志、线程） | 通过头文件定义宿主需实现的回调 |

建议使用 **CMake** 作为主构建系统；若某宿主需要其他构建系统，可链接生成的静态/动态库。对 Swift 依赖可使用 Xcode toolchain 编译成 `.a` / `.framework`，再链接给主库。

---

## 5. API 设计示例

```cpp
// VisionOSXRRuntime.h
namespace VisionOSXR
{
    struct FVXRConfig
    {
        void* SwiftLayerRenderer = nullptr;
        void* MetalDevice = nullptr;
        bool  bEnableEyeTracking = true;
        // ...更多开关
    };

    bool Bootstrap(const FVXRConfig& Config);
    void Shutdown();

    PFN_xrGetInstanceProcAddr GetInstanceProcAddr();

    // RHI 接口
    bool GetGraphicsBinding(XrGraphicsBindingOXRVisionOSEPIC& OutBinding);
    bool QueryDisplayCaps(struct FVXRDisplayCaps& OutCaps);

    // 调试辅助
    void PumpWorldTick();          // 原 OnWorldTickStart
    void NotifyAppBackgrounded();  // 映射 session state
}
```

---

## 6. 迁移风险与验证

1. **线程模型差异**：Unreal 依赖 Game/Render/RHI 三线程；独立库需支持宿主自定义线程。建议将 `xrWaitFrame`、`xrBeginFrame`、`xrEndFrame` 的调用限制与当前实现一致，并在文档中明确多线程锁。
2. **Metal/Swift 互操作**：现实现 heavily 依赖 Unreal RHI 工具（`FRHICommandList`, `ENQUEUE_RENDER_COMMAND`）。迁移时需抽象出 `IRenderBackendHooks`，否则其他宿主无法正确同步 GPU。
3. **ARKit session 生命周期**：UE 通过世界 Tick 管理 cp_layer 状态，抽离后需要一个显式 loop（例如 `VisionOSXR::PumpWorldTick()`）。宿主必须保证在 UI RunLoop/SceneKit delegate 中调用。
4. **代码许可**：确认 Epic 对 visionOS 模块的许可条款（大部分 Unreal 源码为 EULA，不能直接重分发）。若目标是内部复用，可保留；若要公开，需要法律评估。

验证建议：
1. **单元测试**：对 `VisionOSXRInstance`/`Session` 编写模拟测试，确保状态迁移与原实现一致。
2. **集成测试**：编写 minimal sample（或接入目标引擎），渲染旋转立方体，验证 `xrWaitFrame`/`xrBeginFrame`/`xrEndFrame` 与 ARKit/Metal 协同。
3. **性能回归**：对每帧 `cp_layer_renderer` latency、swapchain acquire/release 时间进行对比。

---

## 7. 里程碑

1. **Phase 0：代码拆分**
   - 建立新仓库，导入现有源码。
   - 去除对 Unreal 的直接依赖，替换为抽象回调/宏。
2. **Phase 1：独立编译**
   - 完成 CMake 构建，编译成静态库/动态库。
   - 通过 dummy host 加载库，跑 `xrEnumerateApiLayerProperties` 等 smoke test。
3. **Phase 2：宿主示例**
   - 发布原生示例或目标引擎插件，演示如何消费该库。
   - 跑渲染/输入/手势回归。
4. **Phase 3：文档与发布**
   - 输出宿主集成指南、API 参考。
   - 发布 sample app / SDK 包。

---

## 8. 结论

通过把 visionOS OpenXR runtime 抽离为独立库，可以：
- 在不同渲染引擎或原生应用间共享对 Vision Pro 的支持，减少重复实现。
- 为未来支持 visionOS 原生应用提供基础设施，同时降低依赖特定引擎的耦合。

迁移过程中需重点关注 RHI/ARKit 交互抽象、线程模型、许可问题，并通过阶段性验证保证行为与当前 Unreal 内置实现一致。
