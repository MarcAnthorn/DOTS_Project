# DOTS-RTS — 大规模 RTS 群体模拟系统

![Unity](https://img.shields.io/badge/Unity-2022.3%2B-black)
![DOTS](https://img.shields.io/badge/Tech-ECS%20%7C%20Jobs%20%7C%20Burst-blue)
![NetCode](https://img.shields.io/badge/Network-NetCode%20for%20Entities-green)

基于 **Unity DOTS**（ECS + Jobs + Burst）的大规模 RTS 群体模拟系统。以 RTS 高密度单位移动作为压力测试场景，自下而上实现从玩法指令到物理求解的完整管线：

```
玩法指令 → 流场寻路 → 预测接触分类 → 软避让 → XPBD 约束求解 → ECS 写回
```

---

## 控制指数（模块总目录）

本表是整份 README 的导航索引：所有模块的职能与业务脚本路径范围一览。

| 编号 | 模块路径 | 职能 | 详细章节 |
|---|---|---|---|
| M1 | `Scripts/Core/Movement/` | 核心群体模拟：流场寻路、增量接触管线、软避让、XPBD 求解、诊断 | [§2](#2-核心群体模拟模块-m1) |
| M2 | `Scripts/Core/Units/Components/`、`Scripts/Core/Units/Authoring/` | 单位组件契约与 Authoring 烘焙 | [§3](#3-单位组件与-authoring-m2) |
| M3 | `Scripts/Core/Units/Systems/`（非 Movement） | 单位初始化、选择、血条等生命周期系统 | [§4](#4-单位辅助系统-m3) |
| M4 | `Scripts/Core/Buildings/` | 建筑组件、RPC 与 Authoring | [§5](#5-建筑模块-m4) |
| M5 | `Scripts/Core/Cameras/` | 主相机组件与初始化 | [§6](#6-相机模块-m5) |
| M6 | `Scripts/Core/Common/` | 通用玩法系统：攻击、伤害、销毁、追踪、技能移动、RPC 生成 | [§7](#7-通用玩法系统-m6) |
| M7 | `Scripts/Core/Replay/` | 事件溯源录制与回放 | [§8](#8-回放模块-m7) |
| M8 | `Scripts/Networking/` | NetCode 客户端/服务端、RPC、连接与入队 | [§9](#9-网络模块-m8) |
| M9 | `Scripts/Input/` | 输入状态机与右键移动指令 | [§10](#10-输入模块-m9) |
| M10 | `Scripts/Framework/` | QFramework 风格 UI/建造/经济框架 | [§11](#11-框架模块-m10) |
| M11 | `Scripts/Entry/` + `Scripts/Utilities/` | 架构入口、服务定位器、调试工具 | [§12](#12-入口脚本与工具-m11) |
| M12 | `.github/` + `Tools/Analysis/` | CI 静态合约与基准分析 | [§13](#13-ci-与工具链-m12) |
| M13 | 根目录本地文档 | 实验计划、交接文档、面试与概念手册（仅本地保存） | [§14](#14-本地文档索引) |

**技术架构与核心机制**：[§15 技术架构](#15-技术架构) ｜ [§16 性能数据](#16-性能数据) ｜ [§17 诊断工具](#17-诊断工具) ｜ [§18 CI 静态合约](#18-ci-静态合约) ｜ [§19 已知限制](#19-已知限制)

> 概念速查：ECS / 流场 / 增量接触 / XPBD / 证书 / 组合根等概念的详解手册与面试问答文档仅保存在本地工作区，未纳入版本控制（见 §14）。

---

## 1. 项目概览

### 核心特性

1. **流场寻路**：确定性 8 邻域 BFS，`Grid / PendingGrid` 双缓冲异步烘焙；单位 O(1) 查询方向；接近目标后切换直接驶入 + 减速 + 到达滞回。
2. **增量预测接触管线**：跨帧持久候选 + 脏体增量修补 + `InteractionCertificate` 证书签发，O(N²) Oracle 验证不漏报。
3. **Timestep Contact Set 缓存**：首个子步完成分类后跨子步复用；包络逃逸时增量修复，脏比例超阈值回退全量重建。
4. **XPBD + 软避让**：Surface Velocity Buffer / RVO 两种软避让模式；XPBD 位置投影（Gauss-Seidel 串行参考 + Jacobi 并行实现）；基于流场格阻挡的软墙/硬墙约束。
5. **事件溯源回放**：只录制输入指令（Command Buffer），零快照，毫秒级状态重置与指令重演（`L` 录制 / `R` 回放）。
6. **混合式网络架构**：Unity NetCode for Entities，服务端权威 + 客户端预测；联网与本地模式共享同一 `BaseFlowMovementSystem` 调度逻辑。

### 顶层目录总览

```text
.
├── Scripts/                   # 全部业务脚本
│   ├── Core/                  # Units / Movement / Buildings / Cameras / Common / Replay
│   ├── Networking/            # NetCode 网络初始化与 RPC
│   ├── Input/                 # 输入状态机与指令
│   ├── Framework/             # QFramework 风格 UI / 建造 / 经济框架
│   ├── Utilities/             # 通用调试工具
│   └── Entry/                 # 架构入口、服务定位器与帧率显示
├── Tools/Analysis/            # 诊断 CSV 趋势分析脚本
├── .github/                   # CI 静态合约脚本与工作流
└── README.md                  # 项目总入口（其余 *.md 仅本地保存）
```

---

## 2. 核心群体模拟模块 M1

路径范围：`Entities/Unit/Systems/FlowField/`

这是整个项目的核心，按职责分为 6 个分区：

### 2.1 组合根与模式入口

| 脚本 | 职能 |
|---|---|
| `BaseFlowMovementSystem.cs` | 唯一组合根：阶段顺序、资源生命周期、JobHandle 依赖、串行/并行路径选择；不实现任何算法 |
| `LocalUnitFlowMovementSystem.cs` | 本地模式入口（要求 `LocalInstance`），继承 `BaseFlowMovementSystem` |
| `NetCodeUnitFlowMovementSystem.cs` | 联网模式入口（要求 `NetworkStreamInGame`），继承同一组合根 |

### 2.2 指令、编队与目标分配

| 脚本 | 职能 |
|---|---|
| `RtsCommandSystem.cs` | 消费 `MoveOrder`：选中快照排序、生成目标槽位、按原队形分配 `UnitMoveDestination`、触发流场重烘焙 |
| `MoveOrderReceiveSystem.cs` | 服务端接收 `RequestMoveOrderRPC`，写入 `MoveOrder` |
| `Utility/MoveDestinationSlotUtility.cs` | 目标点周围可行走槽位生成 + 保队形贪心分配（仅订单时执行） |
| `Utility/FlowFieldUtility.cs` | 世界坐标 ↔ 格子坐标、一维下标、8 方向偏移等纯函数 |

### 2.3 流场烘焙与可视化

| 脚本 | 职能 |
|---|---|
| `FlowFieldBakeSystem.cs` | 双缓冲异步烘焙：Cost 采样、Integration BFS、Vector 生成、原子发布、读者句柄跟踪 |
| `FlowFieldJobs.cs` | `PreparePendingFlowFieldJob` / `GenerateIntegrationFieldJob`（BFS）/ `GenerateVectorFieldJob`（并行方向选择） |
| `GenerateCostFieldJob.cs` | 基于 Unity Physics `CollisionWorld.CalculateDistance` 的障碍 Cost 采样 |
| `FlowFieldDebugSystem.cs` | 把流场编码为纹理的运行时可视化 |

### 2.4 每帧移动 Jobs

路径：`Jobs/`

| 脚本 | 职能 |
|---|---|
| `BuildCrowdMotionIntentJob.cs` | 从 ECS 收集单位事实，生成 body 快照、导航状态与移动意图 |
| `CalculateSoftAvoidanceJob.cs` | 软避让数学：Surface Velocity Buffer / RVO / 墙斥力 / 指数响应 |
| `CalculateArrivalAreaJob.cs` | 到达区域辅助计算 |
| `ApplyFlowMovementJob.cs` | 把求解结果写回 `LocalTransform` / `Velocity` |
| `BuildCrowdBodyResultsJob.cs` | 从子步状态构建最终写回结果（含速度重建语义） |

### 2.5 接触管线（Runtime/ContactPipeline）

路径：`Runtime/ContactPipeline/`，完整分层说明见其下的 `ARCHITECTURE.md`。

| 分区 | 脚本路径范围 | 职能 |
|---|---|---|
| Contracts | `Contracts/Body/`、`Contracts/Certification/`、`Contracts/Execution/`、`Contracts/Interaction/` | 数据契约 ABI：body 产品、`InteractionCertificate`、`ContactPipelineConfiguration`、`ContactConstraint`/`BodyPair`/`PersistentSweptProxy`/`PersistentPredictiveContact`/`StableEntityPairKey` 等 |
| State | `State/Persistent/`（`InteractionCandidateStore.cs`）、`State/Frame/`（5 个 frame 资源所有者） | 跨帧候选仓库与每帧资源生命周期 |
| Kernels | `Kernels/` | 无容器共享纯算法：`PersistentProxyBuilder`、`PersistentContactMath`、`PersistentStoreLookup`、`IncrementalDirtyBodyStore`、`PersistentCacheReusability`、`PredictiveContactScheduler`、`DirtyIncidentPairMapper`、`ActiveConstraintIncidentIndexBuilder`、`ContactPipelineMath`、`ContactPipelineSharedHelpers` |
| Scheduling | `Scheduling/CrowdContactPipelineScheduler.cs`、`Scheduling/Parallel/` | 只做调度接线的组合器；并行 Job 在 `Scheduling/Parallel/Jobs/` |
| Stages/Certification | `Stages/Certification/`（BroadPhase / Persistent / Prediction / Validation） | 扫掠宽相、持久增量分类、包络逃逸验证、证书签发、O(N²) Oracle |
| Stages/Lifecycle | `Stages/Lifecycle/` | 串行/并行管线状态初始化 Job |
| Stages/SoftAvoidance | `Stages/SoftAvoidance/` | 软避让 Job 与并行 workset 构建 |
| Stages/Motion | `Stages/Motion/` | 基准速度、未约束预测、速度重建 |
| Stages/Solver | `Stages/Solver/` | XPBD（GS/Jacobi）、CSR Incident Index、墙壁约束、观测捕获 |
| Observability | `Observability/Contracts/` | 观测数据 ABI，不参与正确性 |

### 2.6 诊断与实验（Diagnostics / Editor）

路径：`Diagnostics/`、`Editor/`

| 分区 | 脚本路径范围 | 职能 |
|---|---|---|
| Capture | `Diagnostics/Capture/` | 快照发布、空间回读、统计发布 Job |
| Presentation | `Diagnostics/Presentation/` | IMGUI 调试面板、世界覆盖层、相机跟随、单位拾取 |
| Recording | `Diagnostics/Recording/` | CSV 录制器、本地录制器 |
| Runtime | `Diagnostics/Runtime/` | 运行时设置、按 World 注册的调试运行时 |
| Experiments | `Diagnostics/Experiments/` | 自适应参数调优器、实验场景、跨帧缓存覆盖 |
| Instrumentation | `Diagnostics/Instrumentation/` | Profiler 时钟 |
| Editor | `Editor/` | 验证入口、Benchmark 窗口、调试器窗口、编译开关设置 |

---

## 3. 单位组件与 Authoring M2

路径：`Entities/Unit/Components/`、`Entities/Unit/Authoring/`

| 脚本 | 职能 |
|---|---|
| `Components/BasicUnitComponents.cs` | 单位核心组件：`BasicUnitTag`、`UnitMoveSpeed`、`UnitSelected`、`Velocity`、`UnitMoveDestination`、`FlowArrivalState`（Ghost 预测）、`UnitContactBody`（XPBD 逆质量）等 |
| `Components/FlowField/GridComponent.cs` | 流场网格、设置、运行时状态、Cost 状态、空间映射等组件契约 |
| `Components/FlowField/ShadowNeighborCacheTypes.cs` | 阴影邻居缓存类型 |
| `Components/AttackAspect.cs` | 攻击 Aspect 组合视图 |
| `Components/UnitRpcs.cs` | 单位相关 RPC 契约 |
| `Authoring/BasicUnitAuthoring.cs` | 单位 Prefab Authoring 烘焙 |
| `Authoring/RtsUnitPrefabsAuthoring.cs` | 单位 Prefab 集合引用 |
| `Authoring/FlowField/FlowFieldManagerAuthoring.cs` | 流场管理器配置烘焙 |

---

## 4. 单位辅助系统 M3

路径：`Entities/Unit/Systems/`（不含 FlowField）

| 子系统 | 脚本路径范围 | 职能 |
|---|---|---|
| 初始化 | `Initialization/InitializeUnitSystem.cs`、`Initialization/UnitCreateInServerSystem.cs` | 单位实体初始化、服务端单位生成 |
| 选择 | `Selection/UnitSelectedSystem.cs`、`Selection/SelectedTagStateSwitchSystem.cs`、`Selection/SelectedTagAuthoring.cs` | 选中状态维护与标签切换 |
| 血条 | `HealthBar/CreateHealBarSystem.cs` | 单位血条实体创建 |

---

## 5. 建筑模块 M4

路径：`Entities/Building/`

| 脚本 | 职能 |
|---|---|
| `BasicBuildingComponents.cs` | 建筑基础组件契约 |
| `BuildingRpcs.cs` | 建筑相关 RPC |
| `Authoring/BuildingAuthoring.cs` | 通用建筑 Authoring |
| `Authoring/BarracksAuthoring.cs` | 兵营建筑 Authoring |
| `Authoring/BarracksPerfabAuthoring.cs` | 兵营 Prefab Authoring |

---

## 6. 相机模块 M5

路径：`Entities/Camera/`

| 脚本 | 职能 |
|---|---|
| `MainCameraComponents.cs` | 主相机组件契约 |
| `InitializeMainCameraSystem.cs` | 主相机初始化系统 |
| `MainCameraAuthoring.cs` | 主相机 Authoring |

---

## 7. 通用玩法系统 M6

路径：`Entities/_Common/`

| 子系统 | 脚本路径范围 | 职能 |
|---|---|---|
| 生成 RPC | `SpawnEntityRpc/ICreateEntityRpc.cs`、`SpawnEntityRpc/CreateBaseUnitRpc.cs` | 服务端生成单位 RPC 抽象与实现 |
| 技能移动 | `Systems/AbilityMove/` | 技能位移组件、Authoring、系统 |
| 攻击 | `Systems/Attack/` | 攻击组件、触发、攻击执行 |
| 碰撞伤害 | `Systems/DamageOnTrigger/` | Trigger 伤害组件、Authoring、系统 |
| 销毁 | `Systems/Destroy/` | 定时销毁、实体销毁、初始化销毁 |
| 生命值 | `Systems/HealPoint/` | 伤害组件、每帧伤害计算、伤害应用、血点 Authoring |
| 追踪 | `Systems/Track/` | 追踪组件、Authoring、触发系统 |
| 辅助 | `ClientHelpSystem.cs`、`Json/JsonConverter.cs` | 客户端辅助与 JSON 转换 |

---

## 8. 回放模块 M7

路径：`Entities/_RePlay/`

| 脚本 | 职能 |
|---|---|
| `Base/LocalInstance.cs` | 本地模式标记组件 |
| `Base/PlayerInputCommand.cs` | 输入指令类型枚举 |
| `NewReplay/ReplaySchema.cs` | `ReplayCommandElement`（指令缓冲）与 `ReplaySystemState`（录制/回放状态） |
| `NewReplay/RequestCommandRpcSystem.cs` | 发送输入 RPC 并顺带录制回放指令 |
| `NewReplay/CommandReplayingSystem.cs` | 回放执行：清场、重置流场、按时间戳用 ECB 重放指令 |
| `NewReplay/ReplayAuthoring.cs` | 回放系统 Authoring |
| `NewReplay/RTSUnitSpawner.cs` | 回放单位生成器 |

---

## 9. 网络模块 M8

路径：`NetWorkInitialize/`

| 分区 | 脚本路径范围 | 职能 |
|---|---|---|
| Client | `Client/ClientConnectManager.cs`、`Client/ClientRequestGameEntrySystem.cs` | 客户端连接管理与入队请求 |
| Common | `Common/TeamType.cs`、`Common/RpcComponents.cs`、`Common/ClientComponents.cs`、`Common/RtsPrefabs.cs` | 队伍类型、RPC 契约、客户端组件、Prefab 集合 |
| Helps | `Helps/LoadConnectionSceneSystem.cs` | 连接场景加载 |
| Server | `Server/ServerProcessGameEntityRequestSystem.cs` | 服务端处理游戏实体请求 |

---

## 10. 输入模块 M9

路径：`_PlayerInput/`

| 脚本 | 职能 |
|---|---|
| `UnitControl/UnitMoveInputSystem.cs` | 右键射线 → 选中单位快照 → `MoveOrder` + RPC 发送 |
| `InputStateSwitchSystem.cs` | 输入状态切换 |
| `_Events/EnterControlStateEvent.cs`、`_Events/EnterBuildingStateEvent.cs` | 控制/建造状态切换事件 |

---

## 11. 框架模块 M10

路径：`_QFrameWork/`

| 分区 | 脚本路径范围 | 职能 |
|---|---|---|
| BuildingManagement | `BuildingManagement/Base/`、`Buildings/`、`Commands/`、`Services/`、`Utils/`、`_Controllers/` | 建造数据模型、建筑实现、建造命令、网格管理、建造工具与服务 |
| UISystem | `UISystem/EcoUI/`、`UISystem/MapUI/`、`UISystem/HpUI/`、`UISystem/Editor/`、`UISystem/BasicBuildUIController.cs`、`UISystem/CameraController.cs`、`UISystem/RTSSelectionManager.cs` | 经济 UI、地图 UI、血条 UI、相机控制、RTS 选择管理 |
| _CommonUtils | `_CommonUtils/CoroutineManager.cs` | 协程管理器 |

---

## 12. 根目录脚本与工具 M11

| 脚本 | 职能 |
|---|---|
| `MainGameArchitecture.cs` | QFramework 风格 `Architecture<MainGameArchitecture>`：注册建造工具与 UI Model |
| `SystemServiceLocator.cs` | ECS System 服务定位器（`ISystemService` + `ServiceSystemBase<T>`） |
| `ServerObjectSystem.cs` | MonoBehaviour 服务定位器（`ServiceObject<T>` + `IService`） |
| `FpsDisplay.cs` | 帧率显示 |
| `Utils/DebugSystem.cs` | 日志系统 |

---

## 13. CI 与工具链 M12

路径：`.github/`

| 文件 | 职能 |
|---|---|
| `scripts/validate_contact_architecture.py` | 层所有权与目录契约静态检查 |
| `scripts/validate_contact_diagnostics.py` | 诊断关闭零额外 container / job / profiler 检查 |
| `scripts/validate_contact_pipeline_audit.py` | 禁止 aggregate bag、调度器不可实现算法 |
| `scripts/validate_contact_static_contracts.py` | 步身份不可从缓存推导、Oracle 不写游戏状态 |
| `workflows/*.yml` | 三条 CI 工作流 |

根目录 `analyze_contact_diagnostic_trends.py` 用于接触诊断 CSV 趋势分析。

---

## 14. 文档索引

| 文档 | 内容 |
|---|---|
| [TECHNICAL_CONCEPTS_GUIDE.md](./TECHNICAL_CONCEPTS_GUIDE.md) | 全部技术概念从零详解（概念 → 为什么 → 项目落地） |
| [INTERVIEW_TECHNICAL_QA.md](./INTERVIEW_TECHNICAL_QA.md) | 求职面试技术问答手册 |
| [Entities/Unit/Systems/FlowField/Runtime/ContactPipeline/ARCHITECTURE.md](./Entities/Unit/Systems/FlowField/Runtime/ContactPipeline/ARCHITECTURE.md) | 接触管线架构与不变式 |
| [Entities/Unit/Systems/FlowField/Runtime/ContactPipeline/DEBT.md](./Entities/Unit/Systems/FlowField/Runtime/ContactPipeline/DEBT.md) | 已知技术债 |
| [Entities/Unit/Systems/FlowField/Diagnostics/README.md](./Entities/Unit/Systems/FlowField/Diagnostics/README.md) | 诊断工具说明 |
| [Entities/Unit/Systems/FlowField/Diagnostics/VERIFICATION_MATRIX.md](./Entities/Unit/Systems/FlowField/Diagnostics/VERIFICATION_MATRIX.md) | 验证矩阵 |
| `DYNAMIC_CONTACT_FRAMEWORK_EXPERIMENT_PLAN.md` | 动态接触框架实验计划 |
| `FAT_AABB_CACHE_TASK_HANDOFF.md` | Fat AABB 缓存任务交接 |
| `INCREMENTAL_PREDICTIVE_CONTACT_PIPELINE.md` | 增量预测接触管线设计 |
| `INCREMENTAL_PREDICTIVE_CONTACT_BENCHMARK.md` | 增量接触基准方法 |
| `SIMULATION_DEBUGGER_USAGE.md` | 模拟调试器使用说明 |
| `SIMULATION_DEBUGGER_REWORK_PLAN.md` | 模拟调试器重构计划 |

---

## 15. 技术架构

### 数据流

```
玩法指令（UnitMoveInputSystem）
        │  右键 → MoveOrder + 选中快照
        ▼
RtsCommandSystem — 编队槽位分配 → UnitMoveDestination
        │
        ▼
FlowFieldBakeSystem — Cost → Integration → Vector（双缓冲异步烘焙）
        │
        ▼
BaseFlowMovementSystem（组合根，每帧）
  ├─ [初始化]    BuildCrowdMotionIntentJob
  ├─ [认证]      InteractionCertificationJob
  │               ↳ 持久拓扑验证 / 增量修补 / 全量回退
  │               ↳ InteractionCertificate 签发，下游 fail-closed
  └─ for each substep
       ├─ [SoftAvoidance]  RVO / Surface Velocity Buffer
       ├─ [Motion]         速度整合 → 预测位置
       └─ for each iteration
            ├─ [Wall]      墙壁约束投影
            └─ [Contact]   XPBD pair 评估 → body gather（GS / Jacobi）
  └─ ApplyFlowMovementJob — Transform + Velocity 写回 ECS
```

### 架构约束（CI 强制执行）

- `BaseFlowMovementSystem` 是唯一组合根，不实现分类、证书、求解器或诊断算法；
- 持久候选状态仅 Certifier 可写；SoftAvoidance / Motion / Solver 只消费已签发的认证视图；
- `RTS_CONTACT_DIAGNOSTICS` 关闭时零额外 NativeContainer、零额外 Job、零 Profiler 读取；
- Oracle 可观测但不可改变候选状态或签发证书。

---

## 16. 性能数据

**1k 单位 — Substep 缓存关闭 / 开启对比：**

> 在 1,000 单位开阔往返的高密度持续接触场景中（4 Substeps × 4 Iterations，跨帧缓存关闭），每种配置独立采样 2 次、每次 1,200 帧。

| 指标（每帧均值） | 缓存关闭 | 缓存开启 | 改善 |
|---|---:|---:|---:|
| 整体求解管线 | 43.35 ms | 16.19 ms | **-62.7%（2.68×）** |
| 接触 Pair 管线 | 35.37 ms | 9.59 ms | **-72.9%（3.69×）** |
| 全量候选扫描 | 24.94 ms | 6.98 ms | **-72.0%（3.57×）** |
| XPBD 迭代投影 | 3.94 ms | 3.15 ms | **-19.9%（1.25×）** |

性能提升并非通过减少接触约束换取：平均接触 Pair 从 **596** 增至 **721**，平均活跃 Pair 从 **543** 增至 **672**。

**5k 单位同屏运行：** 演示动画仅保存在本地，不纳入版本控制。

---

## 17. 诊断工具

| 工具 | 入口 | 功能 |
|------|------|------|
| **SimulationDebuggerPanel** | `F8` 开关 | IMGUI 面板：概况 / 接触分类 / 增量统计 / 运行时参数覆盖 |
| **CSV 录制** | `F6` 开始/停止，`F7` 重置录制 | 按帧输出接触统计 CSV |
| **Validate Incremental Predictive Contact Pipeline** | `RTS/Diagnostics/Validate…` | 增量管线合规验证 |
| **Incremental Contact Benchmark Tuner** | `RTS/Diagnostics/Incremental Contact Benchmark Tuner` | 自动参数搜索 + CSV 对比 |
| **Incremental Contact Pipeline** | `RTS/Diagnostics/Incremental Contact Pipeline` | 增量管线指标实时监控 |
| **Select Build Settings** | `RTS/Diagnostics/Select Build Settings` | 切换 `RTS_CONTACT_DIAGNOSTICS` 编译开关 |
| **Local Gameplay Mode Validation** | `RTS/Validation/Local Gameplay Mode` | 本地模式功能验证 |

`RTS_CONTACT_DIAGNOSTICS` 关闭时所有诊断路径由预处理器移除；仿真正确性不受影响。

---

## 18. CI 静态合约

`.github/workflows/` 三条工作流在每次 push 时执行四份静态检查：

| 脚本 | 检查内容 |
|------|---------|
| `validate_contact_architecture.py` | 层所有权；禁止历史 `Jobs/ContactPipeline` 布局回归 |
| `validate_contact_diagnostics.py` | 诊断关闭合约；零额外 container / job / profiler |
| `validate_contact_pipeline_audit.py` | 禁止 aggregate bag；调度器不可实现算法 |
| `validate_contact_static_contracts.py` | 调度步骤身份不可从缓存 generation 派生；Oracle 不写游戏状态 |

CI 不替代 Unity Editor 编译、Burst 编译、Collections Safety 和运行时性能验证。

---

## 19. 已知限制

- Contact Island 休眠未实现——持续活跃接触始终参与求解；
- Gauss-Seidel 无并行路径（需图着色或冲突无关批次）；
- 持久 Spatial Membership 依赖容量上限，超限回退全量扫描；
- 流场为全局烘焙，未支持动态障碍局部增量更新。
