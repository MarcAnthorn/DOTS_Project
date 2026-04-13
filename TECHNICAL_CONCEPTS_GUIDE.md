# DOTS-RTS 技术概念详解手册（从零补基础）

> 用途：为不熟悉 DOTS 生态的同学准备的「概念词典 + 落地手册」。
> 每个概念按统一模板讲解：**① 是什么（含类比）② 为什么需要 ③ 在这个项目里怎么落地**。
> 读完本文再回头看 README 和面试问答手册，应该会顺畅很多。

---

# Part A DOTS 与数据导向基础

## A1. ECS（Entity-Component-System，实体-组件-系统）

**① 是什么**

传统 OOP 编程把「对象」做成一个类，对象里同时装着数据（血量、位置）和行为（移动、攻击）。ECS 把这两件事拆开：

- **Entity（实体）**：只是一个编号，本身没有内容，像超市里手推车上的编号牌；
- **Component（组件）**：纯数据，比如「位置」「速度」「半径」，像放在推车里的货物；
- **System（系统）**：处理逻辑，每帧遍历「带某些组件的实体」批量干活，像流水线上的工人，只处理自己负责的货物。

**② 为什么需要**

RTS 有成千上万个单位，传统对象模型的 CPU 缓存命中率很差（一个对象的数据散落在内存各处），并且逻辑难以并行。ECS 把相同类型的组件连续存放在一起（见 A7），System 遍历时就像顺序读一个大数组，天然适合多线程并行。

**③ 在项目里怎么落地**

单位的「数据」就是各种组件，例如：

```csharp
public struct BasicUnitTag : IComponentData { }          // 标记：我是单位
public struct UnitMoveDestination : IComponentData { }   // 目的地
public struct UnitMoveSpeed : IComponentData { }         // 移动速度
public struct Velocity : IComponentData { }              // 当前速度
```

逻辑都在 System 里：`BuildCrowdMotionIntentJob`（一个 `IJobEntity`）遍历「同时拥有 LocalTransform、Velocity、UnitMoveSpeed、UnitMoveDestination 等组件」的所有实体，批量生成移动意图；`ApplyFlowMovementJob` 再遍历同一批实体，把求解结果写回位置和速度。

---

## A2. World 与 System 分组（SystemGroup、WorldSystemFilter）

**① 是什么**

- **World**：一个独立的 ECS 容器，里面有各自的实体和系统。可以同时存在多个 World（服务端一个、客户端一个、本地模式一个）。
- **SystemGroup**：把系统按执行顺序分组（例如「模拟组」「表现组」），System 通过 `[UpdateInGroup]` 挂进某个组，通过 `[UpdateAfter]` / `[UpdateBefore]` 声明先后。
- **WorldSystemFilter**：声明「我这个系统只在这个 World 里运行」，比如只跑在服务端 World 或只跑在客户端 World。

**② 为什么需要**

一个游戏帧里有很多系统，必须规定谁先谁后（输入必须先于移动、移动必须先于渲染）；同时网络程序里服务端、客户端、本地演示跑的是不同逻辑，必须能区分。

**③ 在项目里怎么落地**

```csharp
[WorldSystemFilter(WorldSystemFilterFlags.LocalSimulation)]   // 只在本地模拟 World 跑
[UpdateInGroup(typeof(PredictedSimulationSystemGroup))]      // 挂在预测模拟组
public partial class RtsCommandSystem : SystemBase { ... }
```

`FlowFieldBakeSystem` 挂在 `SimulationSystemGroup` 且在 `FixedStepSimulationSystemGroup` 之后；`LocalUnitFlowMovementSystem` 要求存在 `LocalInstance` 组件才运行，`NetCodeUnitFlowMovementSystem` 要求存在 `NetworkStreamInGame` 才运行——这是本地模式与联网模式切换的机制。

---

## A3. 组件类型：IComponentData / DynamicBuffer / Tag / IEnableableComponent

**① 是什么**

- **IComponentData**：一个实体上的「一份数据」，例如 `float3 Position`；
- **DynamicBuffer**：实体上的「一列数据」，类似 `List<T>`，但存在 ECS 的块内存里；
- **Tag 组件**：没有任何字段的空组件，只用来标记实体「属于某类」；
- **IEnableableComponent**：可以在运行时「启用/禁用」的组件。禁用不等于删除，只是让查询暂时不包含它，比销毁重建便宜得多。

**② 为什么需要**

有了这些，System 才能精确描述「我要处理哪类实体」，避免每次遍历都做类型判断。

**③ 在项目里怎么落地**

- Tag：`BasicUnitTag` 标记单位、`LocalInstance` 标记本地模式的实体；
- DynamicBuffer：`MoveOrderSelectionElement` 保存一次右键指令的选中单位快照；`ReplayCommandElement` 保存回放指令列表；
- IEnableableComponent：`MoveOrder` 平时禁用，右键下达指令后启用，`RtsCommandSystem` 消费完立刻禁用——用「开关」代替「建删实体」；
- `RecalculateFlowFieldTag` 同理，作为流场重烘焙的请求开关，还带 `RequestVersion` 计数器区分新旧请求。

---

## A4. Jobs：IJob / IJobEntity / IJobParallelFor / JobHandle 依赖链

**① 是什么**

Unity 的 Job System 是「把一段纯计算交给工作线程跑」的框架：

- **IJob**：一个整体任务，只有一个 Execute 方法，跑在一个工作线程上；
- **IJobParallelFor**：一个任务被拆成 N 份，每份处理一个数组下标，多个线程同时跑；
- **IJobEntity**：按 ECS 查询自动遍历实体，每个实体执行一次（背后也是并行）；
- **JobHandle**：任务的「票据」。你拿到句柄，就能声明「这个任务完成后才能做下一个」；`JobHandle.CombineDependencies` 把多个任务合并成一个前置依赖。

**② 为什么需要**

单线程计算 5,000 个单位的物理接触根本跑不满 CPU。Job System 让计算自动利用多核，而且用依赖链（而不是锁）保证数据安全：任务 A 写出的数据，任务 B 等 A 的句柄完成后再读，就不会有竞争。

**③ 在项目里怎么落地**

`BaseFlowMovementSystem.OnUpdate` 里每个阶段都是一个 Job，比如：

```csharp
JobHandle footprintHandle = new CalculateUnitCollisionFootprintJob { ... }
    .ScheduleParallel(_movementQuery, Dependency);
JobHandle intentHandle = new BuildCrowdMotionIntentJob { ... }
    .ScheduleParallel(_movementQuery, footprintHandle);   // 等 footprint 完成
JobHandle initializeHandle = new InitializeCrowdStepStateJob { ... }
    .Schedule(unitCount, 64, intentHandle);
```

参数 64 是批大小（每 64 个元素一个批次），最后一行的 `Dispose(finalReader)` 保证容器被释放时所有读它的 Job 都已结束。

---

## A5. Burst 编译器

**① 是什么**

Burst 是 Unity 附带的一个编译器，可以把 Job 里的 C# 代码（限制为无托管引用的子集）编译成高度优化的原生机器码，通常能带来几倍到几十倍的性能提升。它依赖 Job 结构体「可 Blittable」（内存布局简单、不包含托管对象引用）。

**② 为什么需要**

接触求解的内层循环（逐 pair 计算最近距离、逐迭代投影）每帧要执行几十万次，解释执行的 C# 不可能扛住；Burst 把这些循环变成紧凑的 SIMD 友好原生代码，是 5,000 单位实时模拟的基石之一。

**③ 在项目里怎么落地**

所有热路径 Job 都标了 `[BurstCompile]`：`GenerateCostFieldJob`、`BuildCrowdMotionIntentJob`、`InteractionCertificationJob`、`ConstraintSolverJob`、`ApplyFlowMovementJob` 等等。为了配合 Burst，项目把算法从 Job 里抽成 `Kernels/` 下的纯静态函数（如 `PersistentContactMath`、`SoftAvoidanceMath`），参数全部显式传入，没有隐藏的实例状态。

---

## A6. NativeContainer 与 Allocator / Collections Safety

**① 是什么**

普通 C# 数组由垃圾回收器（GC）管理，且会被移动；Job 线程不能安全访问它们。Unity 提供 **NativeContainer**——一块非托管内存的包装类型，例如：

- `NativeArray<T>`：定长数组；
- `NativeList<T>`：可扩容列表；
- `NativeHashMap<K,V>`：哈希表（单线程写）；
- `NativeParallelMultiHashMap<K,V>`：允许并行写入的「一个键对应多个值」哈希表；
- `NativeReference<T>`：单个值的容器，方便 Job 之间传递结果；
- `NativeQueue<T>`：队列。

它们必须声明分配器：`Allocator.Persistent`（长期存在，必须手动释放）、`Allocator.TempJob`（跨 Job 的临时内存，一帧内释放）、`Allocator.Temp`（主线程极短期）。Collections Safety 系统会在调试模式下检查「容器是否被释放、是否被多个写者同时写」。

**② 为什么需要**

没有它们，Job 无法共享大块数据，也无法保证线程安全。它们的释放是显式的（不像 GC 自动回收），所以性能可控、可预测。

**③ 在项目里怎么落地**

项目里到处是 NativeContainer：

