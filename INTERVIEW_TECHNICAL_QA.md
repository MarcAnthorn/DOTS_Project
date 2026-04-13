# DOTS-RTS 面试技术问答手册

> 用途：求职面试中介绍本项目的速查手册。所有答案都基于仓库真实代码，标注了对应文件与关键常量。
> 使用建议：先记住「项目一句话介绍」和「核心技术点清单」，再按章节逐条准备；每条答案后面都附了「追问」，提前准备能让回答有纵深。

---

## 0. 项目一句话介绍与亮点清单

**一句话介绍（30 秒版）：**

> 这是一个基于 Unity DOTS（ECS + Jobs + Burst）的大规模 RTS 群体模拟系统，支持 5000 单位同屏。核心是一条自下而上的完整管线：玩法指令 → 流场寻路 → 增量预测接触分类 → 软避让 → XPBD 约束求解 → ECS 写回。项目最大的技术亮点是在接触检测上实现了「跨帧持久候选 + 脏体增量修补 + 证书签发」机制，并用 O(N²) Oracle 持续验证不漏报。

**核心技术点清单（面试官通常从这里面挑）：**

1. 确定性 8 邻域 BFS 流场寻路，三层 Cost / Integration / Vector 烘焙，`Grid / PendingGrid` 双缓冲异步发布。
2. 增量预测接触管线：`StableEntityPairKey` 驱动的接触生命周期（Dormant / Approaching / Predictive / Actual / Expired），`PersistentSweptProxy` Guard Bounds 证明拓扑完备性。
3. 每帧只修补「脏 body」的邻居对；脏比例超过 70% 才全量重建；`NativeHashMap` 替代排序 + 二分查找，消除热路径。
4. `InteractionCertificate` 证书机制：所有下游（SoftAvoidance / Motion / Solver）只能消费被证书保护的紧凑视图，证书作用域失效时 fail-closed。
5. Timestep Contact Set 缓存：首个子步完成分类后整个 timestep 复用，包络逃逸时增量修复；跨帧持久缓存进一步保留 Dormant 接触的唤醒时序。
6. XPBD 位置投影（含 Compliance 柔度），Gauss-Seidel 串行参考实现 + 无原子操作的确定性 Jacobi 并行实现（CSR Incident Index）。
7. 软避让双模式：Surface Velocity Buffer 与类 RVO 速度障碍。
8. 网络：Unity NetCode for Entities，服务端权威 + 客户端预测，本地与联网模式共享同一个 `BaseFlowMovementSystem` 组合根。
9. 事件溯源回放：只录制输入指令（Command Buffer），零快照，毫秒级重置重演。
10. 工程化：CI 四条 Python 静态合约脚本 + `RTS_CONTACT_DIAGNOSTICS` 编译开关，诊断关闭时零额外 NativeContainer / Job / Profiler。

**性能背书（README 实测数据，回答性能题时引用）：**

1,000 单位开阔往返高密度持续接触场景，4 Substeps × 4 Iterations，仅开启 Substep 接触集缓存（跨帧缓存关闭）：

| 指标（每帧均值） | 缓存关闭 | 缓存开启 | 改善 |
|---|---:|---:|---:|
| 整体求解管线 | 43.35 ms | 16.19 ms | -62.7%（2.68×） |
| 接触 Pair 管线 | 35.37 ms | 9.59 ms | -72.9%（3.69×） |
| 全量候选扫描 | 24.94 ms | 6.98 ms | -72.0%（3.57×） |
| XPBD 迭代投影 | 3.94 ms | 3.15 ms | -19.9%（1.25×） |

关键点：提速不是靠减少约束——平均接触 Pair 从 596 增至 721，活跃 Pair 从 543 增至 672，处理了更多持续接触反而总耗时降低 62.7%。

---

## 1. 项目总览与架构

### Q1.1 这个项目整体架构是怎样的？

**回答：**

整个项目是一个 ECS 世界里的「命令 → 寻路 → 接触 → 求解 → 写回」流水线，组合根是 `BaseFlowMovementSystem`（`Entities/Unit/Systems/FlowField/BaseFlowMovementSystem.cs`）。它只负责三件事：

1. 阶段顺序：每个子步固定走 `SoftAvoidance → Motion(PredictUnconstrained) → 多次 Wall + Contact 迭代 → ReconstructVelocity`；
2. 资源生命周期：每帧创建 frame 资源（body 数组、certification / soft / solver 资源），并在 JobHandle 依赖链末尾统一 Dispose；
3. JobHandle 依赖接线：所有阶段 Job 由它调度，并把整个求解句柄注册给 `FlowFieldBakeSystem` 作为 ActiveGrid 读者，保证双缓冲交换安全。

接触管线本身按「契约 → 状态 → 内核 → 阶段 → 调度」分层（`Runtime/ContactPipeline/`）：

- `Contracts/`：只放数据产品 ABI，比如 `CrowdBodySnapshot`、`ContactConstraint`、`InteractionCertificate`；
- `State/Persistent/`：`InteractionCandidateStore` 持有所有跨帧候选容器（代理、邻居对、接触、调度表）；
- `Kernels/`：无容器共享的纯 Burst 算法（`PersistentProxyBuilder`、`PersistentContactMath`、`DirtyIncidentPairMapper` 等）；
- `Stages/`：Certification / SoftAvoidance / Motion / Solver 四个阶段，每个阶段是独立的 `IJob`；
- `Scheduling/`：`CrowdContactPipelineScheduler` 只做调度组合，不实现算法；可执行并行 Job 全部放在 `Scheduling/Parallel/Jobs/`。

**追问（常问）：为什么要把调度器和算法分开？**

因为 Unity Collections Safety 校验的是「Job 结构体上直接声明的 NativeContainer 字段」。把能力全部塞进一个 aggregate 调度 Job 会隐藏真实容器边界；拆成四个独立 `IJob`（`InteractionCertificationJob`、`SoftAvoidanceJob`、`MotionIntegrationJob`、`ConstraintSolverJob`），每个 Job 只带自己需要的容器，Collections Safety 能真正检查到每个阶段的能力。CI 脚本也明确禁止 aggregate composition job 回归。

