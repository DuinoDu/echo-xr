# Godot OpenXR 模块技术梳理

## 模块布局与职责
- 模块入口由 `initialize_openxr_module()` 注册运行时单例、扩展封装、场景节点以及编辑器插件，确保在 Core/Servers/Scene/Editor 不同初始化阶段安装对应能力 (`modules/openxr/register_types.cpp:124`).
- 核心运行时代码集中在 `OpenXRAPI`/`OpenXRInterface`，分别负责 OpenXR loader 与 Godot XRInterface 适配；输入资源、扩展、场景节点则放在 `modules/openxr/action_map`, `modules/openxr/extensions`, `modules/openxr/scene`, `modules/openxr/editor` 等子目录 (`modules/openxr/register_types.cpp:239`).
- 默认项目设置（例如 `xr/openxr/enabled`, `xr/openxr/default_action_map`）直接由 `OpenXRAPI` 读取，用于决定是否在启动时创建实例与自动生成动作映射资源 (`modules/openxr/openxr_api.cpp:300`, `modules/openxr/openxr_api.cpp:304`).

## 核心运行时（OpenXRAPI）
- `OpenXRAPI` 将实例、系统、视图配置、参考空间、swapchain 以及扩展列表等状态集中在单例里，并通过 `OpenXRGraphicsExtensionWrapper` 适配不同渲染后端；同时维护跨线程共享的 `render_state` 与 `OpenXRSwapChainInfo`，保证主线程与渲染线程之间只通过命令队列交换数据 (`modules/openxr/openxr_api.h:52`).
- 初始化流程 `initialize()` 选择图形扩展、让所有 `OpenXRExtensionWrapper` 先执行 `on_before_instance_created()`，随后枚举 API layer、扩展、创建实例、解析符号、读取系统与视图配置；若任一步失败则销毁实例回滚 (`modules/openxr/openxr_api.cpp:1654`).
- `initialize_session()` 负责 `xrCreateSession`、枚举参考空间与 swapchain 格式、创建 view space，并在引擎侧分配 view/projection/depth 缓冲区；只有成功后 `OpenXRInterface` 才能同步 Godot 视图尺寸 (`modules/openxr/openxr_api.cpp:1749`).
- 运行时维护的 `render_state` 通过 `_set_render_*_rt()` 系列函数仅允许渲染线程写入，确保如预测显示时间、play space、渲染区域等数据不会被主线程直接篡改 (`modules/openxr/openxr_api.cpp:2114`).
- 默认动作映射资源名来自项目设置，并在首次运行时生成 `OpenXRActionMap` Tres，保证没有自定义文件时也能创建标准交互 (`modules/openxr/openxr_api.cpp:304`).

## XR 接口、输入与动作系统
- `OpenXRInterface::_load_action_map()` 会清空旧 tracker/动作集，加载或创建 `OpenXRActionMap` 资源，并把每个 `OpenXRAction` 的 top-level 路径映射到 Godot 自定义 `Tracker`；不存在的路径会跳过以支持跨平台运行 (`modules/openxr/openxr_interface.cpp:229`).
- `OpenXRActionMap::create_default_action_sets()` 定义默认 action set、trigger/grip/joystick/按钮/pose/haptic 等动作，并附带 Vive Tracker、眼球等扩展路径，覆盖常见控制器 (`modules/openxr/action_map/openxr_action_map.cpp:184`).
- `OpenXRInterface::initialize()` 在加载动作后启动 OpenXR session、注册 head tracker、把 action sets attach 到 session，并向 XRServer/DisplayServer 注册自身以成为主接口和额外输出 (`modules/openxr/openxr_interface.cpp:669`).
- 每帧 `OpenXRInterface::process()` 调用 `OpenXRAPI::process()` 以驱动 xrWaitFrame、更新头部 pose，再调用 `sync_action_sets()`、`handle_tracker()` 将布尔/浮点/Vector2/pose 输入写入 Godot 的 `XRControllerTracker`，并驱动手部追踪扩展 (`modules/openxr/openxr_interface.cpp:1187`, `modules/openxr/openxr_interface.cpp:556`, `modules/openxr/openxr_interface.cpp:1145`).
- Haptic、手势、追踪器 profile 变更等都委托给 `OpenXRAPI` 的 action/tracker RID 接口，使动作系统与 OpenXR 原生资源一一对应 (`modules/openxr/openxr_interface.cpp:606`).