- `FlowFieldGrid` 里的 `Grid / PendingGrid` 是两块 `NativeArray<FlowFieldCell>`，由 `FlowFieldBakeSystem` 创建、交换、释放（`Allocator.Persistent`）；
- 每帧的 body 数据（`Bodies`、`StepStates` 等 7 个数组）用 `Allocator.TempJob` 创建，帧末挂到依赖链上统一 `Dispose`；
- 持久候选库 `InteractionCandidateStore` 用 `Allocator.Persistent` 持有 `NativeList` / `NativeHashMap`，系统创建时分配、销毁时释放，并按单位数量预扩容量（incident 表 `unitCount * 64`、空间哈希 `unitCount * 128`）；
- `SweptCellEntries` 是 `NativeList<SweptDiscCellEntry>`，BroadPhase 先把每个 body 撒进格子，再排序成连续区间；
- `NativeQueue<int2>` 用作 BFS 的待访问队列。

---

## A7. 数据导向设计（SoA / 缓存友好 / 确定性）

**① 是什么**

数据导向设计（Data-Oriented Design，DOD）主张「先设计数据的组织和访问模式，再写逻辑」。核心手法是 **SoA（Structure of Arrays）**：同一类字段连续排列，比如 5,000 个单位的半径放在一个连续数组里，而不是 5,000 个对象各自带着半径。

**② 为什么需要**

现代 CPU 读内存是按缓存行（通常 64 字节）批量读的。连续数组遍历时，读一个缓存行能用上多个数据；对象散列时，读一个缓存行可能只有一个有用数据，其余全浪费——内存带宽成了瓶颈。另外连续数组也方便直接切成多段并行处理。

**③ 在项目里怎么落地**

每帧的模拟数据就是一组平行数组：`Bodies`、`NavigationStates`、`MotionIntents`、`MotionEvidence`、`StepStates`、`Results`，下标即单位编号，Job 按 `[EntityIndexInQuery]` 把实体映射到数组下标。流场网格本身就是一个 `NativeArray<FlowFieldCell>`（连续存放所有格子），单位查询某格是 O(1) 下标访问。

---

# Part B 寻路与移动

## B1. 流场寻路（Flow Field Pathfinding）

**① 是什么**

流场寻路是「为整个地图的每一个格子预计算一个方向」的寻路方案：从目标点向外扩散，算出每个格子「往哪个邻居走离目标最近」，得到一个覆盖全图的方向场。单位寻路时不用搜索，直接查自己所在格子的方向即可。

可以这样类比：A* 是「每个人各自打开手机导航规划一条路线」，流场是「城市在每个路口画好路标，任何人走到路口看路标就能走」。

**② 为什么需要**

5,000 个单位同时下达移动指令，如果每个单位跑一次 A*，是 5,000 次独立搜索。而流场只搜一次（以目标点为源），之后所有单位共享结果，单次查询 O(1)。RTS 里大量单位往往共享同一目标区域，这是最划算的方案。

**③ 在项目里怎么落地**

整体链路：`RtsCommandSystem` 写目标 → 置位 `RecalculateFlowFieldTag` → `FlowFieldBakeSystem` 检测到请求，在 PendingGrid 上执行三阶段烘焙（见 B2）→ 双缓冲发布（见 B4）→ `BuildCrowdMotionIntentJob` 每个单位读 `BestDirectionIndex` 生成移动意图。

---

## B2. Cost / Integration / Vector 三层字段

**① 是什么**

流场网格 `FlowFieldCell` 只有三个字段，对应烘焙的三个阶段：

- **Cost Field（代价场）**：每格标记「能不能走」（0 = 障碍，1 = 可行走）；
- **Integration Field（积分场）**：每格记录「到目标的 BFS 步数」（数值越大离目标越远，`ushort.MaxValue` 表示不可达）；
- **Vector Field（向量场）**：每格记录「8 个邻居里积分值最小的那个方向」（0~7，对应 8 邻域）。

**② 为什么需要**

把「能不能走」「多远」「往哪走」拆成三个独立阶段，是因为它们的失效条件不同：障碍变了才需要重算 Cost；目标变了只需要重算 Integration + Vector；Cost 完全可以复用。这是「按需烘焙」的基础。

**③ 在项目里怎么落地**

- Cost：`GenerateCostFieldJob`（IJobParallelFor）对每格中心调用 Unity Physics 的 `CollisionWorld.CalculateDistance`，命中障碍层就把 Cost 置 0；
- Integration：`GenerateIntegrationFieldJob` 用 `NativeQueue<int2>` 做 8 邻域 BFS，从目标格出发，每格 IntegrationValue = 父格 + 1，障碍格不扩散（见 B3）；
- Vector：`GenerateVectorFieldJob`（IJobParallelFor）每格独立比较 8 个邻居的 IntegrationValue，取最小值方向写入 `BestDirectionIndex`；无法到达的格子写 `0xFF` 表示无效。

---

## B3. BFS（广度优先搜索）

**① 是什么**

BFS 是从一个起点开始、一圈一圈向外扩散的图搜索算法：先访问起点（距离 0），再访问起点所有邻居（距离 1），再访问它们的邻居（距离 2）……像水面涟漪。它保证「第一次访问到某格时，走的就是最短步数」。

**② 为什么需要**

网格寻路需要每个格子到目标的距离；BFS 在等代价网格上就是「最短步数」，实现简单且确定性好（用队列先进先出，扩散顺序唯一）。相比 Dijkstra/A*，这里所有格子移动代价相同（Cost 只有 0/1），BFS 就够了。

**③ 在项目里怎么落地**

`GenerateIntegrationFieldJob`：目标格 IntegrationValue = 0 入队；循环出队，检查 8 个邻居，若邻居可行走且尚未被赋值（`IntegrationValue == ushort.MaxValue`），赋「当前值 + 1」并入队；队列空则完成。目标格本身是障碍时直接返回（整场不可达）。

---

## B4. 双缓冲（Double Buffering）与异步烘焙

**① 是什么**

双缓冲是「写一份、读一份、写满后原子交换」的经典技术：正在被读者使用的数据不会被写者碰，写者永远在另一份副本上工作。类似页面渲染里的前台帧/后台帧。

**② 为什么需要**

流场烘焙很贵（可能要几帧），而单位每帧都在读流场。如果不加缓冲，读者可能读到「一半新一半旧」的网格；如果加锁等烘焙完成，主线程会被卡住。双缓冲让「上一份完整快照继续可读」与「新快照在后台慢慢算」同时成立。

**③ 在项目里怎么落地**

`FlowFieldGrid` 持有 `Grid`（已发布）和 `PendingGrid`（烘焙中）：

1. 新请求到达 → `PreparePendingFlowFieldJob` 把已发布的 Cost 复制进 PendingGrid，重置 Integration/Vector；
2. 在 PendingGrid 上跑 BFS + Vector；
3. 完成后一次性交换两个数组引用，`ActiveVersion++`，读者立刻看到新快照；
4. 交换出去的旧 ActiveGrid 变成下一次的 PendingGrid，写入前必须等旧读者完成——`FlowFieldBakeSystem.RegisterActiveGridReader()` 收集所有读者句柄（移动管线每帧把整个求解句柄注册进来），保证这个等待；
5. 版本守卫：烘焙期间如果请求版本变了（来了新目标），旧结果直接丢弃，在 PendingGrid 上重跑最新版。

---

## B5. 空间查询与网格坐标系（WorldToCell / 8 邻域 / uniform grid）

**① 是什么**

- **WorldToCell**：把世界坐标换算成格子坐标：`(世界坐标 - 网格原点) / 格子尺寸` 取整；
- **8 邻域**：一个格子周围的 8 个格子（上、下、左、右 + 4 个对角）；
- **Uniform Grid（均匀网格）**：把连续空间切成固定大小的格子，物体按所在格子登记，查邻居只查邻近格子，是一种最简单的空间索引。

**② 为什么需要**

「单位在哪格」「相邻格是谁」是寻路和接触检测共同的基础操作。Uniform Grid 用 O(1) 的整数运算代替昂贵的浮点距离搜索，是 RTS 大规模场景的空间索引起点。

**③ 在项目里怎么落地**

- `FlowFieldUtils.WorldToCell` / `GetFlatIndex`（`y * width + x`）把二维格子压成一维数组下标；
- `GetDirectionOffset` 把 0~7 方向索引映射成 `int2` 偏移；
- 接触检测的 BroadPhase 也用同一套网格：`SweptDiscCellEntry` 记录「格下标 + body 下标」，排序后同一格内的 body 两两组成候选对（见 C1）。

---

## B6. 到达滞回（Hysteresis）与减速距离

**① 是什么**

- **滞回**：进入某个状态和退出某个状态使用不同阈值，避免在边界反复横跳。例如空调：26°C 开启制冷，28°C 才关闭，中间 2°C 是缓冲带；
- **减速距离**：距离目标足够近时，速度按距离比例线性下降，让单位平滑停下而不是急刹。

**② 为什么需要**

如果不加滞回，单位在到达区边缘会「这一帧判定到了停下、下一帧又判定没到重新启动」，表现为终点抖动。如果不减速，高速单位会冲过目标再折返，来回震荡。

**③ 在项目里怎么落地**

`FlowArrivalState.IsSettled` 跨帧保留。进入到达区用较小的 `ArrivalRadius`，已 settled 后要用更大的退出半径（`arrivalEnterRadius + radius * 0.5`）才会重新激活。直接驶入阶段用 `speedScale = saturate(distance / brakingDistance)` 线性减速；settled 状态下速度还被额外乘以 `0.8^(dt*60)` 做指数阻尼，让单位彻底安静下来。

---

## B7. 编队槽位分配（Formation Slots）

**① 是什么**

一次移动指令可能包含几十上百个单位，它们不能全部挤向同一个点。系统在目标点周围生成一排排「停车位」（槽位），再按每个单位原来的相对位置分配一个槽位，让整支队伍保持队形移动。

**② 为什么需要**