---

### Q1.2 为什么用 ECS + Jobs + Burst，而不是传统 MonoBehaviour 或者直接上 Unity Physics？

**回答：**

三个原因：

1. **规模**：目标是 5,000 单位同屏。传统 OOP 的每单位组件查找、虚函数、GC 分配在万级实体下不可控；ECS 把数据按结构连续排布（SoA），`IJobEntity` / `IJobParallelFor` 直接并行遍历，配合 Burst（LLVM 编译、无托管引用）才能达到目标帧率。
2. **确定性**：RTS 需要可复现。纯函数式 Burst 内核 + 显式排序 + 确定性 fallback 法线，配合网络预测/回放，比传统脚本更可控。
3. **物理不是重点**：单位是圆盘（disc），接触只需要「swept disc + XPBD 投影」，不需要 Unity Physics 的通用刚体、碰撞材质、多体岛。README 明确写「Unity Physics 中的单位保持 Kinematic，XPBD 接触只读取 `UnitContactBody.InverseMass`」。所以自研一个数据导向的定制管线比引入完整物理引擎更贴合需求、也更容易做增量优化。

---

### Q1.3 为什么接触检测不做成每帧全量 O(N²)，而要做增量？

**回答：**

RTS 群体场景有一个强先验：**相邻帧的接触拓扑高度相干**。上一帧 600 个接触对，下一帧绝大多数还是同一对实体在接触；真正变化的是少量「跑进/跑出彼此影响范围」的 body。全量每帧重扫一遍 BroadPhase + 分类，成本是 O(N) 或 O(N²) 起步；增量思路是把跨帧候选拓扑持久化，每帧只对「脏 body」的邻居做修补，因此成本正比于变化量而不是总量。这也是基准里「全量候选扫描 24.94ms → 6.98ms」的来源。

---

## 2. 流场寻路

### Q2.1 为什么用流场寻路而不是给每个单位跑 A*？

**回答：**

A* 是逐单位查询，5,000 个单位同时下达移动指令就是 5,000 次独立搜索，且结果互相独立、重复劳动大。流场寻路把搜索从「逐单位」变成「逐目标」：同一批单位共用一个目标点，只在目标变化时烘焙一次网格，之后每个单位对网格做 O(1) 查询——读当前格子的方向索引，乘以速度就是移动意图。README 里写得很明确：「以共享网格流场替代逐单位 A* 查询」。

代价是流场不擅长多目标/动态障碍，这也是项目已知限制之一（动态障碍只支持全局重烘焙，没有局部增量更新）。

---

### Q2.2 流场的三层字段和烘焙流程是怎样的？

**回答：**

每个 `FlowFieldCell`（`Entities/Unit/Components/FlowField/GridComponent.cs`）只有三个字段：

```csharp
public byte Cost;            // 0 = 障碍，1 = 可行走
public ushort IntegrationValue; // BFS 距离，ushort.MaxValue = 不可达
public byte BestDirectionIndex; // 0-7 的 8 方向，0xFF = 无效
```

烘焙由 `FlowFieldBakeSystem` 分三阶段（`FlowFieldBakeSystem.cs` + `FlowFieldJobs.cs`）：

1. **Cost Field**（仅障碍变化时重跑）：`GenerateCostFieldJob` 并行采样 Unity Physics `CollisionWorld` 的 `CalculateDistance`，网格中心点与障碍物距离小于 `CellRadius` 就标为 Cost=0。用的是自定义 `CollisionFilter`（`CollidesWith = 1 << 2`），只与指定层碰撞。
2. **Integration Field**：`GenerateIntegrationFieldJob` 用 `NativeQueue<int2>` 做单源 8 邻域 BFS，从目标格向外扩散，每格 IntegrationValue = 到目标的 BFS 步数；目标格本身是障碍则直接放弃。
3. **Vector Field**：`GenerateVectorFieldJob` 是 `IJobParallelFor`，每格独立比较 8 个邻居的 IntegrationValue，选最小者的方向写入 `BestDirectionIndex`。

Cost 与 Integration/Vector 的失效粒度是分开的：`FlowFieldCostState.IsDirty` 只在障碍布局变化时置位，改目标点只重跑 Integration + Vector，Cost 直接复用。

---

### Q2.3 双缓冲异步烘焙是怎么保证单位读不到半成品网格的？

**回答：**

这是这个项目我比较满意的一个设计点：

1. `FlowFieldGrid` 持有两块 `NativeArray<FlowFieldCell>`：`Grid`（已发布快照）和 `PendingGrid`（正在烘焙）。
2. 新目标到达时，`PreparePendingFlowFieldJob` 把 ActiveGrid 的 Cost 复制进 PendingGrid，同时把 Integration/Vector 重置为无效值；随后在 PendingGrid 上做 Integration BFS + Vector 并行计算，全程不碰 ActiveGrid。
3. 烘焙完成时做一次缓冲交换（`(grid.Grid, grid.PendingGrid) = (grid.PendingGrid, grid.Grid)`），原子地让新快照生效。
4. 交换后的旧 ActiveGrid 变成 PendingGrid，会被下一次烘焙写入。`FlowFieldBakeSystem.RegisterActiveGridReader()` 收集所有读取 ActiveGrid 的 JobHandle（移动管线每帧都会注册求解句柄），写入前必须等这些旧读者完成，避免 write-after-read 数据竞争。
5. 请求版本守卫：`RecalculateFlowFieldTag.RequestVersion` 在烘焙期间再次变化时，旧结果不发布，直接复用 PendingGrid 重启最新版的烘焙。

单位侧通过 `FlowFieldRuntimeState.ActiveVersion` 判断网格是否已发布；`BuildCrowdMotionIntentJob` 始终读 `FlowFieldRuntimeState.ActiveRequestVersion` 对应的稳定快照。