## 扩展体系与管理
- `OpenXRExtensionWrapper` 是所有扩展的基类，定义了请求扩展名、将自定义结构插入 `pNext` 链、响应生命周期事件、提供 composition layer/帧信息/事件回调等钩子，并可暴露给 GDExtension (`modules/openxr/extensions/openxr_extension_wrapper.h:49`).
- 模块初始化时根据平台与项目设置注册内置扩展（Palm pose、Foveation、Hand tracking、Spatial Entity、调试工具、绑定修饰器等），并在 `OpenXRAPI::register_extension_wrapper()` 里记录；实例创建后不可再增删 (`modules/openxr/register_types.cpp:140`, `modules/openxr/openxr_api.cpp:1790`).
- `OpenXRAPI::get_all_requested_extensions()`/`create_instance()` 会汇总所有 wrapper 申明的扩展，自动剔除运行时不支持的可选扩展或在缺失必需扩展时直接报错；同一扩展可被多个 wrapper 请求，`enabled_extensions` 去重后传入 `xrCreateInstance` (`modules/openxr/openxr_api.cpp:535`, `modules/openxr/openxr_api.cpp:553`).
- 所有 wrapper 能在实例/会话创建流程里追加自定义 `pNext` 结构，如系统属性、session create info、swapchain create info 等，从而在 Godot 内部透明启用 vendor feature (`modules/openxr/extensions/openxr_extension_wrapper.h:70`).
- 通过 `OpenXRAPIExtension` 暴露的 API，GDScript/GDExtension 可在运行时注册 composition layer provider、projection-view 扩展、帧信息扩展、swapchain 工具等，方便第三方实现私有渲染管线或调试工具 (`modules/openxr/openxr_api_extension.h:43`, `modules/openxr/extensions/openxr_extension_wrapper.h:162`).
- 场景层的 `OpenXRCompositionLayer*`、`OpenXRRenderModel`、`OpenXRVisibilityMask` 等节点最终都借助上述 extension API 将额外的 XR composition layer 或模型数据推入 `OpenXRAPI` (`modules/openxr/register_types.cpp:258`).

## 私有与供应商扩展
- 供应商扩展通常自带元数据注册：例如 `OpenXRMxInkExtension` 请求 Logitech 专用扩展并向 `OpenXRInteractionProfileMetadata` 注册 stylus 的 IO 路径，使动作地图可以绑定厂商按钮/pose (`modules/openxr/extensions/openxr_mxink_extension.cpp:38`, `modules/openxr/action_map/openxr_interaction_profile_metadata.h:25`).
- `OpenXRFutureExtension` 封装 XR_EXT_future：它在实例创建时解析 `xrPollFutureEXT/xrCancelFutureEXT`，在 `on_process()` 中轮询 future 状态，并把完成的结果回调到 Godot `OpenXRFutureResult` 资源；session 销毁时会自动取消仍在运行的 future (`modules/openxr/extensions/openxr_future_extension.cpp:132`, `modules/openxr/register_types.cpp:166`).
- 空间实体相关的 `OpenXRSpatialEntityExtension` 及 anchor/plane/marker capability wrapper 作为单例注册，暴露资源与节点方便脚本访问特定厂商的空间锚或 Marker 跟踪 (`modules/openxr/register_types.cpp:176`).
- 其他平台/厂商扩展（Pico/Meta/HTC/Huawei/Valve/WMR 等控制器、Palm pose、本地地板、Foveation/Swapchain 扩展）也通过 wrapper 形式集中维护，必要时由项目设置开关控制加载 (`modules/openxr/register_types.cpp:147`).