没有槽位，所有单位会往同一个点挤，在终点堆成一团；没有按原队形分配，队伍到达后会乱序重排。RTS 玩家对「编队整齐」有明确预期。

**③ 在项目里怎么落地**

`MoveDestinationSlotUtility`：

1. `GenerateWalkableSlots`：从目标点开始按环（ring = 0、1、2…）生成间距固定的候选点，跳过 Cost=0 的障碍格，直到凑够数量；
2. `AssignSlotsPreservingFormation`：先算选中单位质心，把「每个单位相对质心的偏移」投影到目标点，得到它的理想槽位；按「离质心最远的先选」（贪心 + 确定性 tie-break），每个单位选离自己理想位置最近的空闲槽。

槽位分配只在订单到来时执行一次，不进逐帧热路径。每个单位拿到自己的 `UnitMoveDestination`（槽位坐标、到达半径、直接驶入距离、订单版本号）。

---

## B8. 移动意图与转向（Motion Intent / Steering）

**① 是什么**

「移动意图」是寻路层输出的中间产物：单位此刻想以什么速度往哪走（`PreferredVelocity`）。它不直接写位置，而是交给后续的避让/求解阶段去修正，类似自动驾驶的「规划层」与「控制层」分离。

**② 为什么需要**

如果把「寻路方向」直接当最终速度，单位会互相穿透、撞墙、挤成一团。必须把「想走的方向」和「实际能走的方向」分开：先有意图，再经过软避让（Part E）、接触求解，最后才变成位置变化。

**③ 在项目里怎么落地**

`BuildCrowdMotionIntentJob` 输出 `CrowdMotionIntent`：

```csharp
intent.PreferredVelocity = preferredDirection * speed.Value;
intent.SteeringVelocityError = preferredVelocity - body.Velocity;
```

`SteeringVelocityError` 是「期望速度与当前速度的差」，即转向力。后续 `ContactPipelineMath.CalculateBaseVelocityForSubstep` 把它限制在 `MaxAcceleration` 内，累加到当前速度上形成本子步的基准速度，再进入软避让和约束求解。

---

# Part C 接触检测

## C1. BroadPhase / NarrowPhase（宽相 / 窄相）

**① 是什么**

碰撞检测的标准两段式：

- **BroadPhase（宽相）**：用廉价手段筛掉「明显不可能相交」的物体对，输出候选对。常用手段是网格/空间哈希/AABB；
- **NarrowPhase（窄相）**：对候选对做精确的几何测试，判定是否真的接触、计算法线和穿透量。

**② 为什么需要**

全量两两比较是 O(N²)，5,000 个单位就是 1,250 万对，即使每对只花一纳秒也要 12.5ms。宽相先用网格把每对比较缩小到「同一格/邻近格」，候选对数量大幅下降，窄相只处理这些候选。

**③ 在项目里怎么落地**

- BroadPhase：`BuildSweptInteractionPairs` / `FullRebuildPersistentNeighborTopology` 把每个 body 的扫掠包络撒进格子（`SweptCellEntries`），排序后同格内两两配对，再按 BodyA/BodyB 排序去重；
- NarrowPhase：`FilterAndClassifyPairs` / `ClassifyPersistentNeighborPair` 对每个候选对做精确的 swept-disc 最近距离测试（见 C2），判定生命周期和接触模式。

增量管线（Part C 后面）本质上就是「把 BroadPhase 的结果跨帧缓存，只对变化部分重做宽相 + 窄相」。

---

## C2. Swept Disc 与相对运动最近距离

**① 是什么**

- **Disc（圆盘）**：单位在 XZ 平面被近似成圆（半径来自碰撞体 footprint）；
- **Swept（扫掠）**：考虑的是「这一帧从起点移动到终点的整个轨迹」，而不是两个静态圆；
- **相对运动最近距离**：把两个单位放在相对坐标系里（一个不动，另一个带着两者的相对速度运动），求它们在时间 `[0,1]` 内最近的距离。公式是点积投影：

```
closestTime = clamp(-dot(relativeStart, relativeDisplacement) / |relativeDisplacement|², 0, 1)
minDistance  = |relativeStart + closestTime * relativeDisplacement|
```

**② 为什么需要**

RTS 单位速度很快、子步时间不小，只看「帧首位置」会漏掉「帧中擦肩而过」的碰撞（隧穿）。扫掠测试保证：只要轨迹上某时刻两圆距离小于半径和，就能被检测到。这也是「预测接触（Predictive）」的来源——还没碰到，但预测会碰到。

**③ 在项目里怎么落地**

`PersistentContactMath.CalculatePairClosestTime` 和 `FilterAndClassifyPairs` 都用这个公式。判断阈值：

- `candidateDistance = radiusSum + PredictiveSkin`：进入「未来可能接触」的范围；
- `retainedDistance = candidateDistance + TimestepContactMargin * 2`：保留余量，超出就 Expired。

Oracle（D6）也用完全相同的 swept-disc 测试独立复算所有对，用于验证增量结果。

---

## C3. AABB / 包络（Envelope）

**① 是什么**

- **AABB（Axis-Aligned Bounding Box）**：轴对齐包围盒，即「恰好包住一个物体的最小矩形/立方体」，边与世界坐标轴平行；
- **包络（Envelope）**：在 AABB 基础上考虑运动范围——把「轨迹起点、轨迹终点、加上半径和预测余量」都包进去，得到这一帧该单位所有可能到达的空间范围。

**② 为什么需要**

圆的相交测试比 AABB 相交测试贵；而 AABB 相交测试又只有「4 次比较」的廉价实现。用 AABB 先粗筛，再用精确几何精判，是标准的分层剔除。包络则保证「运动中的单位」也能被粗筛抓住。

**③ 在项目里怎么落地**

每个单位有一组包络（`CrowdMotionEvidence`）：

- `ContactEnvelopeMin/Max`：轨迹起点/终点 + 半径 + PredictiveSkin + margin；
- `InteractionEnvelopeMin/Max`：进一步把软避让 shell、RVO 视域、未约束位置也并进来（`ContactPipelineMath.CalculateInteractionBounds`）。

`ContactPipelineShared.AabbContains(outer, inner)` 就是「外框是否包含内框」的校验，是 Guard 完备性检查（C4）和包络逃逸检测（D4）的核心原子操作。

---

## C4. Guard Bounds 与 TopologyGuard（位置圆守护）

**① 是什么**

`PersistentSweptProxy` 里一个 body 有三套边界，从里到外：

- `TightMin/Max`：最近一次分类时的精确交互包络；
- `GuardMin/Max`：Tight 再外扩 `GuardEnvelopeMargin`——用于证明「我覆盖了所有可能的交互对象」；
- `TopologyGuardMin/Max`：以「当前位置为圆心的位置圆」再外扩 margin——专门用来判断拓扑（邻居关系）是否变化，与速度方向无关。

**② 为什么需要**

增量管线的正确性建立在「上一帧的候选集对本帧仍然完备」之上。Guard 就是这份证明的物证：只要旧 Guard 还能包住当前轨迹，就说明候选集没漏；包不住了，才需要重查这个 body 的邻居。TopologyGuard 的存在是为了把「转向/加速（只改包络）」和「真的跑远（改邻居关系）」区分开，避免每帧转向都触发全局重建。

**③ 在项目里怎么落地**

`PersistentProxyBuilder.BuildFromState` 构造三套边界；`ClassifyAndUpdateForBody` 每帧比对：

```csharp
bool topologyDirty = 有效性/半径变化 || 当前位置圆逃出旧 TopologyGuard;
bool motionDirty  = topologyDirty || MotionVersion 变化;
```

只动包络不动位置 → 只标 `Motion`，滑动 Guard 跟踪新包络；位置逃出位置圆 → 标 `Topology`，需要重查邻居。

---

## C5. 空间哈希 / Incident 索引

**① 是什么**

- **空间哈希**：把「格子坐标」哈希成键，值为该格内的物体列表。查邻居时把周围几格的值取出来。`NativeParallelMultiHashMap<int, int>`（格下标 → body 下标）就是这种结构；
- **Incident 索引（关联对索引）**：为每个实体记录「它参与的所有邻居对」，反向索引：实体 → 对列表。

**② 为什么需要**

增量修补时，我们只知道「哪些 body 脏了」，需要快速回答「这个脏 body 的候选邻居是谁」。空间哈希回答「脏 body 周围有哪些其他 body」；Incident 索引回答「这个 body 在持久邻居池里参与哪些对」。两者互补，都是为了把 O(N) 全量扫描变成 O(邻居数)。

**③ 在项目里怎么落地**

- `PersistentSpatialMembership`（格 → body）配合 `SpatialMembershipEpoch` 用于 `IncrementallyRepairPersistentNeighborTopology` 的局部邻居查询；
- `PersistentIncidentPairLookup`（Entity → pair 下标）配合 `IncidentLookupEpoch` 用于 `DirtyIncidentPairMapper` 的脏体 incident 枚举；
- 两个索引都带 epoch 版本号，与 `TopologyEpoch` 绑定：版本不匹配说明索引过期，直接走正确的全量回退路径，绝不拿旧索引硬查。

---

## C6. 持久候选（Persistent Candidate）

**① 是什么**

「候选」是还没被确认的接触对（宽相产物）；「持久」是说这些候选对跨帧保存，而不是每帧重建。`InteractionCandidateStore` 就是存放它们的「跨帧仓库」。

**② 为什么需要**

RTS 相邻帧的接触拓扑高度相似：上一帧的 600 个接触对，下一帧绝大多数原样保留。与其每帧从零扫描全部 N 个单位找对，不如把「上一帧找到的对」留到这一帧，只修补变化部分。成本从「正比于全体」变成「正比于变化量」。