---

### Q2.4 单位是怎么消费流场的？到达终点怎么不抖动？

**回答：**

`BuildCrowdMotionIntentJob`（`Entities/Unit/Systems/FlowField/Jobs/BuildCrowdMotionIntentJob.cs`）每帧从 ECS 收集单位事实（位置、速度、半径、目标），然后：

1. 查询当前格子的 `BestDirectionIndex`，`FlowFieldUtils.GetDirectionOffset` 把方向索引变成 8 方向偏移，乘以 `UnitMoveSpeed` 得到首选速度；
2. 当 `IntegrationValue <= DirectApproachIntegrationDistance` 时切换到「直接驶入」：不再跟流场格子方向，而是朝目标槽位直线走，并按 `distance / brakingDistance` 线性减速；
3. 到达判定带滞回：`FlowArrivalState.IsSettled` 跨帧保留，进入时用较小的 `ArrivalRadius`，已 settled 后要用更大的 exit radius 才会重新启动，避免单位在到达区边界一帧走一帧停地抖动。

槽位（`UnitMoveDestination`）由 `RtsCommandSystem` 在订单到来时分配：在目标点周围生成固定间距的可行走槽位环，再按单位相对质心的原有队形贪心匹配，保证编队整体平移而不是乱成一团。槽位分配只在订单时刻执行，不进逐帧热路径。

---

## 3. 增量预测接触管线（核心中的核心）

### Q3.1 讲一下增量预测接触管线的整体思路

**回答：**

一句话：**把「接触检测」从每帧重算改成「跨帧维护一份候选拓扑，只修补变化的部分」，并且用证书保证修补结果的正确性**。

具体分为四层：

1. **持久候选层**（`InteractionCandidateStore`）：持有 `PersistentSweptProxies`（每个 body 的扫掠包络）、`PersistentNeighborPairs`（可能的接触对）、`PersistentPredictiveContacts`（每对的生命周期状态）、`PersistentActiveContactKeys` / `SoftAvoidancePairKeys` / `DormantContactSchedule`（下游视图），以及两个加速索引（`PredictiveContactIndex` O(1) 哈希查找、`IncidentPairLookup` 按实体查邻居）。
2. **认证层**（`InteractionCertificationJob`）：每帧先验证 proxy 有效性并标记脏体；然后选择一条路径——直接复用、增量修补、或全量重建；最后把紧凑视图（SoftAvoidancePairs / TimestepContactPairs / PredictiveContactSchedule）提交并签发 `InteractionCertificate`。
3. **消费层**：SoftAvoidance / Motion / Solver 只读证书保护下的紧凑视图，永远不接触持久候选容器。
4. **验证层**：诊断模式下跑 O(N²) Oracle 与增量结果逐对对比，漏报计数必须为零。

---

### Q3.2 PersistentSweptProxy 的 Guard Bounds 是怎么证明拓扑完备性的？

**回答：**

`PersistentSweptProxy`（`Contracts/Interaction/PersistentSweptProxy.cs`）每个 body 一份，携带两套边界：

- `TightMin/TightMax`：最近一次分类时的「交互包络」，精确覆盖 XPBD swept contact + Soft Avoidance shell + RVO 视域（`ContactPipelineMath.CalculateInteractionBounds` 计算的路径起点/终点 ∪ 未约束位置/求解位置 ∪ RVO 视域终点，再外扩半径 + 预测 skin + 余量）；
- `GuardMin/GuardMax`：Tight 边界再外扩 `GuardEnvelopeMargin`，用于证明「我这一帧所有可能的交互对象都没有漏掉」。

**完备性论证**：如果每个 body 的 Guard 边界都覆盖了自己真实运动范围，那么「任何两个 guard 相交的 body 对」必然包含所有潜在接触对；反之不相交的对在本帧不可能产生接触。全量重建时（`FullRebuildPersistentNeighborTopology`）正是把所有 Guard 相交的对写入 `PersistentNeighborPairs`——这一步做完，候选集就是完备的。之后每帧只要验证旧 guard 还能包住当前轨迹（`AabbContains` 检查），就说明上一帧的候选集对这个 body 仍然完备；只有包不住（逃逸）才需要重查它的邻居。

这里还有一个很关键的工程细节：代理里区分了 **TopologyGuard（位置圆守护）** 和 **Guard（包络守护）**。转向/加速只改变 InteractionEnvelope 而不改变物理位置，所以只用位置圆判断拓扑 dirty，避免 AI 导航下「每帧转向都触发拓扑重建」的 dirty 率雪崩（代码注释里明确写了这是此前用 InteractionEnvelope 做判据时的核心缺陷）。

---

### Q3.3 每帧的脏体判定和增量修补流程是什么？

**回答：**

每帧开始，`PersistentProxyBuilder.ClassifyAndUpdateForBody` 对每个 body 做三类比对，产出 `IncrementalBodyDirtyFlags`：

- `EntitySet`：body 集合变化（新增/删除/槽位替换），不可修补，强制全量重建；
- `Topology`：物理位置逃出上一帧的 TopologyGuard 位置圆，或半径/有效性变化——需要重查邻居对；
- `Motion`：轨迹、RVO 视域、半径变化但仍在 topology guard 内——只需要更新 proxy，不需要动拓扑。

然后 `BuildContactPairsFromPersistentNeighborSet`（`Stages/Certification/Persistent/IncrementalPredictiveContactPipeline.cs`）做决策：

```csharp
float dirtyRatio = Bodies.Length > 0 ? (float)topologyDirtyCount / Bodies.Length : 1f;
bool useFullRebuild = !cacheCanBePatched || entitySetDirty ||
                      dirtyRatio > IncrementalDirtyBodyRatioThreshold; // 0.7f
```

