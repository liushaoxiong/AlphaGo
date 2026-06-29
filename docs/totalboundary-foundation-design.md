# TotalBoundary Clone — 基础版 (Foundation / Tier 1) 详细开发设计

> 版本: v1.0 | 日期: 2026-06-29 | 配套文档: `totalboundary-dev-plan-v2.md`
>
> 本文档面向**基础版的可实现详细设计**,目标是用自研算法完整复刻 TotalBoundary 的核心能力:
> **从一组(近似)封闭的几何中提取闭合边界多段线,且具备超越原生 `BOUNDARY`/`BPOLY` 的间隙容差与中段相交处理能力。**

---

## 一、基础版范围 (Scope)

### 1.1 必须交付的核心能力 (In Scope)

| 能力 | 说明 |
|------|------|
| 多类型实体输入 | Line / Arc / Circle / Ellipse / Spline / (LW/2D)Polyline / BlockReference(单层展开) |
| 间隙容差桥接 | 端点距离 ≤ 容差即视为连接,自动闭合近似封闭图形 |
| 中段相交处理 | 实体在中段交叉(X/T 形)时正确打断并参与成环 |
| 闭合环提取 | 提取所有最小闭合面,输出为 `LWPolyline` |
| 圆弧精确保留 | Arc/Circle 段以 **bulge** 表示,不线性化 |
| 样条/椭圆线性化 | 按矢高(sagitta)自适应细分,精度可控 |
| 三个命令 | `TBBOUNDARY`(选择对象)、`TBQUICKPICK`(点选区域)、`TBALL`(全图) |
| 输出属性 | 图层 / 颜色 / 线宽可配置 |
| 进度与中断 | 大图显示进度,支持 Esc 取消 |

### 1.2 明确不在基础版的内容 (Out of Scope → Pro/ProPlus)

- 面积/周长计算与孔洞扣除 → Pro
- Solid Hatch 生成、DXF 导出、批量、INI 持久化 → Pro
- GUI 配置面板、递归多层块展开、性能面板、许可证 → ProPlus

> 基础版仍**检测**岛屿/孔洞层级(供输出排序与方向),但不做面积扣除。

---

## 二、总体数据流与分层

```
┌─────────────────────────── TB.AutoCAD (主线程 / STA) ───────────────────────────┐
│  Command (TBBOUNDARY/QUICKPICK/ALL)                                              │
│        │ 1. 取选择集 / 拾取点                                                      │
│        ▼                                                                          │
│  EntityExtractor ── 读 DBObject → InputCurve[] (2D 平面坐标 + 反变换矩阵)          │
└────────┬─────────────────────────────────────────────────────────────────────────┘
         │ POCO (无 CAD 依赖)
         ▼
┌─────────────────────────── TB.Core (可后台并行) ──────────────────────────────────┐
│  BoundaryExtractor.Extract(InputCurve[], BoundaryOptions, IProgress, CT)          │
│     ① Tessellator      曲线 → 直线边 Edge(带 provenance 回溯)                      │
│     ② SpatialIndex     均匀网格,广相过滤                                          │
│     ③ SegmentIntersector  线段求交并在交点打断                                     │
│     ④ VertexMerger     端点容差聚类(间隙桥接)→ Vertex                              │
│     ⑤ PlanarGraphBuilder  半边(half-edge)结构 + 各顶点出边极角排序                  │
│     ⑥ FaceTracer       面遍历提取所有最小闭合面 → RawLoop                          │
│     ⑦ LoopHierarchy    包含树 + 方向归一(外环 CCW / 内环 CW)                       │
│     ⑧ LoopRebuilder    直线边链 → BoundaryLoop(按 provenance 还原 bulge 圆弧)      │
│     └─> BoundaryResult (BoundaryLoop[])                                            │
└────────┬─────────────────────────────────────────────────────────────────────────┘
         │ POCO
         ▼
┌─────────────────────────── TB.AutoCAD (主线程 / STA) ───────────────────────────┐
│  PolylineWriter ── BoundaryLoop → LWPolyline(bulge),反变换回 WCS,写入事务         │
└───────────────────────────────────────────────────────────────────────────────────┘
```