**③ 在项目里怎么落地**

仓库里的内容：

- `PersistentSweptProxies`：每个 body 的包络 + guard（C4）；
- `PersistentNeighborPairs`：guard 相交的候选对（完备集）；
- `PersistentPredictiveContacts`：每对的跨帧生命周期状态（C9）；
- `PersistentActiveContactKeys` / `PersistentSoftAvoidancePairKeys` / `PersistentDormantContactSchedule`：下游视图的键列表；
- `PredictiveContactIndex`（O(1) 哈希）、`IncidentPairLookup`（反向索引）、`SpatialMembership`（空间索引）。

一个重要纪律：**这些是「候选源数据」，下游消费者永远不允许直接读它们**，只能读认证后的紧凑视图（Part D）。

---

## C7. 脏标记（Dirty Flag）

**① 是什么**

「脏」表示「这份缓存数据可能过时，需要更新」。用位标志描述过时原因，例如：

```csharp
[Flags] enum IncrementalBodyDirtyFlags {
    None = 0,
    Motion = 1 << 0,       // 轨迹/半径变了，包络要刷新
    Topology = 1 << 1,     // 位置逃出守护圆，邻居关系要重查
    EntitySet = 1 << 2,    // 实体集合变了，缓存整体不可信
    CorrectedEscape = 1 << 3  // 求解修正把位置推出了包络
}
```

**② 为什么需要**

缓存不是每帧无脑全量刷新，而是先标记「谁变了、为什么变」，再决定最小代价的更新路径。这就像代码仓库的增量编译：只重编改动的文件。

**③ 在项目里怎么落地**

每帧 `PrepareInitialPersistentDirtyBodySet` 遍历所有 body，由 `PersistentProxyBuilder.ClassifyAndUpdateForBody` 返回脏标记并写入 `IncrementalDirtyBodies`（紧凑列表）+ `IncrementalDirtyFlagsByBody`（O(1) 按 body 查标记的数组）。子步中每次包络逃逸（D4）也会追加脏标记。最后 `SummarizePreparedIncrementalDirtyBodiesP1P6` 统计 `topologyDirtyCount`，决定走增量修补还是全量重建。

---

## C8. 增量修补 vs 全量重建

**① 是什么**

- **增量修补（Incremental Repair）**：保留缓存中「没脏」的部分，只对脏 body 重查邻居、重分类、合并结果；
- **全量重建（Full Rebuild）**：清空所有缓存，重新做完整 BroadPhase + 分类，从零建立新候选集。

**② 为什么需要**

两种路径各有适用场景：变化小时修补便宜，变化接近全体时修补反而要付「保留旧 + 查新 + 合并去重」的额外成本，全量重建更简单更快。项目用「脏比例阈值」做二分决策。

**③ 在项目里怎么落地**

`BuildContactPairsFromPersistentNeighborSet` 里的决策：

```csharp
bool useFullRebuild = !cacheCanBePatched || entitySetDirty ||
                      dirtyRatio > IncrementalDirtyBodyRatioThreshold; // 0.7f
```

- 修补路径：保留两端都不脏的旧对 → 用空间哈希只查脏体的邻居 → 排序去重 → 覆盖；
- 重建路径：`FullRebuildPersistentNeighborTopology` 清空后重新撒格子、生成全部邻居对。

注意：**脏比例阈值只是成本选择，不是正确性依据**。正确性由「guard 包含校验 + 证书」保证（Part D），即使阈值判断失误，也会在证书校验处被发现并回退。

---

## C9. 生命周期状态机（Lifecycle）

**① 是什么**

每一对候选接触都有一个跨帧状态，描述它们的关系演进：

```
Dormant（休眠）→ Approaching（接近）→ Predictive（预测）→ Actual（实际接触）
                ↘ Expired（过期）  （任何状态都可能直接过期）
```

- Dormant：将来可能接触，但现在还远；
- Approaching：正在接近，还没碰到；
- Predictive：起终点都分离，但轨迹会穿过接触半径（擦肩而过），用稳定法线提前约束；
- Actual：当前已经重叠；
- Expired：超出保留余量，从活跃池移除。

**② 为什么需要**

状态机让系统记住每对关系的「历史」，从而：

1. 只对状态可能变化的对做重分类（MotionVersion 没变就可以直接复用分类结果）；
2. Dormant 对不用每帧参与求解，可以按计划在特定子步唤醒（C10）；
3. 状态信息能区分「预测接触」和「实际穿透」，给求解器不同的约束语义。

**③ 在项目里怎么落地**

`ClassifyPersistentNeighborPair` 用几何量（起点距离、终点距离、最近距离）判定状态并写入 `PersistentPredictiveContact`。分类器有明确的复用条件：

```csharp
bool canReuse = !dirtyEndpoint && hasPrevious &&
                previous.ClassificationEpoch == 当前epoch &&
                previous.MotionVersionA == proxyA.MotionVersion &&
                previous.MotionVersionB == proxyB.MotionVersion;
```

只要两个端点本帧都没脏、版本没变，直接沿用上一帧的分类结果——这是「跳过分类」这一核心省时的入口。

---

## C10. Dormant 调度与子步唤醒（Substep Wakeup）

**① 是什么**

- **Substep（子步）**：一帧（timestep）被分成多个更小的时间片（如 4 substeps × 4 iterations），每个子步做一次完整的「预测 → 避让 → 求解」，提高碰撞连续性；
- **Dormant 调度**：给每个 Dormant 对计算「最早可能发生接触的子步」，写进一张按子步排序的调度表；每个子步只检查「该唤醒的条目」，而不是每帧检查所有 Dormant 对。

**② 为什么需要**

Dormant 对可能占候选池的大头（远处未接触的对）。如果每个子步都精确计算它们的最近距离，等于把省下来的分类成本又花回去。调度让「还没到时间」的对完全不参与计算。

**③ 在项目里怎么落地**

`PredictiveContactScheduler.BuildTimestepSchedule`：Dormant 对的唤醒子步 = 最近接近时间对应的子步再提前 1 格（留安全余量）；无相对运动的对写 `ushort.MaxValue` 哨兵（本 timestep 不唤醒）。`ActivateScheduledPredictiveContactsForSubstep` 每子步推进游标：到期的重新精确判断，真接触 → 激活加入 `TimestepContactPairs`，没到 → 顺延下一子步，不会接触 → 标记 Expired。

---

## C11. 接触集缓存（Timestep Contact Set Cache）

**① 是什么**

把「一个 timestep 内多个 substep 的接触检测结果」缓存复用：首个子步完成完整分类后，后续子步直接使用同一份 `TimestepContactPairs` 视图。

**② 为什么需要**

如果每个子步都重跑 Swept Disc BroadPhase + 分类，子步数 N 就带来 N 倍检测开销（基准里全量候选扫描占 24.94ms）。但子步之间时间很短，接触拓扑变化通常很小——缓存首个子步的结果，只在单位轨迹跑出 Guard 包络时（逃逸）才增量修复，是性价比最高的优化。基准显示仅此一项就让整体求解管线 -62.7%。

**③ 在项目里怎么落地**

- `EnableTimestepContactSetCache` 开启后，证书覆盖 `[0, SubstepCount)` 整个范围；
- 每个子步先做 `ValidateBaseMotionInteractionEnvelope`：旧包络能包住新轨迹就继续用缓存视图；
- 逃逸的 body 走 `RepairOrRebuildContactViewForRemainingTime`：优先只重分类脏体的 incident 对，失败才全量重建（`BuildOrRefreshTimestepContactViews`）。

---

## C12. 跨帧持久缓存（Persistent Contact Cache）

**① 是什么**

在「timestep 内复用」之上再进一步：**跨 timestep** 保留候选拓扑、生命周期和 Dormant 调度，让下一帧直接接着上一帧的状态继续。

**② 为什么需要**

Timestep 缓存只省了「子步间的重复」，但每帧开头仍然要从零扫描找候选对。跨帧持久缓存把「扫描 + 分类 + 状态建立」的成本也摊到很多帧上——一帧只修补变化，是这套增量体系的完整形态。它要求 timestep 缓存同时开启（否则没有可跨帧的视图语义）。

**③ 在项目里怎么落地**

`InteractionCandidateStore` 的全部容器就是这份缓存的物理载体；`IncrementalContactCacheState` 保存有效性、版本、配置指纹（`GuardMargin`、`PredictiveSkin`、子步数等）。每帧开头 `IsPersistentCacheStructurallyReusableP1P6` 检查所有配置轴是否与上次构建一致，任何漂移直接全量重建——配置变了绝不硬套旧缓存。

---

## C13. 排序 + 二分 vs O(1) 哈希

**① 是什么**

- 有序列表 + 二分查找：数据按 key 排序，查找复杂度 O(log N)；
- 哈希表：key 经过哈希函数直接定位槽位，查找复杂度平均 O(1)。

**② 为什么需要**

每帧要对候选池里大量对做「按实体对查持久接触」的操作。O(log N) 在 N = 几千时是十几次比较，看似不多，但出现在每帧几十万次的循环里就变成实打实的耗时；哈希表用空间换时间，把热路径变成常数级。

**③ 在项目里怎么落地**

`PersistentContactIndex` 是 `NativeHashMap<StableEntityPairKey, PersistentPredictiveContact>`：

- 全量重建后从 `PersistentPredictiveContacts` 列表整体回填（一次 O(N)）；
- 增量 patch 路径直接插入/删除单条，不需要重排序；
- 列表仍然保留，用于迭代（视图重建、脏过滤），查找走哈希。

`StableEntityPairKey` 用两个实体的 (Index, Version) 构造稳定键（小者在前），保证跨帧身份一致。