- 脏比例 ≤ 70%：`IncrementallyRepairPersistentNeighborTopology`——先保留两端都不脏的旧邻居对，再用空间哈希（`PersistentSpatialMembership`）只对脏 body 查邻居，补进新对，排序去重后替换；
- 脏比例 > 70%：全量重建（`FullRebuildPersistentNeighborTopology`），清空并重新做 uniform grid broadphase。

**为什么阈值定 70% 而不是 README 里写的 35%？** 这是代码演进后的实际值：注释里说明了理由——局部密集接触簇（比如两支编队对撞）可能在一个区域内弄脏很多 body，但增量修补是「空间哈希限定到脏列表」的，即使 50%-60% 的局部脏比例仍然比全局 O(N) 重建便宜；只有接近全体的脏才值得全量重建。**面试时主动指出「README 还写着 35%，但代码里实际阈值已经是 0.7f」会是很加分的诚实细节。**

---

### Q3.4 增量修补时怎么快速找到脏 body 的邻居对？

**回答：**

两个数据结构配合：

1. `PersistentIncidentPairLookup`：`NativeParallelMultiHashMap<Entity, int>`，每个实体映射到它参与的所有 `PersistentNeighborPairs` 的下标，O(1) 枚举脏体的邻居；`IncidentLookupEpoch` 与 `TopologyEpoch` 绑定，epoch 不匹配就说明索引过期，回退全量扫描。
2. `DirtyIncidentPairMapper` 的 eligibility filter：不是所有邻居对都要重分类——只有「上次分类非 Expired」或者「两端 tight 扫掠 AABB 本帧仍重叠」的对才输出。远处已 Expired 的对被跳过，这正是省时间的核心：持久邻居池是 Guard 扩大的集合，大部分是远处 Dormant/Expired 对。

还有一个自适应退化：当 dirty 体通过索引枚举出的邻居访问量超过持久池一半（`LinearScanIncidentVisitRatio = 0.5f`）时，索引+去重的成本反而不如单趟线性扫描，于是 `TryMap` 自动切到对持久邻居池的一趟扫描（每对只做一次 dirty 端点 + eligibility 测试，无去重）。这是典型的「在正确性不变的前提下按成本选算法路径」。

---

### Q3.5 O(1) PersistentContactIndex 替代了什么？

**回答：**

旧实现是「有序列表 + 二分查找」定位持久接触。新实现维护 `NativeHashMap<StableEntityPairKey, PersistentPredictiveContact>`：

- 全量重建后从 `PersistentPredictiveContacts` 列表整体回填（O(N)，无排序）；
- 增量 patch 路径直接写入/删除单条，无需重排序；
- 查询路径从 O(log N) 变成 O(1)。

`StableEntityPairKey` 用两个 Entity 的 (Index, Version) 做稳定排序键（小者在前），保证跨帧、跨列表、跨哈希表身份一致，且不受帧内 body 数组排序顺序影响。这是热路径（每帧对每个候选对做生命周期查询）的关键优化之一。

---

### Q3.6 接触生命周期状态机和 Dormant 调度是怎么工作的？

**回答：**

生命周期枚举（`Contracts/Interaction/PersistentPredictiveContact.cs`）：

```
Dormant → Approaching → Predictive → Actual → Expired
```

分类依据是相对运动的几何量（`ClassifyPersistentNeighborPair`）：

- **Expired**：相对轨迹最近距离超过保留余量 `candidateDistance + TimestepContactMargin * 2`，或起点已分离且未开启预测对生成（`EnablePredictivePairGeneration`）；
- **Actual**：起点已重叠（`startDistanceSq <= radiusSum²`）；
- **Dormant**：起点分离、最近距离仍大于 `radiusSum + PredictiveSkin`，但没超过保留余量——未来可能接触，先休眠；
- **Approaching**：起点分离、最近距离 ≤ `radiusSum + PredictiveSkin`，且不会发生"起点分离、终点分离、中途穿过"的擦肩而过；
- **Predictive**：起终点都分离但相对路径穿过接触半径（会擦肩而过），用初始分离平面做稳定法线约束，防止换侧穿模。

Dormant 对不进入活跃 XPBD 视图，而是写进 `PredictiveContactSchedule`：根据相对轨迹的最近接近时间算出「最早可能接触的子步」，`PredictiveContactScheduleEntry.Substep` 表示在第几个 substep 唤醒。每个子步开始，`ActivateScheduledPredictiveContactsForSubstep` 扫描调度表，到了唤醒子步就重新做一次精确判断：确实会接触 → 加进 `TimestepContactPairs` 激活；还没到 → 顺延到下一子步；不会接触 → 标记 Expired。调度表按子步排序 + 游标推进，避免每子步全表扫描。

这里有一个细节：`ushort.MaxValue` 是「本 timestep 不唤醒」的合法哨兵（无相对运动），证书校验专门放行它，不能当越界处理。

---

### Q3.7 Timestep Contact Set 缓存和 Persistent Contact Cache 有什么区别？

**回答：**

这是两层缓存：

1. **Timestep Contact Set 缓存**（`EnableTimestepContactSetCache`）：一个 timestep 内有多个 substep。关闭时每个 substep 都重跑 Swept Disc BroadPhase + 分类；开启后只有首个子步做完整分类，把 `TimestepContactPairs` 视图缓存，后续子步直接复用。单位轨迹偏出 Guard Envelope 时，Certifier 对受影响 body 做增量修复，只重分类脏 body 的邻居对。
2. **Persistent Contact Cache**（`EnablePersistentContactCache`）：跨 timestep 保留接触生命周期和激活时序。Dormant 接触在预计到达的子步直接按调度激活，不用每帧从零分类。它要求 Timestep Contact Set 缓存同时开启才生效（`effectivePersistentContactCache = requested && effectiveTimestepContactSetCache`）。

基准里 62.7% 的提速主要来自第一层；README 也注明测试时「跨帧缓存关闭」，说明单独第一层就能带来这个收益。

---

### Q3.8 包络逃逸是怎么检测和修复的？

**回答：**