**关键设计原则:**
1. **所有 CAD 读写在主线程**(STA),纯几何运算在 `TB.Core`(POCO,可并行/可单测)。
2. **统一直线边拓扑 + provenance 回溯**:把所有曲线展平为直线边做拓扑,既让相交/成环只需处理"线段-线段"(健壮、单一代码路径),又能在输出时凭来源把圆弧还原为精确 bulge。

---

## 三、坐标系与工作平面

边界提取在 2D 平面进行。基础版策略:

1. 取**当前 UCS** 作为工作平面;计算 `ucsToWcs` / `wcsToUcs` 变换矩阵。
2. `EntityExtractor` 把每个实体几何先转 WCS,再用 `wcsToUcs` 投影到 UCS 的 XY 平面,丢弃/记录 Z(elevation)。
3. 所有 `TB.Core` 运算在该 2D 平面坐标系内完成。
4. `PolylineWriter` 输出时用 `ucsToWcs` 把 2D 结果反变换回 WCS,并写入原 elevation。

> 基础版假设参与计算的实体**近似共面**。对明显非共面的实体,投影后由几何容差自然处理或在预处理中剔除(记录日志)。

---

## 四、层间接口与数据模型 (TB.Core, 纯 POCO)

### 4.1 输入 DTO

```csharp
namespace TB.Core.Model;

public readonly struct Point2d
{
    public readonly double X, Y;
    public Point2d(double x, double y) { X = x; Y = y; }
    public double DistanceTo(in Point2d p) => Math.Sqrt(Sq(X - p.X) + Sq(Y - p.Y));
    private static double Sq(double v) => v * v;
}

public enum CurveKind { Line, Arc, EllipseArc, Spline }

/// 与 CAD 无关的输入曲线(已投影到工作平面 2D)。
/// 每条曲线有唯一 Id,供输出阶段回溯还原。
public sealed class InputCurve
{
    public int Id { get; init; }
    public CurveKind Kind { get; init; }

    // 通用端点(开口曲线);闭合曲线(Circle/闭合 Ellipse/闭合 Spline)Start==End
    public Point2d Start { get; init; }
    public Point2d End { get; init; }
    public bool IsClosed { get; init; }

    // Arc 专用: 圆心/半径/起止角(逆时针,弧度)
    public Point2d? Center { get; init; }
    public double Radius { get; init; }
    public double StartAngle { get; init; }
    public double EndAngle { get; init; }

    // EllipseArc / Spline: 评估委托(参数 t∈[0,1] → 点),用于自适应线性化
    public Func<double, Point2d>? Evaluate { get; init; }

    // 来源标识(用于诊断/日志,可选)
    public long SourceHandle { get; init; }
}
```

> **实体 → InputCurve 映射(在 TB.AutoCAD 完成):**
> - `Line` → 1 条 Line。
> - `Arc` → 1 条 Arc。
> - `Circle` → 1 条 Arc(IsClosed=true,起止角 0~2π)或拆为两段半弧(便于打断,推荐拆两段)。
> - `Polyline/Polyline2d` → 逐段:直线段→Line,带 bulge 段→Arc。
> - `Ellipse` → EllipseArc(提供 `Evaluate`);圆形特例可退化为 Arc。
> - `Spline` → Spline(提供 `Evaluate`,内部用 `GetClosestPointTo`/`GetSamplePoints` 或 NURBS 评估)。
> - `BlockReference` → 单层展开:对块内实体应用块变换后递归映射(基础版**仅展开一层**,多层留 ProPlus)。

### 4.2 配置项

```csharp
public sealed class BoundaryOptions
{
    public double GapTolerance { get; init; } = 0.001;   // 端点桥接容差(绘图单位)
    public double MaxSagitta   { get; init; } = 0.0;     // 0=按尺寸自适应; >0 固定最大弦高
    public double MaxArcStepDeg { get; init; } = 15.0;   // 线性化最大角步进
    public double GeomEpsilon  { get; init; } = 1e-9;    // 浮点比较基准(按包围盒缩放)
    public bool   IncludeOuterFace { get; init; } = false; // 是否输出最外无界面边界
    public ExtractMode Mode { get; init; } = ExtractMode.AllLoops;
}

public enum ExtractMode { AllLoops, PointPick }
```

### 4.3 输出 DTO