---

## C14. Eligibility Filter（资格过滤）

**① 是什么**

增量修补时，不是脏 body 的每个邻居对都需要重分类。资格过滤先筛掉「本帧不可能产生接触」的对，只有通过资格的对才进入重分类。

**② 为什么需要**

持久邻居池是 Guard 扩大后的集合，包含大量远处 Dormant/Expired 对。如果每次修补都重分类整个池，增量就退化成全量。资格过滤让每次修补只碰真正可能接触的对。

**③ 在项目里怎么落地**

`DirtyIncidentPairMapper.TryEvaluateEligibility` 的判定：

1. 该对上次分类非 Expired → 有资格（状态还在演化中）；
2. 或者两端 tight 扫掠 AABB 本帧重叠 → 有资格（Expired 对重新接近了）；
3. 既 Expired 又不再重叠 → 跳过（这正是被省下的分类工作，计入 `ClassificationSkippedCount`）。

这个规则是完备的：真实接触对要么非 Expired、要么仍在重叠，绝不会被误跳过。

---

# Part D 正确性机制

## D1. 证书（InteractionCertificate）

**① 是什么**

证书是一个「数据产品的质检合格证」：认证器把本帧的紧凑视图（接触对、软避让对、调度表）检查完毕并提交后，签发一份 `InteractionCertificate`，声明「这份视图在以下作用域内是可信的」。

可以类比：视图是一份财务报表，证书是审计师的签字。下游只认「有签字的报表」，不认「审计草稿」。

**② 为什么需要**

增量缓存的正确性依赖大量前提（实体没变、配置没变、包络没漏、分类没过期……）。如果每个下游阶段都自己去验证这些前提，逻辑会散落各处且容易漏。证书把验证集中到唯一边界：**认证器是持久状态唯一写者，证书是下游唯一入口**。前提不满足就不签发，下游 fail-closed（D3），错误不可能悄悄传播。

**③ 在项目里怎么落地**

`IssueInteractionCertificate` 签发前必须凑齐 6 个验证位 + 最后 `Issued` 位：

1. `StructureVerified`：所有视图容器存在、pair 下标合法；
2. `EntityMappingVerified`：实体 → body 下标映射一一对应；
3. `ConfigurationVerified`：配置指纹匹配（所有参数哈希）；
4. `TopologyCoverageVerified`：持久拓扑已覆盖全部可能接触；
5. `ClassificationVerified`：分类有效（分类 epoch 一致）；
6. `ConsumerViewsCommitted`：消费者视图已提交。

证书内容还包括 `WorldId`、`SimulationStepId`、body 集指纹、起止子步、各视图计数。求解/避让阶段读 `certificate.Covers(worldId, stepId, substep)` 检查自己所在子步是否在证书范围内。

---

## D2. 作用域（Scope）与指纹（Fingerprint）

**① 是什么**

- **作用域**：证书有效的时间和身份范围——哪个 World、哪个模拟步、哪一段子步；
- **指纹**：把一组配置/数据压成一个哈希值，用于快速判断「是否和上次一样」。`BodySetFingerprint` 遍历所有实体的 (Index, Version) 累加哈希；`ConfigurationFingerprint` 把所有求解参数哈希。

**② 为什么需要**

证书不能是「永久有效」的：body 换了、配置改了、子步变了，旧证书立刻失效。指纹提供 O(1) 级别的「是否一致」判断——不一致就吊销证书，而不是逐项重查。

**③ 在项目里怎么落地**

- 每帧 `InteractionCertificationJob.CalculateBodySetFingerprint()` 重算一次 body 集指纹；
- `ContactPipelineConfiguration.CalculateCertificationFingerprint()` 把 `DeltaTime`、子步数、迭代数、Compliance、PredictiveSkin、GuardMargin、软避让参数、求解模式等全部混入；
- `GetConsumerCertificateFailure` 里一旦发现指纹不匹配，返回明确的 `BodySetMismatch` / `ConfigurationMismatch`，并吊销证书。

---

## D3. Fail-Closed（失败即关闭）

**① 是什么**

安全设计原则：系统无法证明「没问题」时，按「有问题」处理，而不是按「可能没事」继续。对应的反义词是 fail-open（失败放行）。

**② 为什么需要**

接触求解一旦用了不完整/过期的接触集，单位会互相穿透或漏约束，而且很难事后察觉。fail-closed 保证：证书不完整时直接跳过该子步求解、记录原因、进入恢复路径——宁可不求解，也不能用脏数据求解。

**③ 在项目里怎么落地**

`ContactSolverSkipReason` 有二十多种原因值（证书未签发、作用域不匹配、结构未验证、计数不一致……）。`ValidateConsumerViewsSerial/P1P6` 是两道闸门：

- 软避让前查一次；
- 求解器前查一次；

失败时：记录 `SolverSkipReason` → `RevokeInteractionCertificate` → 置 `RecoveryRequired = 1` → 后续进入恢复求解（只用已认证的视图重新解一次），而不是继续迭代。

---

## D4. 包络逃逸（Envelope Escape）

**① 是什么**

证书证明的范围是「已认证的包络」。任何原因导致单位当前位置/轨迹可能超出这个包络，就叫「逃逸」——证书的证明前提被打破，必须吊销并修复。

**② 为什么需要**

包络是「候选集完备性」的证明边界：只要所有单位都在自己的包络里，上一帧的候选集就能覆盖这一帧所有可能的接触。一旦有人逃逸，候选集可能漏掉它的新邻居，所以必须把逃逸者标脏、重查它的邻居、重新签发覆盖剩余子步的证书。

**③ 在项目里怎么落地**

四个逃逸检查点，覆盖所有可能改变位置/速度的来源：

1. `ValidateBaseMotionInteractionEnvelope`：软避让前的基准运动；
2. `ClampSoftOutputToInteractionEnvelope`：软避让输出后（D5）；
3. `ValidatePredictedContactEnvelope`：未约束预测积分后；
4. `ValidateSolverCorrectionContactEnvelope`：墙/接触投影后（墙后和接触后各一次）。

每次逃逸：吊销证书 → 记录 violation（哪个 body、哪个子步、什么原因、观察到的边界）→ 标脏（`Motion | CorrectedEscape`）→ 进入 `RepairOrRebuildContactViewForRemainingTime` 修复，修复完成重发证书。

---

## D5. 软输出钳位（二分搜索缩放）

**① 是什么**

软避让算出的修正速度可能把单位推出已认证包络。钳位就是「在 0 到 1 之间找一个最大缩放系数」，让「基速度 + 缩放后的避让速度」仍然停留在包络内。找法用二分搜索：8 次迭代把区间从 1 缩小到 1/256 精度。

**② 为什么需要**

软避让是「建议性」修正，不是硬约束；如果完全不限制，它可能把单位推离候选覆盖范围，导致漏检测。限制后既保留避让效果（尽可能接近原始输出），又守住证书边界。

**③ 在项目里怎么落地**

`ClampSoftOutputToInteractionEnvelope` 对每个逃逸 body：

1. 模拟「基速度 + 避让 × scale」后的终点是否在 `InteractionEnvelope` 内；
2. scale=0 都在外 → 直接归零（放弃避让，保正确性）；
3. scale=1 在内 → 原样保留；
4. 否则二分搜索最大安全 scale。

统计里对应 `InteractionEnvelopeEscapeCount`，用于诊断避让压力。

---

## D6. Oracle 验证（O(N²) 参考实现）

**① 是什么**

Oracle（预言机）是一个「正确但昂贵」的参考实现：不依赖任何增量技巧，对每对 body 做精确 swept-disc 测试，生成 ground truth 接触集，然后和增量管线的输出逐对对比。

**② 为什么需要**

增量优化最大的风险是「为了快而漏掉真实接触」（False Negative）。O(N²) Oracle 提供独立的正确性基准，把「我认为没漏」变成「程序验证没漏」——诊断模式下漏报计数必须为零。

**③ 在项目里怎么落地**

`ValidateIncrementalContactSetAgainstQuadraticOracle`（仅 `RTS_CONTACT_DIAGNOSTICS` 编译）：

- 全量两两扫掠测试生成 `IncrementalOracleContactPairs`；
- 统计 `OracleMissingPairCount`（增量漏掉的）和 `OracleExtraPairCount`（增量多报的）；
- 漏报 > 0 → 置 `OracleMismatch = 1`，在验证模式下视为失败。

纪律：Oracle 只读不写，不参与修复决策、不修改任何缓存或证书——它被 CI 脚本强制检查。

---

## D7. 确定性（Determinism）

**① 是什么**

相同输入 + 相同代码 → 完全相同输出，不依赖线程调度、浮点顺序、哈希随机性。这是模拟类游戏和联网对战的基础要求。

**② 为什么需要**

本项目的消费方有三个，都需要确定性：

1. **并行求解**：多线程下如果修正顺序不定，结果会随核数抖动；
2. **回放**：事件溯源回放（F8）要求同一指令序列重放得到同一结果；
3. **网络预测**：客户端预测和服务端权威计算同一逻辑，结果不一致会造成回滚。

**③ 在项目里怎么落地**

- 所有 pair 列表、邻居对、调度表写完后按稳定比较器排序去重，消费顺序唯一；
- Jacobi 修正用 CSR 前缀和收集，每个 body 的求和顺序固定（见 E8）；
- 退化法线用 `DeterministicFallbackNormal(bodyA, bodyB)`：按 body 索引哈希固定选 +X 或 +Z，不依赖浮点瞬时值；
- 版本语义：`MotionVersion` 按轨迹字段逐位相等推进（不用 32-bit hash 当正确性依据）；`TopologyEpoch` / `ClassificationEpoch` 单调递增；
- 分类不读上一帧接触状态，A0（无持久）与 A1（持久）输出同一中层 InteractionSet，差异只在来源成本。