证书的证明范围是「已认证的 InteractionEnvelope」。任何可能让单位跑出这个范围的变更来源都要被验证（`Stages/Certification/Prediction/ContactEnvelopeGuard.cs`）：

1. **BaseMotion 逃逸**：软避让之前，验证基速度预测位置仍在包络内；
2. **Soft 输出逃逸**：`ClampSoftOutputToInteractionEnvelope` 先模拟「base + 软避让响应」后的终点，如果越界就对软避让向量做二分搜索（0~1 缩放、8 次迭代）找最大安全缩放系数；zero 都不行就归零；
3. **Predicted 逃逸**：未约束预测积分后的位置越出 ContactEnvelope；
4. **SolverCorrection 逃逸**：墙/接触投影后的位置越出包络（墙后、接触后各查一次）。

每次逃逸都：撤销证书（`RevokeInteractionCertificate`，记录 violation 证据）→ 把 body 标脏（`Motion | CorrectedEscape`）→ 走到 `RepairOrRebuildContactViewForRemainingTime`：优先 `TryIncrementallyRepairEscapedContactSet`（只重分类脏 body 的 incident 对、保留未受影响对的历史激活状态）；脏比例超阈值或修复失败时 `BuildOrRefreshTimestepContactViews` 回退全量重建。修复完成后重新签发覆盖剩余子步的证书。

---

### Q3.9 InteractionCertificate 是什么？下游怎么保证不消费脏数据？

**回答：**

证书（`Contracts/Certification/InteractionCertificationContracts.cs`）是「紧凑视图的作用域描述 + 完整性证据」，字段包括：

- 身份：`WorldId`、`SimulationStepId`（由组合根分配，绝不由缓存代年龄推导）；
- 指纹：`BodySetFingerprint`（遍历实体 Index/Version 的哈希）、`ConfigurationFingerprint`（所有求解参数哈希）、`TopologyEpoch`、`ClassificationFingerprint`；
- 范围：`StartSubstep / EndSubstepExclusive / HorizonDuration`；
- 结果：`SourceMode`（FullSweep / PersistentReuse / PersistentRepair / PersistentFullRebuild）、各视图计数、`Flags`（六个验证位 + `Issued`）。

签发条件是六项证据全部通过：结构、实体映射、配置指纹、拓扑覆盖、分类、消费者视图已提交。

消费端是 fail-closed：`ValidateConsumerViewsSerial / P1P6` 在每个 Soft 阶段前和 Solver 阶段前检查 `GetConsumerCertificateFailure`——证书未签发、作用域不匹配、指纹不一致、视图计数对不上，都会映射成明确的 `ContactSolverSkipReason`（有 20 多种枚举值），并撤销证书、置 `RecoveryRequired`，而不是带病继续求解。整个设计保证「持久候选状态只有 Certifier 能写；下游要么拿到被证明的视图，要么整个子步被跳过并进入恢复路径」。

---

### Q3.10 Oracle 是什么？怎么证明增量不漏报？

**回答：**

`IncrementalContactOracle`（`Stages/Certification/Validation/IncrementalContactOracle.cs`）是**只读的 O(N²) 参考实现**：对每对 body 独立做精确的 swept-disc 最近距离测试（用相对位移做点积投影求最近时间，比较 `radiusSum + PredictiveSkin`），完全不依赖增量拓扑，生成 ground truth 接触对集合。然后与 `TimestepContactPairs` 逐对对比，统计：

- `OracleMissingPairCount`（漏报，增量漏掉的真实接触）；
- `OracleExtraPairCount`（多报，增量多出来的假接触）。

漏报数大于 0 时置 `OracleMismatch = 1`，在诊断模式下必须为零。Oracle 只做观测：它不写任何缓存状态、不参与修复决策、不能影响证书，这也被 CI 静态脚本强制检查。它只在 `RTS_CONTACT_DIAGNOSTICS` 宏下编译，正式构建零成本。

---

## 4. 求解器：软避让、XPBD、墙

### Q4.1 XPBD 是怎么实现的？公式是什么？

**回答：**

`XpbdContactSolver.cs` 实现的是位置投影形式的 XPBD（Extended Position Based Dynamics）。每个 substep 开始时 lambda 清零，每次迭代：

```
alpha = Compliance / (substepDeltaTime^2)          // 柔度转刚度倒数
denominator = invMassA + invMassB + alpha
deltaLambda = -(constraintValue + alpha * lambda) / denominator
lambda = max(0, lambda + deltaLambda)              // 单向接触，只推不拉
correctionA = normal * invMassA * appliedLambda
correctionB = -normal * invMassB * appliedLambda
```

Regular 模式用当前圆心距做约束值 `C = distance - radiusSum`，法线取实时相对位置；Predictive 模式用分类时记录的稳定法线，约束值改为 `dot(delta, normal) - radiusSum`，避免高速擦肩而过时法线翻转导致换侧穿模。

`Compliance` 提供柔度控制；`InverseMass` 来自 `UnitContactBody`（单位在 Unity Physics 里保持 Kinematic，接触质量完全由这个自定义组件控制）。

---

### Q4.2 Gauss-Seidel 和 Jacobi 两种求解模式怎么选？

**回答：**

- **Gauss-Seidel（串行参考）**：逐 pair 立即写回位置，收敛快，但天然的串行依赖，无法安全并行（相邻约束共享 body）。
- **Jacobi（并行）**：一次迭代分成两段——`EvaluateParallelJacobiPairsJob` 从同一份位置快照并行评估所有 pair，把每个 pair 的修正写进 `JacobiPairCorrections`（不动位置）；`GatherAndApplyParallelJacobiBodiesJob` 再按 body 并行收集该 body 的所有 incident 修正，取平均后一次性应用。

关键点：