```csharp
public enum LoopSegmentKind { Line, Arc }

public readonly struct LoopSegment   // 多段线的一段
{
    public LoopSegmentKind Kind { get; init; }
    public Point2d End { get; init; }   // 该段终点(起点=上一段终点/环起点)
    public double Bulge { get; init; }  // Arc 段凸度 = tan(includedAngle/4); Line 段=0
}

public sealed class BoundaryLoop
{
    public Point2d StartPoint { get; init; }
    public IReadOnlyList<LoopSegment> Segments { get; init; } = [];
    public bool IsOuter { get; init; }       // true=外环(CCW), false=内孔(CW)
    public int ParentLoopIndex { get; init; } = -1; // 包含树:父环索引
    public double SignedArea { get; init; }  // 朝向判断与排序用(基础版仅辅助,不对外计算)
}

public sealed class BoundaryResult
{
    public IReadOnlyList<BoundaryLoop> Loops { get; init; } = [];
    public IReadOnlyList<string> Diagnostics { get; init; } = [];  // 警告/剔除记录
}
```

### 4.4 门面 (Facade)

```csharp
namespace TB.Core;

public interface IBoundaryExtractor
{
    BoundaryResult Extract(
        IReadOnlyList<InputCurve> curves,
        BoundaryOptions options,
        Point2d? pickPoint,                 // PointPick 模式下的拾取点
        IProgress<ProgressInfo>? progress,
        CancellationToken cancellationToken);
}
```

---

## 五、算法各阶段详细设计

### 5.1 ① Tessellator — 曲线展平为直线边 (带 provenance)

**目的**:把所有曲线统一为直线边 `Edge`,使后续相交/成环只处理线段-线段;同时记录每条边来自哪条曲线、哪段参数,供输出还原。

```csharp
internal readonly struct Edge
{
    public Point2d A { get; init; }
    public Point2d B { get; init; }
    public int CurveId { get; init; }       // 来源 InputCurve.Id
    public double TA { get; init; }         // A 在源曲线上的参数(Arc=角度, 其它=t∈[0,1])
    public double TB { get; init; }
}
```

**展平规则:**
- **Line**:直接 1 条 Edge,`TA=0, TB=1`。
- **Arc / Circle**:按 `step = min(MaxArcStepDeg, 由 MaxSagitta 反算的角步)` 均匀细分;`TA/TB` 记录角度。注意 **provenance 保留圆心/半径**(在 LoopRebuilder 凭 CurveId 取回),故展平的密度只影响相交精度,不影响输出精度。
- **EllipseArc / Spline**:用 `Evaluate(t)` **自适应矢高细分**(见 5.1.1),`TA/TB` 记录 t。
- **闭合曲线**(Circle/闭合样条):展平为首尾相接的边环。

#### 5.1.1 自适应矢高细分

```
SampleAdaptive(eval, t0, t1, maxSagitta):
    p0 = eval(t0); p1 = eval(t1); tm = (t0+t1)/2; pm = eval(tm)
    # 中点到弦 p0p1 的垂距
    if dist(pm, segment(p0,p1)) <= maxSagitta:
        emit Line(p0 -> p1)        # 弦高足够小,接受
    else:
        SampleAdaptive(eval, t0, tm, maxSagitta)
        SampleAdaptive(eval, tm, t1, maxSagitta)
```
- `maxSagitta` 为 0 时按整体包围盒对角线 × 比例(如 1e-4)自适应取值。
- 递归设最大深度,避免病态曲线无限细分。

### 5.2 ② SpatialIndex — 均匀网格广相过滤

```csharp
internal sealed class UniformGrid
{
    // cell 尺寸取 max(平均边长, GapTolerance) 量级
    public void Insert(int edgeIndex, in BBox box);
    public IEnumerable<int> Query(in BBox box);  // 返回候选边索引(可能重复,需去重)
}
```
- 用途:为相交检测(5.3)与端点聚类(5.4)提供 O(1) 邻域查询,避免 O(n²)。
- 基础版用均匀网格即可;Bentley–Ottmann 扫描线作为后续性能优化(在 v2 主方案中列为优化项)。

### 5.3 ③ SegmentIntersector — 线段求交并打断

**目的**:实体中段相交时,在交点处把边打断,使交点成为图的顶点(否则 X/T 交叉无法成环)。