---

## D8. 不变式与层所有权（Invariants / Layer Ownership）

**① 是什么**

- **不变式**：整个系统任何时刻都必须成立的性质，比如「持久候选只被认证器修改」「下游只消费证书内的视图」；
- **层所有权**：每种数据/能力有唯一负责人，别的层不能越权访问。

**② 为什么需要**

当系统复杂度高到「一个人记不住所有细节」时，不变式和所有权把正确性变成结构约束：只要分层正确，某些 bug 在结构上就不可能发生（而不是靠人肉小心）。

**③ 在项目里怎么落地**

项目架构文档里列出 12 条不变式，例如：

1. 所有可能交互都在已认证视图内，或已被标记待修复；
2. 持久候选绝不是 SoftAvoidance / Motion / Solver 的直接输入；
3. dirty ratio 只选「修复还是重建」，不参与正确性证明；
4. Oracle 可观察但不可修改任何状态；
5. 诊断关闭时零额外容器/Job/Profiler。

代码结构上：`BaseFlowMovementSystem` 只做组合根（F1）；四个阶段是独立 `IJob`，各自只带自己的容器；CI 脚本（F7）把这些约束变成静态检查。

---

# Part E 求解器

## E1. 软避让（Soft Avoidance）

**① 是什么**

软避让是「在碰撞发生之前」的速度层修正：当两个单位进入彼此的 soft shell（一个比接触半径更大的缓冲区）时，给双方各加一个推离速度，让它们提前错开。它不直接改位置，改的是速度。

可以类比：硬约束是「撞上了再推开」，软避让是「还没撞就先减速让路」。

**② 为什么需要**

纯硬约束（XPBD）只在穿透发生时起作用，高密度人群会频繁挤撞、抖动。软避让在穿透前就化解大部分冲突，让群体运动更平滑，也减少硬约束的求解压力。

**③ 在项目里怎么落地**

邻居对来自认证过的 `SoftAvoidancePairs` 视图（不是自己现查）。`SoftAvoidanceJob` 在每个子步：清零避让速度 → 累加墙体避让（E9）→ 对每个候选对调用 `SoftAvoidanceMath.TryCalculatePairVelocities`（E2/E3）→ 平均邻居贡献 → 限幅到 `MoveSpeed` → 交给运动积分。输出还会被证书包络检查（D5）钳位。

---

## E2. Surface Velocity Buffer（表面速度缓冲）

**① 是什么**

一种简单的避让算法：两个圆距离越近，彼此受到的推离速度越强，呈线性衰减：

```
surfaceGap = 当前距离 - 半径A - 半径B
softFactor = saturate((softShell - surfaceGap) / softShell)   // 越近越大，0~1
correction = 指向对方反方向 × softFactor × moveSpeed
```

**② 为什么需要**

它是 O(1) 每对的廉价启发式，不需要预测未来，实现简单、稳定、确定性好。在密集编队中「大家互相让一点」就能显著减少硬穿透。

**③ 在项目里怎么落地**

`SoftAvoidanceMath.CalculateUnitVelocity` 实现上述公式；`AccumulateUnitAvoidanceVelocities` 对每个候选对累加双向修正，最后每个 body 除以邻居数取平均（避免多个邻居的修正叠加过大）。它是默认的 `SoftAvoidanceVelocitySolverMode.SurfaceVelocityBuffer`。

---

## E3. 速度障碍（Velocity Obstacle / RVO）

**① 是什么**

速度障碍是另一种避让思路：预测「如果双方保持当前速度，会不会在时间视域（timeHorizon）内相撞」。如果会，就计算一个修正速度把相对速度移出碰撞锥。

项目的实现是简化版（代码注释明确说明不是完整 ORCA/RVO2 线性规划）：

```
相对位移 r = posA - posB
相对速度 v = velA - velB
closestTime = clamp(-dot(r, v) / |v|², 0, timeHorizon)   // 视域内最近时刻
closestDelta = r + v * closestTime
若 |closestDelta| < radiusSum + softShell：
    correction = 法线 × (安全距离 - 最近距离) / 修正时间
    按逆质量比分配给双方
```

**② 为什么需要**

相比表面速度缓冲只看当前距离，速度障碍利用速度预测，对「双方相向而行但当前还远」的情况反应更早，适合开阔地带的高速迎面移动。

**③ 在项目里怎么落地**

`SoftAvoidanceMath.TryCalculateRvoVelocities`；只有 RVO 模式下，proxy 的包络才会外推 `RvoTimeHorizon` 视域终点（`AvoidanceHorizonEnd`），保证「预测到的未来位置」也在证书覆盖范围内——这是 RVO 与证书机制衔接的关键细节。

---

## E4. 响应率与指数衰减

**① 是什么**

- **响应率（Response Rate）**：避让速度叠加到实际速度上的强度。项目用指数公式把「每秒响应率」换算成与帧率无关的比例：

```
alpha = 1 - exp(-responseRate × dt)
```

- **指数衰减**：settled（已到达）单位的速度每帧乘以 `0.8^(dt*60)`，让单位以指数曲线安静下来。

**② 为什么需要**

如果直接用「避让速度 × dt」，结果依赖帧率/子步数，不同帧率下表现不同。指数公式让「每秒衰减多少」恒定，任意子步数下行为一致——对确定性和网络预测都重要。

**③ 在项目里怎么落地**

`SoftAvoidanceMath.CalculateBufferAlpha` / `ApplyVelocityBuffer`；`MotionIntegrationJob` 在积分前应用，settled 单位响应率乘 `SettledSoftAvoidanceMultiplier`（通常缩小），并叠加指数阻尼。

---

## E5. XPBD 约束（Extended Position Based Dynamics）

**① 是什么**

XPBD 是一种「位置级」的物理求解方法：不直接解加速度/力，而是把不满足的约束（如「两圆距离 ≥ 半径和」）直接投影到位置修正上，迭代若干次逼近满足。

核心公式（项目的 `XpbdContactConstraintMath.Evaluate`）：

```
alpha      = Compliance / dt²            // 柔度转刚度倒数
denominator= invMassA + invMassB + alpha
deltaLambda= -(C + alpha × lambda) / denominator
lambda     = max(0, lambda + deltaLambda)   // 单向接触：只推不拉
修正A      = normal × invMassA × appliedLambda
修正B      = -normal × invMassB × appliedLambda
```

其中 C 是约束违反量：Regular 模式 `C = distance - radiusSum`；Predictive 模式 `C = dot(delta, stableNormal) - radiusSum`。

**② 为什么需要**

相比传统力法物理（要求积分稳定、调质量刚度），XPBD 直观、稳定、可迭代、天然适合并行，并且可以直接控制柔度（穿透允许量）。对「圆盘单位互不穿透」这种简单约束，XPBD 是性价比极高的选择。

**③ 在项目里怎么落地**

每个子步开头把 `TimestepContactPairs` 的 lambda 清零；每个迭代对每个 pair 计算 `deltaLambda` → 更新 lambda（夹到 ≥0）→ 按逆质量把位置修正分给两端；最后 `ReconstructVelocity` 用「位置差 / dt」反推速度。`Compliance`、`IterationCount`、`SubstepCount` 都是 `UnitContactSolverSettings` 里可调的参数。

---

## E6. Compliance（柔度）与 Lambda

**① 是什么**

- **Compliance**：约束的「柔度」，相当于弹簧的软硬程度。越大越软（允许更多穿透），0 是刚硬；
- **Lambda**：约束的「累积力度」，表示这个约束到目前为止已经推了多少。它是迭代间传递的记忆，让多次迭代逐渐收敛，而不是每次从头算。

**② 为什么需要**

完全刚硬的约束在少迭代时会有抖动；柔度给系统「让步空间」，让单位在拥挤时有一点弹性而不是绝对硬碰。Lambda 记忆让迭代求解能收敛到稳定解。

**③ 在项目里怎么落地**

`alpha = Compliance / (substepDeltaTime²)` 每子步重算；`ContactConstraint.Lambda` 在子步开始时清零、迭代中累积、子步结束丢弃（不跨帧持久化——架构文档明确说 lambda 是帧内状态，避免把求解历史混进跨帧候选）。`ActivatedSubstepCount`、`WasActivated` 等字段用来统计实际生效的约束。

---

## E7. Gauss-Seidel 与 Jacobi

**① 是什么**

两种迭代求解顺序：

- **Gauss-Seidel（GS）**：逐个处理约束，**算完一个立刻把修正写回位置**，后面的约束马上能看到前面的结果。收敛快，但顺序依赖，难以并行；
- **Jacobi**：一轮里所有约束**从同一份位置快照**独立计算修正，全部算完后再统一应用。可并行，但收敛略慢。

**② 为什么需要**

GS 是串行参考实现：正确性好、收敛快，用于验证和低压力场景。Jacobi 是为了多核并行：5,000 单位 × 数百接触对，逐对串行会吃掉整个 CPU；Jacobi 的「评估」和「应用」两阶段都可以并行。

**③ 在项目里怎么落地**

- `ContactPositionSolverMode.GaussSeidel`：`SolveGaussSeidelContactIteration` 逐 pair 立即写回；
- `ContactPositionSolverMode.Jacobi`：`SolveJacobiContactIteration` 先把每对修正写进 `JacobiPairCorrections`，再按 body 收集 incident 修正取平均应用（E8）。

两条路径共用同一证书提交边界和验证闸门，切换求解模式不影响正确性语义。