1. **无浮点原子操作**：修正通过帧局部的 CSR Incident Index（`ActiveIncidentOffsets` 前缀和 + `ActiveIncidentPairIndices`）做确定性收集，每个 body 的求和顺序固定，结果可复现；
2. **不收敛无界**：Jacobi 平均化会慢一点，但换来了可并行性和确定性，README 已知限制里也如实写了「Gauss-Seidel 无并行路径（需图着色或冲突无关批次）」；
3. 两条路径共享同一个 `CommitTimestepContactViews` 提交边界和 `ValidateConsumerViews*` 闸门，保证求解模式切换不影响证书语义。

---

### Q4.3 软避让的两种模式分别怎么算？

**回答：**

入口是 `SoftAvoidanceMath.TryCalculatePairVelocities`（`Jobs/CalculateSoftAvoidanceJob.cs`）：

1. **Surface Velocity Buffer（默认）**：按「shell 内线性衰减」的斥力速度——`softFactor = saturate((softShell - surfaceGap) / softShell)`，单位速度乘 softFactor 乘 moveSpeed 沿对方反方向推；一个 body 收到多个邻居的贡献后取平均（`/= SoftAvoidanceNeighborCount`）。
2. **ReciprocalVelocityObstacle（类 RVO）**：用相对位置和相对速度算时间视域 `timeHorizon` 内的最近接近点；如果最近距离小于安全距离，就用 `(safetyDistance - closestDistance) / correctionTime` 生成相对修正，再按逆质量比分配给自己和对面的 body。代码注释明确说明这只是兼容枚举名，不是 ORCA/RVO2 的线性规划实现——面试时主动澄清这一点能体现严谨。

速度缓冲还有个工程细节：响应率通过 `alpha = 1 - exp(-responseRate * dt)` 转成与子步数无关的指数响应，避免固定帧率假设；settled 单位响应率乘 `SettledSoftAvoidanceMultiplier`，并对速度额外乘 `0.8^(dt*60)` 做阻尼，让到达后的单位安静下来。

---

### Q4.4 墙壁约束是怎么做的？

**回答：**

两层：

1. **软墙避让**（速度层）：`AccumulateWallAvoidanceVelocity` 检查 body 周围 3×3 格，对 Cost==0 的障碍格中心计算 `(wallCheckRadius - distance) / distance * 10` 的斥力速度，叠加进 `SoftAvoidanceVelocity`；
2. **硬墙投影**（位置层，`WallConstraintSolver.cs`）：每次接触迭代前扫 3×3 格，若 body 距障碍格中心小于 `CellRadius + Radius`，就沿格心方向推出 `(hardDistance - distance) * 0.5`，累计进 `WallCorrection`（参与最后的「硬碰撞」判定和速度重建）。

环境语义被封装成 `GridObstacleView`（IsBlocked / CellCenter），导航读的是 `FlowNavigationView`（IsReachable / direction）。两套语义共用同一份 `FlowFieldCell` 存储，但都不直接解读 `Cost` 字段，为将来把地形代价、物理障碍、动态障碍拆成独立后端留了接口。

---

## 5. 性能、确定性与工程化

### Q5.1 接触管线的主要性能优化排序是什么？

**回答：**

按收益从大到小：

1. **Timestep Contact Set 缓存**：避免每个 substep 重跑 BroadPhase + 分类（全量候选扫描 -72%），这是最大头；
2. **持久候选 + 增量修补**：跨帧保留邻居拓扑和生命周期，把每帧重分类变成只分类 dirty incident 对；
3. **O(1) 哈希索引**：`PersistentContactIndex` 和 `IncidentPairLookup` 替代排序 + 二分；
4. **Dormant 调度**：远处对不参与任何求解，只在预定子步唤醒检查；
5. **Jacobi 并行**：pair 评估和 body gather 都是无冲突并行，Burst 编译后吃满多核；
6. **elibility filter + 线性扫描退化**：避免增量路径在脏集过大时做无用功。

---

### Q5.2 怎么保证确定性（Determinism）？

**回答：**

多个层面：

1. **排序**：所有 pair 列表、邻居对、调度表、代理数组在写入后都按稳定比较器排序去重（`ContactConstraintComparer`、`StableEntityPairKeyComparer` 等），消除并行写入顺序带来的不确定性；
2. **无原子操作**：Jacobi 修正走 CSR 前缀和收集，每个 body 的求和顺序固定；
3. **确定性 fallback**：`DeterministicFallbackNormal(bodyA, bodyB)` 用 body 索引哈希固定选择 +X 或 +Z，不依赖 float 比较的瞬时状态；
4. **版本语义**：`MotionVersion` 通过轨迹字段逐位相等推进（不是 32-bit hash 当正确性依据），`TopologyEpoch`/`ClassificationEpoch` 单调递增；
5. **不读旧帧接触状态**：分类、调度、稳定法线只依赖当前帧证据，A0（无持久）与 A1（持久）路径输出同一中层 InteractionSet，差异只在来源成本（README/架构文档明确写了这条不变量）；
6. **Oracle** 独立复算 + CI 脚本静态保护（例如禁止从 CacheGeneration 推导 SimulationStepId、禁止 Oracle 写游戏状态）。

---

### Q5.3 RTS_CONTACT_DIAGNOSTICS 宏的零开销承诺是怎么做到的？

**回答：**

诊断代码全部用 `#if RTS_CONTACT_DIAGNOSTICS` 包裹：

- 诊断容器（统计 `NativeReference`、Pair 诊断列表、迭代遥测）在字段声明层就不存在；
- `PredictiveDiscContactStatistics` 在宏关闭时保留同名 `IComponentData` 契约（属性返回空值），但所有计数器字段和计算分支被 Burst 编译期消除；
- `ContactPipelineConfiguration.EnableDiagnostics` 在宏关闭时是常量 false getter，让每个诊断判断成为编译期分支，而不是运行时字段；
- Oracle、CSV 录制、调试面板捕获全部走宏或运行时 `EnableDiagnostics` 门控。