```
for each edge e:
    candidates = grid.Query(bbox(e) 膨胀 eps)
    for each c in candidates where c.index > e.index:   # 避免重复成对
        hits = SegSegIntersect(e, c, eps)               # 0/1/2(共线重叠)个交点
        record split params on e and c
# 按参数排序,把每条边在其所有交点处切成多条子边(继承 CurveId, 线性插值 TA/TB)
```

**`SegSegIntersect`(线段-线段,闭式、健壮):**
- 用 2D 叉积判定方向与平行;
- 端点接触、T 形(交点是某段端点)、共线重叠都需正确处理;
- 容差:交点与端点距离 ≤ `eps` 时吸附到端点,避免产生极短碎边;
- 共线重叠段:记录重叠区间端点为分裂点(基础版可将完全重叠视为重复,在预处理剔除其一)。

> 输出:打断后的子边集合 `Edge[]`,所有真实交叉点都成为某条子边的端点。

### 5.4 ④ VertexMerger — 端点容差聚类(间隙桥接)

**这是"间隙容差"特性的核心。** 把彼此距离 ≤ `GapTolerance` 的端点合并为同一 `Vertex`,从而桥接近似封闭图形的缝隙。

```
grid2 = 网格(cell = GapTolerance)
uf    = UnionFind(所有边端点)
for each endpoint p:
    for q in grid2.NeighborCells(p):       # 仅查相邻 9 格
        if dist(p,q) <= GapTolerance: uf.Union(p,q)
# 每个并查集簇 → 一个 Vertex,坐标取簇代表点(质心或第一个点,保持确定性)
# 每条边的 A/B 改为引用其所属 Vertex 的 Id
```

**注意:**
- 代表点选取须**确定性**(如簇内最小坐标点),避免并行/重排导致结果漂移。
- 合并后可能出现**退化边**(两端点落入同簇)→ 标记为零长边并剔除(进诊断日志)。
- 网格 cell = 容差可保证只需查相邻格,O(n)。

### 5.5 ⑤ PlanarGraphBuilder — 半边结构 + 极角排序

构建用于面遍历的 **DCEL(双向链接边表)** 半边结构:

```csharp
internal sealed class HalfEdge
{
    public int Origin;        // 起点 Vertex Id
    public int Dest;          // 终点 Vertex Id
    public int Twin;          // 反向半边索引
    public int EdgeRef;       // 关联的几何 Edge(含 CurveId/参数)
    public double Angle;      // 从 Origin 出发的方向角 atan2(dy,dx)
    public int Next = -1;     // 同一面内的下一条半边(由 FaceTracer 填)
    public bool Visited;
}
```
- 每条无向边 → 2 条半边(互为 twin)。
- 对每个顶点,收集所有**出边半边**并按 `Angle` 升序排序,存入 `vertexOutgoing[v]`(用于 5.6 选下一条边)。
- 去除重复半边(同一对顶点间的重复几何边在预处理已合并)。

### 5.6 ⑥ FaceTracer — 面遍历提取闭合环

采用平面图面遍历(非 Hierholzer):

```
for each half-edge he not visited:
    loop = []
    cur = he
    repeat:
        mark cur visited
        loop.append(cur)
        # 在 cur.Dest 处,选"紧邻 cur 反向边的下一条(顺时针)出边"
        twinAngle = angle(cur.Twin)                 # 即 Dest→Origin 方向
        outs = vertexOutgoing[cur.Dest]             # 已按角排序
        next = outs 中角度 == twinAngle 的项的"前一个"(环形, 顺时针下一条)
        cur = next
    until cur == he
    record RawLoop(loop)
```

- 该规则遍历出平面细分的**所有面**(含最外无界面)。
- 每个 `RawLoop` 用**有符号面积**(Shoelace)判定朝向:`area>0` 为 CCW,`area<0` 为 CW。
- **最外无界面**:其有符号面积为所有面中绝对值最大且方向相反者,或可由"包含所有其它面"判定;依 `IncludeOuterFace` 决定是否丢弃。
- 仅由开放链(非闭合)构成、无法回到起点的半边不会形成有效面 → 自然排除。

> **正确性要点**:`TBBOUNDARY`/`TBALL` 取所有有界面;每个最小面 = 一条边界多段线。