---

## E8. CSR Incident Index（压缩稀疏行关联索引）

**① 是什么**

CSR（Compressed Sparse Row）是稀疏矩阵的经典存储：用三个数组（offsets、indices、values）表示「每行有哪些非零列」。项目里借用它表达「每个 body 参与哪些接触对」：

- `ActiveIncidentOffsets[body]` 和 `ActiveIncidentOffsets[body+1]` 给出该 body 的对列表区间；
- `ActiveIncidentPairIndices[offset..offset+count]` 是该 body 的接触对下标。

**② 为什么需要**

Jacobi 的「应用修正」阶段，每个 body 需要把「所有涉及自己的 pair 的修正」累加起来。如果每个 body 都全表扫描 pair 列表，就是 O(P × B)；CSR 让每个 body 只访问自己的 incident 列表，且访问顺序确定，做到「无原子操作 + 确定性求和」。

**③ 在项目里怎么落地**

- 认证阶段（`ActiveConstraintIncidentIndexCertification`）构建前缀和：先统计每个 body 的 incident 数，再做前缀和得到 offsets，最后 scatter 写入 pair 下标；
- 求解阶段 `GatherAndApplyParallelJacobiBodiesJob`（并行按 body）读取自己的区间，把 `JacobiPairCorrections` 里的修正求和、取平均、应用。

另外软避让也有对应的 `SoftIncidentOffsets` / `SoftIncidentPairIndices`，机制相同。

---

## E9. 墙壁约束（软墙 / 硬墙）

**① 是什么**

墙壁有两层处理：

- **软墙（速度层）**：body 靠近障碍格中心时，叠加一个随距离增大的斥力速度；
- **硬墙（位置层）**：body 离障碍格中心小于 `CellRadius + Radius` 时，直接把位置推出到边界。

**② 为什么需要**

流场会避开障碍，但单位有速度惯性、受挤压时会横向漂移进墙。软墙提前推开大部分单位，硬墙兜底保证绝不穿墙。

**③ 在项目里怎么落地**

- 软墙：`SoftAvoidanceMath.CalculateWallVelocity`，斥力强度 `(wallCheckRadius - distance) / distance × 10`，在 `AccumulateWallAvoidanceVelocity` 中检查 3×3 邻格；
- 硬墙：`WallConstraintSolver.SolveWallConstraintIteration`，每次迭代前扫 3×3 邻格，修正量 `(hardDistance - distance) × 0.5`，累计进 `WallCorrection`（参与速度重建和硬碰撞判定）。

语义上，墙使用 `GridObstacleView`（只问「是否阻挡」），导航使用 `FlowNavigationView`（只问「往哪走」），两者共享存储但语义隔离。

---

## E10. 速度重建（Velocity Reconstruction）

**① 是什么**

求解器改的是位置（XPBD 投影），但 ECS 里的单位有 `Velocity` 组件。速度重建就是把「位置差」转回「速度」：

```
IntegratedVelocity = (SolvedPosition - PreviousSubstepPosition) / substepDeltaTime
```

**② 为什么需要**

保持「位置」和「速度」两个视图一致：渲染用位置，移动意图和网络预测用速度。如果求解后不重建速度，下一帧的基准速度还是旧的，修正会被下一帧积分抹掉。

**③ 在项目里怎么落地**

`MotionIntegrationJob.ReconstructVelocities` 每子步末执行；随后 `BuildCrowdBodyResultsJob` 判断「settled 且速度很小」就归零、「硬碰撞中」就保留位置修正，最后 `ApplyFlowMovementJob` 把 `Position / Rotation / Velocity` 写回 ECS 组件。

---

# Part F 工程架构

## F1. 组合根（Composition Root）

**① 是什么**

组合根是「知道所有部件并且把它们拼装起来」的唯一地点。在依赖注入领域指「全应用只有一处做对象装配」；在 ECS 项目里指「唯一知道阶段顺序、资源生命周期和 Job 依赖关系的 System」。

**② 为什么需要**

如果每个阶段自己决定执行顺序和资源归属，会出现循环依赖、重复创建、生命周期错乱。组合根把「编排」集中到一个地方，其余阶段只做「自己的那一步」，像导演和演员的关系。

**③ 在项目里怎么落地**

`BaseFlowMovementSystem` 是明确的组合根：

- 它创建每帧资源（body 数组、认证/软避让/求解资源）；
- 它按顺序调度每个阶段 Job 并串联 JobHandle；
- 它选择串行（`ScheduleSerial`）或并行（`ScheduleParallelJacobiP1P6`）路径；
- 它在帧末统一 Dispose 并注册网格读者；
- 它**不实现**任何分类、证书、求解、诊断算法。

架构文档要求 `CrowdContactPipelineScheduler` 只是「托管调度组合」，本身不是 Job、不实现算法。

---

## F2. 分层架构与依赖方向

**① 是什么**

项目把接触管线分成明确的层：

```
Contracts（数据契约）
  → State（Persistent 候选 / Frame 帧资源）
  → Kernels（纯算法）
  → Stages（认证 / 软避让 / 运动 / 求解）
  → Scheduling（调度接线）
  → Observability（观测，只读）
```

依赖方向只能从上往下（或按箭头），低层不能反向依赖高层。

**② 为什么需要**

层间依赖清晰后：单个 Job 可以独立审查、独立测试；替换实现（如换空间索引）不影响下游；CI 可以用静态脚本检查「谁碰了谁」。

**③ 在项目里怎么落地**

例如 `Kernels/` 下的函数全部是纯静态、显式传参（`PersistentProxyBuilder.BuildFromState(...)`），可以被单独单元测试；`Stages/` 的 Job 只接收自己的容器；`Observability/Contracts` 只是数据 ABI，不参与正确性。`validate_contact_architecture.py` 检查这些目录必须存在、阶段 ABI 必须独立、历史 `Jobs/ContactPipeline` 布局禁止回归。

---

## F3. 调度器只做接线（Scheduling Composition）

**① 是什么**

调度器（`CrowdContactPipelineScheduler`）是一个结构体，保存各阶段 Job 的实例和配置，提供 `ScheduleSerial` / `ScheduleParallelJacobiP1P6` 方法——方法体里只有「创建下一个 Job、设置 Operation、调度并传递 JobHandle」。

**② 为什么需要**

这是 Collections Safety 的要求：每个 Job 必须直接声明自己的 NativeContainer 字段，调度器才能被安全检查看到真实边界。如果调度器自己也是 Job 或者把阶段逻辑内联，容器边界会被隐藏，安全检查失效。

**③ 在项目里怎么落地**

`ScheduleSerial` 里就是一段线性接线：

```csharp
handle = SerialLifecycle.Schedule(handle);
certification.Operation = InitializeSerial; handle = certification.Schedule(handle);
certification.Operation = BuildInitialSerial; handle = certification.Schedule(handle);
for (substep...) { ... }
```

并行路径 `ScheduleParallelJacobiP1P6` 里还插入了大量「count → prefix → scatter」的小 Job（见 E8），它们各自是独立 `IJob`，放在 `Scheduling/Parallel/Jobs/`。

---

## F4. 资源生命周期（Persistent vs Frame）

**① 是什么**

两种生命周期：

- **Persistent（世界级）**：随 World 创建，随 World 销毁，跨帧保留（如 `InteractionCandidateStore`、流场网格）；
- **Frame（帧级）**：每帧创建、帧末释放（如 body 数组、帧内统计），用 `Allocator.TempJob`。

**② 为什么需要**

把「应该跨帧复用的」和「用完即弃的」分开，是内存效率和正确性的关键：跨帧数据必须小心管理版本和释放时机；帧数据则避免每帧 GC 和长期占内存。

**③ 在项目里怎么落地**

- 世界级：`InteractionCandidateStore` 在 `BaseFlowMovementSystem.OnCreate` 创建、`OnDestroy` 释放；`FlowFieldGrid` 由 BakeSystem 创建/释放；
- 帧级：`CrowdStepBodyResources.Create(unitCount)` 用 `Allocator.TempJob` 一次分配 7 个数组，帧末 `Dispose(finalReader)` 把所有释放操作挂到最后一个消费者的依赖上，保证「用完了才释放」。

诊断数据（`ContactDiagnosticsFrameResources`）同样是帧级，宏关闭时这些字段根本不创建。

---

## F5. 编译期诊断开关（RTS_CONTACT_DIAGNOSTICS）

**① 是什么**

一个 C# 条件编译符号：定义时编译出全部诊断代码（Oracle、统计、调试面板数据），不定义时这些代码完全不存在于程序集里。

**② 为什么需要**

诊断代码有真实成本（额外的容器、Job、时间戳读取）。正式构建要零开销，调试又要完整观测——用编译期开关而不是运行时开关，可以保证「诊断关闭时零额外 NativeContainer、零额外 Job、零 Profiler 读取」，这是可以静态验证的承诺。

**③ 在项目里怎么落地**

```csharp
#if RTS_CONTACT_DIAGNOSTICS
public NativeReference<PredictiveDiscContactStatistics> Statistics;
public NativeList<ContactPairDiagnostic> PairDiagnostics;
#endif
```

`PredictiveDiscContactStatistics` 在宏关闭时保留同名组件契约但字段变成属性并返回空值；`ContactPipelineConfiguration.EnableDiagnostics` 在宏关闭时是常量 `false` getter——所有诊断判断变成编译期分支，Burst 直接删除。编辑器里通过 `RTS/Diagnostics/Select Build Settings` 菜单切换。

---

## F6. Profiler Marker 与遥测（Telemetry）

**① 是什么**