## Session 生命周期与帧同步
- OpenXR 事件在 `poll_events()` 中统一处理：对实例/会话状态改变触发 `on_state_*()`，并把引用空间变更、交互 profile 变化等事件分发给 `OpenXRInterface` 与扩展 (`modules/openxr/openxr_api.cpp:1997`, `modules/openxr/openxr_api.cpp:1388`).
- 每帧主线程 `OpenXRAPI::process()` 调用 `xrWaitFrame` 并维护 play space / local-floor 模拟、预测显示时间、frame info 扩展链，随后广播 `on_process()` 给扩展 (`modules/openxr/openxr_api.cpp:2206`, `modules/openxr/openxr_api.cpp:939`).
- 渲染线程在 `pre_render()` 中创建/重建 swapchain、调用 `xrLocateViews` 更新 `render_state.views`，并通知扩展在真正渲染前注入逻辑 (`modules/openxr/openxr_api.cpp:2282`).
- `pre_draw_viewport()`/`post_draw_viewport()` 负责获取 Godot 渲染目标、acquire-release swapchain image，同时让扩展在 viewport 渲染前后执行附加操作 (`modules/openxr/openxr_api.cpp:2380`, `modules/openxr/openxr_api.cpp:2471`).
- `end_frame()` 根据 `render_state`、扩展生成的 composition layers 组合最终层列表，附带投影层/深度层/自定义层排序后调用 `xrEndFrame`，并清理 swapchain 镜像；若当前帧无需渲染则提交空层 (`modules/openxr/openxr_api.cpp:2484`).
- 状态切换（READY/VISIBLE/FOCUSED/STOPPING etc.）会影响 `running` 标志与渲染线程同步，并通过 XRInterface 信号让游戏逻辑感知 Session 生命周期 (`modules/openxr/openxr_api.cpp:1388`, `modules/openxr/openxr_interface.cpp:1383`).

## 配置与工具链
- 项目设置控制 OpenXR 是否启用、需要的扩展与绑定修饰器，这些设置会在模块初始化时决定是否注册对应 wrapper，从而避免在不支持的平台加载无效扩展 (`modules/openxr/register_types.cpp:193`).
- `OpenXRActionMap`、`OpenXRInteractionProfile`、`OpenXRBindingModifier` 均为资源，可在编辑器通过 `openxr_action_map_editor` 等 UI 进行可视化管理；编辑器模式下会调用 `create_editor_action_sets()` 以便在无设备情况下展示默认动作 (`modules/openxr/editor/openxr_action_map_editor.cpp:496`, `modules/openxr/openxr_interface.cpp:247`).
- `OpenXRInterface` 在初始化时向 DisplayServer 注册额外输出，确保即便窗口最小化 XR 仍然渲染；同时暴露 API 让脚本调整渲染尺寸、环境混合模式、Foveation、速度贴图等 XR 特性 (`modules/openxr/openxr_interface.cpp:705`, `modules/openxr/openxr_interface.cpp:1018`, `modules/openxr/openxr_interface.cpp:1361`).

## 数据流摘要
- 启动阶段根据项目设置注册扩展 → `OpenXRAPI` 创建实例/Session → `OpenXRInterface` 构建并 attach 动作集/Tracker → 每帧主线程 `process()` 驱动 xrWaitFrame 和输入同步，渲染线程 `pre_render → pre_draw → post_draw → end_frame` 输出 → 扩展通过 `OpenXRExtensionWrapper` 钩子在实例/会话/帧生命周期内插桩，实现私有能力、额外 composition layer 或异步 Future 处理 (`modules/openxr/openxr_api.cpp:1654`, `modules/openxr/openxr_interface.cpp:1187`, `modules/openxr/extensions/openxr_extension_wrapper.h:70`).