### 5.7 ⑦ LoopHierarchy — 包含树与方向归一

- 用**点-在-多边形**测试(射线法)判断环之间的包含关系,构建包含树。
- 约定输出方向:**外环 CCW、内孔 CW**(便于 Pro 阶段做面积扣除与 Hatch 岛屿)。
- 基础版**逐环输出为独立 `LWPolyline`**(与 `BOUNDARY` 行为一致),`ParentLoopIndex` 仅作为元数据保留。

### 5.8 ⑧ LoopRebuilder — 还原 bulge 圆弧

把面遍历得到的"直线边链"凭 provenance 还原为精确多段线:

```
for each RawLoop:
    合并连续同源边: 遍历环上相邻 Edge,若 CurveId 相同且该曲线是 Arc/Circle
        且参数(角度)单调连续 → 合并为一段圆弧
    输出 LoopSegment:
        - 源是 Line  → LoopSegment{Line, End=边终点, Bulge=0}
        - 源是 Arc   → LoopSegment{Arc,  End=子弧终点, Bulge=tan(Δangle/4)}
                       (Δangle 由合并后的起止角得到, 子弧也精确)
        - 源是 Ellipse/Spline → 保留为多条 Line 段(已线性化)
```
- **bulge 符号**取决于子弧的转向(CCW 为正)与环的遍历方向,需据实际走向定号。
- 这样 Arc/Circle 在输出中**精确无损**,而 Spline/Ellipse 为受控精度的折线。

---

## 六、TB.AutoCAD 集成层设计

### 6.1 EntityExtractor(主线程)

```csharp
// 在文档锁 + 读事务内执行
public IReadOnlyList<InputCurve> Extract(IEnumerable<ObjectId> ids, Matrix3d wcsToUcs)
{
    var list = new List<InputCurve>();
    int nextId = 0;
    foreach (var id in ids)
    {
        var ent = tr.GetObject(id, OpenMode.ForRead) as Entity;
        switch (ent)
        {
            case Line ln:        list.Add(MapLine(ln, ...)); break;
            case Arc arc:        list.Add(MapArc(arc, ...)); break;
            case Circle c:       list.AddRange(MapCircleAsTwoArcs(c, ...)); break;
            case Polyline pl:    list.AddRange(MapPolyline(pl, ...)); break;  // 直线/bulge 段
            case Ellipse el:     list.Add(MapEllipse(el, ...)); break;
            case Spline sp:      list.Add(MapSpline(sp, ...)); break;
            case BlockReference br: list.AddRange(ExplodeOneLevel(br, ...)); break;
            default: /* 记录"不支持类型"诊断 */ break;
        }
    }
    return list;  // 所有点已 wcsToUcs 投影到工作平面
}
```
- `MapSpline`/`MapEllipse` 的 `Evaluate` 委托基于 `Curve.GetPointAtParameter` / `GetClosestPointTo`,在主线程一次性"取值闭包"或预采样后传给 Core(若要后台并行,需预采样为数组,避免后台触碰 DBObject)。
- **块展开**:`br.BlockTransform` 应用到块内实体几何后映射;基础版仅一层,嵌套块记录诊断。

### 6.2 PolylineWriter(主线程,写事务)

```csharp
foreach (var loop in result.Loops)
{
    var pl = new Polyline();
    int i = 0;
    var pts = LoopToVertices(loop);          // 含 bulge
    foreach (var v in pts)
        pl.AddVertexAt(i++, ToUcs2d(v.Pt), v.Bulge, 0, 0);
    pl.Closed = true;
    pl.Normal = ucsNormal;                   // 设到工作平面
    pl.Elevation = elevation;
    ApplyProperties(pl, options);            // 图层/颜色/线宽
    btr.AppendEntity(pl); tr.AddNewlyCreatedDBObject(pl, true);
}
```
- 输出 `LWPolyline`(`Polyline`),圆弧用 `bulge`;按 UCS 法向与标高写回。
- 自动创建/复用图层 `TB_Boundary`(可配置)。

### 6.3 线程模型

- **基础版默认同步执行于命令主线程**,通过 `IProgress` 回调刷新进度;在循环中调用 `AcEd` 的可中断检测(轮询 `Esc`)。
- `TB.Core` 内的纯几何步骤实现为可被 `Parallel`/PLINQ 加速,但**前提是输入已是 POCO**(Spline/Ellipse 已预采样),不在后台触碰任何 DBObject。
- 写回阶段回到主线程,单一事务批量提交以支持一次 Undo。