CI 脚本 `validate_contact_diagnostics.py` 静态检查：诊断关闭时不允许出现额外 NativeContainer 字段、额外 Job、Profiler 读取。正确性路径（证书、逃逸修复、全量回退）不依赖诊断，编译开关只影响观测。

---

### Q5.4 CI 静态合约具体检查什么？

**回答：**

`.github/scripts/` 四条 Python 脚本：

1. `validate_contact_architecture.py`：层所有权——四个阶段 ABI 必须存在、调度器不能实现算法、算法 Job 必须位于 `Scheduling/Parallel/Jobs`、历史 `Jobs/ContactPipeline` 布局禁止回归；
2. `validate_contact_diagnostics.py`：诊断关闭零额外 container / job / profiler；
3. `validate_contact_pipeline_audit.py`：禁止 aggregate bag；调度器不可实现算法；
4. `validate_contact_static_contracts.py`：调度步骤身份不可从缓存 generation 派生；Oracle 不写游戏状态。

它不能替代 Unity 编译 / Burst / Collections Safety / 运行时验证，但把最容易悄悄回归的架构红线变成了 merge 阻塞项。

---

### Q5.5 NativeContainer 的生命周期是怎么管理的？

**回答：**

三类生命周期：

1. **世界级**：`InteractionCandidateStore` 在 `BaseFlowMovementSystem.OnCreate` 用 `Allocator.Persistent` 创建，`OnDestroy` 统一 Dispose；容量按单位数自适应扩容（`RequiresCapacity`：incident 表按 `unitCount * 64`、空间哈希按 `unitCount * 128` 预分配，避免首帧 rehash）；
2. **帧级**：`CrowdStepBodyResources`、`CertificationFrameResources`、`SoftAvoidanceFrameResources`、`ConstraintSolverFrameResources`、`ExecutionResources` 各自创建本帧容器，在 `OnUpdate` 末尾把 Dispose Job 挂到所有消费 Job 的依赖链上（`body.Dispose(applyHandle)` + `CombineDependencies(...)`），保证使用完毕才释放；
3. **网格级**：`FlowFieldGrid.Grid/PendingGrid` 由 `FlowFieldBakeSystem` 创建/交换/释放，通过 `RegisterActiveGridReader` 记录读者句柄，写 PendingGrid 前等旧读者完成。

另外 `FlowFieldGrid` 的 NativeArray 直接放在 `IComponentData` 里（非托管 struct 允许），这是刻意的取舍——必须手动管理生命周期，所以所有创建/销毁/交换都集中在 BakeSystem 一处，避免散落。

---

## 6. 网络与事件溯源回放

### Q6.1 网络架构是什么样的？

**回答：**

基于 Unity NetCode for Entities：**服务端权威 + 客户端预测**。

- `NetCodeUnitFlowMovementSystem`（`PredictedSimulationSystemGroup`，要求 `NetworkStreamInGame`）和 `LocalUnitFlowMovementSystem`（`SimulationSystemGroup`，要求 `LocalInstance`）都继承同一个 `BaseFlowMovementSystem`，完全共享求解管线——网络模式与本地模式唯一的差别是系统过滤条件和运行组；
- 输入链路：客户端 `UnitMoveInputSystem`（`GhostInputSystemGroup`）右键 → 快照选中单位 → 本地 `MoveOrder` 生效（预测立刻跑起来）→ 同时 `RequestCommandRpcSystem.SendInputCommand` 通过 RPC 发给服务端；
- 服务端 `MoveOrderReceiveSystem` 接收 `RequestMoveOrderRPC`，写入 `MoveOrder` 后进入同一套 `RtsCommandSystem → 流场 → BaseFlowMovementSystem` 路径，权威结果通过 Ghost 同步回客户端；
- `FlowArrivalState` 等关键状态标记了 `[GhostComponent(AllPredicted)]` / `[GhostField]`，进入预测与回滚协议。

---

### Q6.2 右键移动指令从输入到单位动起来，完整链路是什么？

**回答：**

1. `UnitMoveInputSystem` 右键 → 射线打到地面 → 把当前选中的单位实体**快照**进 `MoveOrderSelectionElement` DynamicBuffer，并启用 `MoveOrder` 组件；同时发 RPC（并顺带录制回放指令）；
2. `RtsCommandSystem`（`LocalSimulation`）消费 `MoveOrder`：对快照里的单位按实体 (Index, Version) 排序 → 生成目标点周围的可行走槽位 → 按原队形贪心分配 → 给每个单位写 `UnitMoveDestination`（槽位、到达半径、直接驶入距离、`OrderVersion`）→ 递增 `RecalculateFlowFieldTag.RequestVersion` 并启用烘焙请求；
3. `FlowFieldBakeSystem` 在 PendingGrid 上重跑 Integration + Vector（Cost 没脏就复用）→ 双缓冲发布，`ActiveVersion++`；
4. `BaseFlowMovementSystem.OnUpdate`：`BuildCrowdMotionIntentJob` 读格子方向生成移动意图 → 接触管线认证/避让/求解 → `ApplyFlowMovementJob` 把结果写回 `LocalTransform` 和 `Velocity`。

两个值得讲的细节：

- **选中快照语义**：`MoveOrderSelectionElement` 在指令下达那一刻就固定了接收者，消费时不能改查实时 `UnitSelected`，否则输入与预测更新的时序差会把订单绑到之后的选择上；
- **订单版本守卫**：`UnitMoveDestination.OrderVersion != ActiveRequestVersion` 的单位会被立即置为 settled，避免旧订单的目标在流场重烘焙完成前误导移动。

---

### Q6.3 事件溯源回放是怎么实现的？为什么能瞬间重置？

**回答：**

核心是「**只录输入，不录状态**」：