- **Profiler Marker**：给一段代码起名并计时，供 Profiler 采样（项目里 `RTS.Simulation.Update` / `RTS.Simulation.Total` 两个主线程标记）；
- **遥测**：系统自己记录的运行指标（每帧接触对数量、逃逸数、耗时、各阶段纳秒数），写进统计结构供面板/CSV 展示。

**② 为什么需要**

性能优化需要「先测量再优化」。项目里大量优化决策（比如缓存开关对比）都依赖这些统计：没有遥测，就无法证明 62.7% 的提升。

**③ 在项目里怎么落地**

- Job 内用 `ProfilerUnsafeUtility.Timestamp` 计时（在 worker 线程也能取时间戳），再换算成纳秒累加进 `PredictiveDiscContactStatistics`（如 `SolverNanoseconds`、`IterationNanoseconds`、`PairGenerationNanoseconds`）；
- `SimulationDebuggerPanel`（F8）展示概况/接触分类/增量统计/运行时参数覆盖；F6 开始/停止录制 CSV、F7 重置录制；
- `AdaptiveParameterTuner` 自动跑参数网格搜索，比较 CSV 找最优参数组合。

---

## F7. CI 静态合约（Static Contracts）

**① 是什么**

在 GitHub Actions 里跑 4 条 Python 脚本，每次 push 对源代码做**正则/结构检查**，把架构红线变成「不满足就 merge 失败」。

**② 为什么需要**

代码评审很难抓住「某个人把算法写进了调度器」「某个人在诊断关分支里加了容器」。静态脚本把「不允许发生的事」写成检查项，防止架构腐化回归。

**③ 在项目里怎么落地**

- `validate_contact_architecture.py`：目录/ABI/所有权；
- `validate_contact_diagnostics.py`：诊断关闭零额外 container / job / profiler；
- `validate_contact_pipeline_audit.py`：禁止 aggregate bag、调度器不可实现算法；
- `validate_contact_static_contracts.py`：SimulationStepId 不可从缓存 age 推导、Oracle 不写游戏状态。

它们不替代 Unity 编译/Burst/Collections Safety，只是第一道廉价防线。

---

## F8. 事件溯源（Event Sourcing）与回放

**① 是什么**

事件溯源是「不保存最终状态，只保存导致状态变化的事件（输入指令），需要状态时从事件序列重放」的架构。类比记账：不记「现在有多少钱」，只记每一笔收支，余额随时可以从账本重算。

**② 为什么需要**

RTS 的录像是「调试回放」而不是「视频录像」。如果每帧保存所有单位 Transform，数据量巨大；只保存输入指令（右键、生成），数据量小几个数量级，而且回放本质是「重跑一次模拟」，还能顺便验证确定性。

**③ 在项目里怎么落地**

- 按 `L` 开始录制：`RequestCommandRpcSystem` 把指令（类型、位置、相对时间）写进 `ReplayCommandElement` DynamicBuffer；
- 按 `R` 开始回放：断网 → 销毁所有单位 → 重置流场（保留 Cost 障碍）→ 加 `LocalInstance` 切本地模式 → 每帧按时间戳取出到期指令，用 `EntityCommandBuffer` 批量执行（生成单位、下达移动）；
- 零快照意味着重置 = 「清实体 + 清网格 + 重放指令」，毫秒级完成。

---

## F9. 服务端权威 + 客户端预测（NetCode）

**① 是什么**

网络同步的一种权威模型：

- **服务端权威**：最终游戏状态由服务端计算并裁决；
- **客户端预测**：客户端不等服务端回包，先用本地模拟「猜」结果，让操作零延迟；收到服务端权威状态后，如果预测正确就继续，错误就回滚纠正。

**② 为什么需要**

纯服务端权威（客户端发请求、等服务端算完再显示）有至少一个 RTT 的延迟，RTS 操作会明显发飘。客户端预测把延迟「藏」起来；服务端权威保证所有玩家看到同一结果，防作弊。

**③ 在项目里怎么落地**

- `LocalUnitFlowMovementSystem`（本地模式）与 `NetCodeUnitFlowMovementSystem`（联网模式）继承同一个 `BaseFlowMovementSystem`，预测逻辑完全共享，保证本地和网络行为一致；
- 客户端右键 → 本地立刻执行 `MoveOrder`（预测）+ 通过 RPC 发给服务端；
- 服务端 `MoveOrderReceiveSystem` 收到 `RequestMoveOrderRPC` 后走同一套指令链路，权威结果通过 Ghost 同步（F10）。

---

## F10. Ghost / RPC

**① 是什么**

- **Ghost**：NetCode 里「服务端所有、客户端只读副本」的实体数据同步机制。客户端看到的是服务端状态的延迟副本（可预测的组件允许客户端写预测值）；
- **RPC**：一次性的远程调用消息，适合「命令/事件」（如移动指令、请求入队），不适合持续状态。

**② 为什么需要**

状态（单位位置、血量）需要持续、高频同步，用 Ghost；一次性操作（右键指令、建单位请求）用 RPC 更合适，避免把瞬时事件当状态传输。

**③ 在项目里怎么落地**

- `FlowArrivalState` 标了 `[GhostComponent(PrefabType = GhostPrefabType.AllPredicted)]` 和 `[GhostField]`，进入预测协议；
- `RequestMoveOrderRPC : IRpcCommand` 由客户端发送、服务端 `MoveOrderReceiveSystem` 消费；
- `RtsTeamRequest` 是入队请求 RPC，客户端 `ClientRequestGameEntrySystem` 在连上后发送，服务端据此处理玩家队伍。

---

## F11. 服务定位器（Service Locator）

**① 是什么**

服务定位器是一个全局注册表：各种服务（UI 控制器、命令系统）按类型注册进去，需要的人通过 `GetService<T>()` 取出来，不必知道服务是谁创建的。

**② 为什么需要**

ECS System 之间不方便直接持有彼此引用（System 生命周期由 World 管理）。服务定位器提供跨系统调用的通道，比如输入系统要调用 RPC 发送系统。

**③ 在项目里怎么落地**

- `ISystemService` 用 `ConcurrentDictionary<Type, object>` 存服务；
- `ServiceSystemBase<T>` 让 ECS System 在 OnCreate 自动注册、OnDestroy 自动注销；
- 例子：`UnitMoveInputSystem` 通过 `this.GetService<RequestCommandRpcSystem>()` 拿到 RPC 系统并调用 `SendInputCommand(...)`。

---

# 附：概念关系图（文字版）

```
输入（UnitMoveInputSystem）
  → 事件溯源录制（ReplayCommandElement）
  → 指令（MoveOrder + 选中快照）
  → RtsCommandSystem：编队槽位（Formation Slots）→ UnitMoveDestination
  → FlowFieldBakeSystem：双缓冲 + Cost/Integration/Vector（BFS）
  → BaseFlowMovementSystem（组合根）
      ├─ BuildCrowdMotionIntentJob：移动意图（Steering）
      ├─ 认证器 InteractionCertificationJob
      │    ├─ 持久候选（Proxy/NeighborPairs/Contacts + 索引）
      │    ├─ 脏标记 → 增量修补 or 全量重建
      │    ├─ 生命周期 + Dormant 调度 + Timestep 缓存
      │    ├─ 包络逃逸检测 + 修复
      │    └─ 签发 InteractionCertificate（fail-closed）
      ├─ SoftAvoidanceJob（SurfaceVelocityBuffer / RVO，软墙）
      ├─ MotionIntegrationJob（预测积分）
      ├─ ConstraintSolverJob（XPBD：GS 串行 / Jacobi 并行 + CSR，硬墙）
      └─ BuildCrowdBodyResultsJob → ApplyFlowMovementJob（写回 ECS）
  → 诊断（Oracle 验证 / Profiler / 调试面板 / CSV）
  → 网络（RPC 上送服务端 → Ghost 同步权威结果）
```

---

# 附：速查表（概念一句话版）

| 概念 | 一句话 |
|---|---|
| ECS | 实体是编号，组件是数据，系统是批量逻辑 |
| Job | 可并行任务，用 JobHandle 依赖链保证安全 |
| Burst | 把 Job 编译成原生机器码的编译器 |
| NativeContainer | 非托管内存容器，必须显式释放 |
| 流场 | 为所有格子预计算方向，单位 O(1) 查方向 |
| 双缓冲 | 写一份读一份，写满原子交换 |
| BroadPhase | 廉价粗筛候选对（网格） |
| Swept Disc | 沿轨迹做圆盘最近距离测试，防隧穿 |
| Guard Bounds | 证明候选集完备性的包络证据 |
| 脏标记 | 记录缓存哪部分、因为什么过时 |
| 增量修补 | 只重算脏 body 的邻居，保留干净部分 |
| 生命周期 | Dormant→Approaching→Predictive→Actual→Expired |
| 证书 | 认证器签发，下游只信有证书的视图 |
| Fail-Closed | 证明不了正确就按失败处理 |
| 包络逃逸 | 位置跑出证明范围，吊销证书并修复 |
| Oracle | O(N²) 参考实现，验证增量不漏报 |
| 确定性 | 相同输入永远相同输出（排序+无原子） |
| 软避让 | 碰撞前改速度让路 |
| XPBD | 位置级约束投影，迭代逼近 |
| Compliance | 柔度，越大越软 |
| Jacobi + CSR | 并行评估 + 前缀和收集，无原子求和 |
| 组合根 | 唯一负责编排和资源生命周期的系统 |
| 编译开关 | 诊断代码在正式构建中物理不存在 |
| 事件溯源 | 只存指令，状态从指令重放 |
| 客户端预测 | 客户端先模拟，服务端后裁决 |