---

## 七、三命令交互流程

### 7.1 `TBBOUNDARY` — 选择对象

```
1. PromptSelectionOptions: 允许窗口/交叉/点选对象, 过滤支持的实体类型
2. 锁文档 + 读事务 → EntityExtractor 取 InputCurve[]
3. BoundaryExtractor.Extract(Mode=AllLoops)
4. PolylineWriter 写回所有闭合环
5. 报告: "提取 N 条边界, 桥接 M 处间隙, 剔除 K 个无效实体"
```

### 7.2 `TBQUICKPICK` — 点选区域 (BPOLY 行为)

```
1. PromptPoint: 取内部拾取点(可循环多次)
2. 候选实体: 以拾取点为中心的窗口做 SelectCrossingWindow, 或取当前空间全部(带网格过滤)
3. EntityExtractor → BoundaryExtractor.Extract(Mode=PointPick, pickPoint)
   - Core 内: 构建平面图后做"点定位":
       a. 从 pickPoint 向 +X 射线, 求与所有边交点, 取最近交点所在半边
       b. 选使 pickPoint 落在其左侧的半边方向, FaceTracer 仅追踪该面
   - 若 pickPoint 不被任何闭合面包围 → 报"未找到封闭边界"
4. 写回该面边界; 支持连续点选累积
```

### 7.3 `TBALL` — 全图提取

```
1. 取当前空间(模型/图纸)全部支持实体(无需用户选择)
2. EntityExtractor → Extract(Mode=AllLoops)
3. 大图: 强制显示进度条 + Esc 中断
4. 写回所有闭合环
```

---

## 八、容差与健壮性模型

| 容差 | 默认 | 作用 |
|------|------|------|
| `GapTolerance` | 0.001 | 端点聚类桥接(5.4),用户主调参数 |
| `GeomEpsilon` | 1e-9 ×包围盒尺度 | 相交/共线/叉积判定基准 |
| `MaxSagitta` | 自适应 | Spline/Ellipse 线性化精度 |
| `MaxArcStepDeg` | 15° | 圆弧展平/线性化角步上限 |

**健壮性处理清单(预处理 + 各阶段):**
- 重复实体、零长线段 → 预处理剔除(诊断记录)。
- 共线重叠段 → 合并或剔除其一,避免图退化。
- 自相交曲线 → 求交阶段自然在自交点打断。
- 端点聚类代表点确定性,避免并行漂移。
- 退化边(聚类后零长)剔除。
- 浮点判定统一走 `RobustPredicates`(叉积/朝向/共线),阈值随包围盒缩放。

---

## 九、模块清单与职责(基础版落地范围)

| 工程 | 文件 | 职责 | 可单测 |
|------|------|------|:----:|
| TB.Core | `Model/*.cs` | DTO:Point2d/InputCurve/Edge/HalfEdge/BoundaryLoop | ✅ |
| TB.Core | `Geometry/Tessellator.cs` | 曲线→直线边 + provenance + 自适应矢高 | ✅ |
| TB.Core | `Geometry/UniformGrid.cs` | 均匀网格空间索引 | ✅ |
| TB.Core | `Geometry/SegmentIntersector.cs` | 线段求交并打断 | ✅ |
| TB.Core | `Geometry/VertexMerger.cs` | 端点容差聚类(并查集) | ✅ |
| TB.Core | `Geometry/PlanarGraphBuilder.cs` | 半边结构 + 极角排序 | ✅ |
| TB.Core | `Geometry/FaceTracer.cs` | 面遍历提取环 | ✅ |
| TB.Core | `Geometry/LoopHierarchy.cs` | 包含树 + 方向归一 | ✅ |
| TB.Core | `Geometry/LoopRebuilder.cs` | 还原 bulge 圆弧 | ✅ |
| TB.Core | `Geometry/RobustPredicates.cs` | 叉积/朝向/共线判定 | ✅ |
| TB.Core | `BoundaryExtractor.cs` | 门面,串联流水线 | ✅ |
| TB.AutoCAD | `Services/EntityExtractor.cs` | DBObject→InputCurve(投影/块展开) | 集成 |
| TB.AutoCAD | `Services/PolylineWriter.cs` | BoundaryLoop→LWPolyline | 集成 |
| TB.AutoCAD | `Services/UcsService.cs` | UCS↔WCS 变换 | 集成 |
| TB.AutoCAD | `Services/{Selection,Layer,Progress}Service.cs` | 选择/图层/进度 | 集成 |
| TB.AutoCAD | `Commands/{TBoundary,TBQuickPick,TBAll}.cs` | 命令入口 | 集成 |