- 按 `L` 开始录制：`RequestCommandRpcSystem` 在发送指令的同时把 `ReplayCommandElement`（类型 + 目标点 + 相对时间戳 `TimeOffset`）追加进 `ReplaySystemState` 实体上的 DynamicBuffer；
- 按 `R` 开始回放：断开当前网络连接 → 摧毁所有 `BasicUnitTag` 单位 → **强制重置流场**（把 `RecalculateFlowFieldTag` 失效、目标归零、用 `ClearGridJob.Run` 清空 Integration/Vector，但保留 Cost 障碍数据）→ 加上 `LocalInstance` 让系统切到本地模式 → 之后每帧按 `PlaybackIndex` 和 `TimeOffset` 取出到期的指令，通过 `EntityCommandBuffer` 批量执行（生成单位、设置 `MoveOrder`）；
- 零快照意味着不需要保存任何 Transform 历史，重置成本就是「清实体 + 清网格 + 重放指令」，所以可以毫秒级回到任意起点。

一个回放正确性细节：录制时 `cmd.Position` 已包含客户端生成的偏移，回放时**不得再次叠加**（代码注释里明确写了），否则第二次回放会累积偏移。

---

## 7. 框架与外围系统

### Q7.1 项目里除了 DOTS 还用了什么框架？

**回答：**

- **QFramework 风格架构**（`_QFrameWork` + `MainGameArchitecture.cs`）：`Architecture<MainGameArchitecture>` 注册 `BuildingUtility`、`MapUIModel`、`EcoUIModel`，UI 侧走 MVC（Controller / Model / Command + Event），建造管理走 Command（`CreateBuildingCommand` 等）；
- **Service Locator**（`SystemServiceLocator.cs` / `ServerObjectSystem.cs`）：`ConcurrentDictionary<Type, object>` 的 `ISystemService`，`ServiceSystemBase<T>` 让 ECS System 也能按接口注册/获取服务（比如 `RequestCommandRpcSystem` 通过 `this.GetService<RequestCommandRpcSystem>()` 被输入系统调用）；
- **Unity Physics** 只用来做静态障碍采样（Cost Field）和单位 footprint 计算，单位本身保持 Kinematic；
- **NetCode for Entities** 负责连接、Ghost、RPC。

---

## 8. 已知限制（面试「你觉得还有什么不足」的标准答案）

必须诚实、并且每个限制都能接一句「如果让我做，我会…」：

1. **Contact Island 休眠未实现**：持续活跃接触始终参与求解。→ 后续可以做 island 检测 + 睡眠唤醒，或者按区域冻结；
2. **Gauss-Seidel 无并行路径**：需要图着色或冲突无关批次。→ 目前用 Jacobi 换并行性，GS 仅作串行参考；
3. **持久 Spatial Membership 依赖容量上限**：超限回退全量扫描。→ 可以改成可增长容器或分级网格；
4. **流场是全局烘焙**：动态障碍变化只能整张网格重烘焙，没有局部增量更新。→ 可以引入局部 Cost 修补 + 增量 BFS（LPA*/D* Lite 思路）；
5. **README 与代码阈值不一致**（35% vs 70%）：文档没跟上代码演进，需要同步；
6. **网络与本地复用同一套 BaseFlowMovementSystem**，但服务端没有客户端那样的本地输入回放/录制，联调需要更完整的自动化验证（仓库里已有 `LocalGameplayModeValidation` 和 Benchmark Tuner，可以扩展成 CI 回归）；
7. **确定性只在单机验证**：跨平台（ARM/x86、不同 Burst 版本）的 bit-exact 复现没有专门 CI 覆盖。

---

## 9. 高频追问速查表

| 追问 | 一句话答案 |
|---|---|
| 为什么不用 Unity Physics？ | 单位是圆盘，只需要 swept-disc + XPBD；通用刚体/岛系统是负担，且无法做增量拓扑 |
| Swept disc 最近距离怎么算？ | 相对位移投影：`closestTime = clamp(-dot(r0, vRel) / |vRel|², 0, 1)`，再比 `radiusSum + PredictiveSkin` |
| 为什么排序能保证确定性？ | 所有并行写入先写 scratch，再按稳定 key 排序去重，消费顺序唯一 |
| 为什么 Jacobi 用平均而不是直接加？ | 直接加会 overshoot/震荡；平均保持对称且无原子竞争，迭代次数更多但可并行 |
| 证书过期后下游怎么办？ | fail-closed：`SolverSkipReason` 记录原因，跳过该子步求解，`RecoveryRequired` 触发恢复路径 |
| 缓存失效的代价是什么？ | 回退 `FullRebuildPersistentNeighborTopology`（O(N) uniform grid broadphase），正确性不依赖缓存 |
| 单位数变化（增删）怎么处理？ | `EntitySet` dirty → 强制全量重建；容量不够先 `EnsureCapacity` 再重算 |
| 多 World 支持吗？ | 支持：证书绑定 `WorldId`，`SimulationDebuggerRuntime` 按 World 注册，实验覆盖也按 World 隔离 |
| 怎么定位性能瓶颈？ | `RTS_CONTACT_DIAGNOSTICS` + F8 调试面板 + F6/F7 CSV 录制 + Benchmark Tuner 自动参数搜索 |
| 这个项目最有成就感的部分？ | 增量接触管线的「Guard 完备性证明 + 证书 + Oracle」三位一体：既快又敢证明自己没错 |

---

## 10. 面试自检清单

讲完项目后，确保你能脱口而出：

- [ ] 完整数据流（指令 → 流场 → 接触 → 软避让 → XPBD → 写回）每一层的一个关键类名；
- [ ] 增量管线的三个核心数据结构（Proxy / NeighborPairs / PredictiveContacts）各自的职责；
- [ ] 脏体阈值为什么是 70%，以及 README 与代码的差异；
- [ ] XPBD 公式能现场写出来（alpha、deltaLambda、lambda clamp、correction）；
- [ ] 证书的六个验证位和一个 Issued 位分别证明什么；
- [ ] 1k 基准的关键数字（62.7%、596→721、24.94→6.98）；
- [ ] 三个已知限制 + 各自的改进思路；
- [ ] Oracle 为什么只观测不写状态（CI 强制）。
