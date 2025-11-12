# Echo-XR Migration Plan

目标：将 Godot `modules/openxr` 相关实现抽离成独立的 C++ 库 `echo-xr`，用 CMake 管理构建，逐步迁移并在每个阶段保持可编译与可测试。首要任务是对需要移植的模块进行分组，并给出迁移顺序与去耦注意事项。

## 迁移阶段与模块分组

| 阶段 | 模块组 | 源文件/目录 | 主要职责 | Godot 依赖 | 迁移要点 |
| --- | --- | --- | --- | --- | --- |
| Phase 0 | 构建脚手架 | *(新建)* `CMakeLists.txt`, `src/`, `include/` | 设立 echo-xr 库目标、导入 OpenXR SDK | 无 | 引入 `thirdparty/openxr/include` 或转为外部依赖；建立基础命名空间/接口。 |
| Phase 1 | 核心运行时 | `modules/openxr/openxr_api.*`, `openxr_api_extension.*`, `openxr_util.*`, `openxr_structure.*`, `openxr_uuid.h`, `openxr_platform_inc.h` | 与 OpenXR Loader 交互、实例/Session 生命周期管理、渲染线程数据桥接 | RenderingServer、Engine、OS (`modules/openxr/openxr_api.cpp:1654`) | 抽象出最小平台接口（日志、计时、GPU 资源钩子），避免直接依赖 Godot 单例。 |
| Phase 1b | XR 接口适配 | `modules/openxr/openxr_interface.*` | Godot XRInterface 实现、动作同步、tracker/haptic 管道 (`modules/openxr/openxr_interface.cpp:669`) | XRServer、DisplayServer、Resource 系统 | 该层与 Godot 强耦合，应重写为 echo-xr 对外 API（例：C API/纯 C++ 接口）；保留逻辑但移除 Godot 具体类型。 |
| Phase 2 | 动作映射系统 | `modules/openxr/action_map/*` | 建立 ActionSet/Action/InteractionProfile 资源、默认映射 (`modules/openxr/action_map/openxr_action_map.cpp:184`) | Godot `Resource`/`Ref` | 需要以纯 C++ 数据结构重写；保留 JSON/自定义序列化，移除 Godot 资源依赖。 |
| Phase 3 | 扩展框架 | `modules/openxr/extensions/openxr_extension_wrapper.*`, `openxr_api_extension.compat.inc` | 提供扩展注册、pNext 链注入、事件回调 (`modules/openxr/extensions/openxr_extension_wrapper.h:49`) | Godot `Object`, 信号系统 | 将基类改为纯接口/抽象 C++ 类；必要时提供 RTTR/自定义反射以替代 ClassDB。 |
| Phase 3b | 核心扩展集合 | `modules/openxr/extensions/openxr_*_extension.*`（非平台特定） | 手部、眼动、Foveation、Future、Performance、Controller 等扩展 (`modules/openxr/extensions/openxr_future_extension.cpp:132`) | 依赖 Godot 信号/Resource（如 FutureResult） | 逐个评估，优先迁移渲染相关（FB Foveation、Swapchain）；将 Godot 信号改为回调或事件总线。 |
| Phase 3c | 平台图形扩展 | `modules/openxr/extensions/platform/openxr_{vulkan,metal,opengl,d3d12,android}_extension.*` | 将 OpenXR swapchain 图像映射到 Godot GPU 抽象 (`modules/openxr/extensions/platform/openxr_vulkan_extension.cpp`) | Godot 渲染服务器、平台封装 | 需要改造成与 echo-xr 外部渲染器的桥接接口，定义抽象 GPU 后端能力。 |
| Phase 4 | 空间实体与高级特性 | `modules/openxr/extensions/spatial_entities/*` | Spatial anchor/plane/marker 能力 (`modules/openxr/extensions/spatial_entities/openxr_spatial_entity_extension.cpp`) | Godot 节点/资源 | 可放入扩展包，待核心稳定后迁移。 |
| Phase 5 | 场景与可视化 | `modules/openxr/scene/openxr_*` | Composition layer 封装、render model、visibility mask (`modules/openxr/scene/openxr_composition_layer.cpp`) | Godot 场景树、RenderingServer | 需评估是否继续支持，如保留则改为与 echo-xr 客户端渲染接口通信。 |
| Phase 6 | 第三方依赖 | `thirdparty/openxr/include/openxr/*` | OpenXR headers | 无 (官方 SDK) | 使用子模块；需同步版本。 |

## 迁移顺序建议
1. **建立 echo-xr 框架**：创建 `src/` 目录与 CMake 目标，配置 OpenXR SDK、基础日志/断言工具，搭建 CI 编译脚本。
2. **抽取核心运行时**：将 `OpenXRAPI` 划分为平台中立层（日志、时间、线程）与渲染后端接口；同时实现抽象 `XrGraphicsBridge` 以替换 Godot `RenderingServer`.
3. **重新定义对外接口**：设计 `EchoXrInstance`、`EchoXrSession`、`EchoXrActionSystem` 等 API，用于取代 `OpenXRInterface` 与 Godot XRServer 交互。
4. **重写动作映射与资源系统**：使用 STL/自定义类型，提供 JSON/YAML 序列化方便工具链在 Godot 之外复用。
5. **逐步导入扩展**：按照运行时依赖关系（先 FB Foveation/Swapchain、再 hand/eye、最后 spatial），每加入一个扩展就补充 CMake 目标与测试。

## 去耦注意事项
- **Engine/Server 单例**：`OpenXRAPI` 与 `OpenXRInterface` 频繁调用 `Engine::get_singleton()`、`XRServer`、`RenderingServer`、`DisplayServer` 等 Godot 单例（例如 `modules/openxr/openxr_api.cpp:2282`, `modules/openxr/openxr_interface.cpp:705`). 迁移时需以依赖注入或抽象接口代替。
- **线程模型**：Godot 渲染线程通过宏 `ERR_NOT_ON_RENDER_THREAD` 保护；在 echo-xr 中需要自定义同步/断言机制，并向调用者暴露「渲染线程上下文」概念。
- **资源/RefCount**：大量类型继承 `Object`/`RefCounted` 并依赖 `ClassDB`，迁移后需重写为纯 C++ RAII/智能指针，或引入轻量型对象系统。
- **信号系统**：扩展与接口使用 `emit_signal` 与 `GDVIRTUAL`，需改为事件回调或观察者模式，使用第三方signal库 https://github.com/palacaze/sigslot
- **运行时配置**：原有 `ProjectSettings` (`GLOBAL_GET`) 需替换为库初始化配置结构或外部配置文件。

## 近期输出
- [x] `godot-openxr.md`（现有文件）记录了 Godot OpenXR 实现原理，作为迁移参考。
- [ ] Phase 0 脚手架：创建 `CMakeLists.txt`、`src/echo_xr/core` 目录与基本目标。
- [ ] Phase 1 核心运行时移植与最小演示。

后续每完成一个 Phase/模块分组，需：
1. 更新本计划文档的状态；
2. 调整 CMake 目标与依赖；
3. 在 echo-xr 仓库添加相应单元/集成测试，确保 `cmake --build` 通过。