---

## 十、开发顺序与验收关卡 (DoD)

| 步骤 | 实现 | 验收样例(可单测/集成) |
|------|------|------|
| 1 | 数据模型 + BoundaryExtractor 空壳 | 项目编译,门面可调用 |
| 2 | Tessellator | 圆/弧/样条展平点数随 sagitta 收敛;Line 不变 |
| 3 | UniformGrid + RobustPredicates | 邻域查询正确;叉积/朝向单测通过 |
| 4 | SegmentIntersector | 1 个十字交叉 → 4 条子边,交点唯一 |
| 5 | VertexMerger | 0.5mm 间隙的矩形 4 边 → 4 顶点(桥接成功) |
| 6 | PlanarGraphBuilder + FaceTracer | 矩形→1 内面;两相邻矩形→3 面(共享边) |
| 7 | LoopHierarchy | 外框+内框 → 2 环,包含关系正确,方向归一 |
| 8 | LoopRebuilder | 含圆弧矩形 → 输出含 bulge,重建几何≈原弧 |
| 9 | EntityExtractor + PolylineWriter | 在 AutoCAD 中矩形/圆/带弧图形提取正确 |
| 10 | 三命令 | TBBOUNDARY/QUICKPICK/ALL 均可用 |
| 11 | 进度/中断 + 诊断 | 1000+ 实体 < 5s,可 Esc;诊断输出合理 |
| 12 | 对比验证 | 干净图与原生 BOUNDARY 一致;带间隙图本插件成功而原生失败 |

---

## 十一、关键测试用例(TB.Core 单测,无需 AutoCAD)

```
[矩形-纯直线]      4 Line 首尾相接           → 1 环, 4 段, 全 Line
[矩形-带间隙]      4 Line 各留 0.5mm 缝       → 容差 1mm: 1 环; 容差 0.1mm: 0 环
[圆]               1 Circle                   → 1 环, 2 段 Arc(bulge=±1)
[圆角矩形]         4 Line + 4 Arc             → 1 环, 8 段(4 Line+4 Arc bulge)
[十字交叉]         2 条交叉 Line              → 中段打断, 不成环(开放)
[井字/四格]        # 形 4 线交叉              → 4 个最小面 + 边界面
[嵌套环]           外矩形 + 内矩形(不连)      → 2 环, 内环 IsOuter=false
[T 形搭接]         一线端点落在另一线中段     → 打断且顶点合并
[样条闭合]         1 闭合 Spline              → 1 环, 多 Line 段(矢高内)
[点选-PointPick]   井字内某格点选            → 仅该格 1 环
[退化]             零长线/重复线             → 剔除, 进诊断, 不崩溃
```

---

## 十二、与原生命令的核心差异验证(基础版即体现)

| 场景 | 原生 BOUNDARY/BPOLY | 基础版预期 |
|------|--------------------|-----------|
| 有 0.5mm 缝隙的"封闭"图 | 报"未找到有效边界" | 容差内自动桥接成环 ✅ |
| 中段交叉但视觉封闭 | 依赖显示精度,易失败 | 显式打断成面 ✅ |
| 含圆弧边界 | 可成功 | 成功且 bulge 无损 ✅ |
| 容差/采样调节 | 不可控 | 全参数化 ✅ |

> 基础版即应在"间隙桥接 + 中段相交"两点上**明显优于**原生命令,这是 TotalBoundary 的核心卖点。

---

> 本设计与 `totalboundary-dev-plan-v2.md` 配套:主方案描述总体架构与分级,本文件给出基础版可直接编码的详细设计(数据模型、算法伪代码、命令流程、验收关卡)。后续可据此生成 TB.Core / TB.AutoCAD 的解决方案骨架代码。
