# TotalBoundary Clone — 基础版 (Foundation / Tier 1) 详细开发设计

> 版本: **v1.1** | 日期: 2026-06-29 | 配套文档: `totalboundary-dev-plan-v2.md`(v2.1)
>
> 本文档面向**基础版的可实现详细设计**,目标是用自研算法**忠实复刻 TotalBoundary 的核心能力**:
> **对用户选定的对象集合,沿其外周界生成闭合多段线(外轮廓),并在内部存在较大空腔时生成孔洞边界;具备超越原生 `BOUNDARY`/`BPOLY` 的间隙容差与中段相交处理能力。**
>
> **v1.1 纠偏要点**:核心目标由"提取所有最小内部面"改正为"**外轮廓 + 孔洞提取**";空腔判定采用**路线 B(形态学闭合 / 闭合半径)**;命令体系改为 `TOTALBOUNDARY`/`TBALL`/`TBPICK`。SuperBoundary 式"逐面全提取"不在范围。

---

## 一、基础版范围 (Scope)

### 1.1 必须交付的核心能力 (In Scope)

| 能力 | 说明 |
|------|------|
| 多类型实体输入 | Line / Arc / Circle / Ellipse / Spline / (LW/2D)Polyline / BlockReference(单层展开) |
| 间隙容差桥接 | 端点距离 ≤ 容差即视为连接,自动闭合近似相接处的缝隙 |
| 中段相交处理 | 实体在中段交叉(X/T 形)时正确打断并参与拓扑 |
| **外轮廓提取** | 沿选定对象集合的外周界生成闭合 `LWPolyline`(可多片段各自外轮廓) |
| **孔洞识别(路线 B)** | 内部空腔 > 闭合半径 → 生成孔洞边界;细小单元被填充覆盖,不输出内部边 |
| 圆弧精确保留 | Arc/Circle 段以 **bulge** 表示,不线性化 |
| 样条/椭圆线性化 | 按矢高(sagitta)自适应细分,精度可控 |
| 无需预清理 | 自动忽略孤立悬挂线(不参与轮廓) |
| 三个命令 | `TOTALBOUNDARY`/`TB`(选对象)、`TBALL`(全图)、`TBPICK`(点选片段轮廓) |
| 输出属性 | 图层 / 颜色 / 线宽可配置 |
| 进度与中断 | 大图显示进度,支持 Esc 取消 |

### 1.2 明确不在基础版的内容 (Out of Scope)

- **SuperBoundary 式"逐个内部面全提取"**(BPOLY 全区域语义)→ 不在 TotalBoundary 复刻范围(可作未来可选模式)
- 面积/周长计算与孔洞扣除 → Pro
- Solid Hatch 生成、DXF 导出、批量、INI 持久化 → Pro(Solid 填充为 TotalBoundary 宣传特性,基础版预留接口)
- GUI 配置面板、递归多层块展开、性能面板、许可证 → ProPlus

> 基础版输出**外轮廓 + 孔洞**;孔洞通过路线 B 的面分类识别,不做面积扣除(面积计算留 Pro)。

---

## 二、总体数据流与分层

```
┌─────────────────────────── TB.AutoCAD (主线程 / STA) ───────────────────────────┐
│  Command (TOTALBOUNDARY / TBALL / TBPICK)                                        │
│        │ 1. 取选择集 / 拾取点                                                      │
│        ▼                                                                          │
│  EntityExtractor ── 读 DBObject → InputCurve[] (2D 平面坐标 + 反变换矩阵)          │
└────────┬─────────────────────────────────────────────────────────────────────────┘
         │ POCO (无 CAD 依赖)
         ▼
┌─────────────────────────── TB.Core (可后台并行) ──────────────────────────────────┐
│  OutlineExtractorService.Extract(InputCurve[], BoundaryOptions, IProgress, CT)    │
│     ① Tessellator      曲线 → 直线边 Edge(带 provenance 回溯)                      │
│     ② SpatialIndex     均匀网格,广相过滤                                          │
│     ③ SegmentIntersector  线段求交并在交点打断                                     │
│     ④ VertexMerger     端点容差聚类(间隙桥接)→ Vertex                              │
│     ⑤ PlanarGraphBuilder  半边(half-edge)结构 + 各顶点出边极角排序                  │
│     ⑥ FaceTracer       面遍历得到所有面(含无界外面)→ Face[]                        │
│     ⑦ FaceClassifier   ★路线 B: 每个有界面按闭合半径分 Solid / Void               │
│     ⑧ OutlineBuilder   ★取"实边且非两侧皆 Solid"的边 → 外轮廓 + 孔洞环            │
│     ⑨ LoopRebuilder    直线边链 → BoundaryLoop(按 provenance 还原 bulge 圆弧)      │
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
3. **核心是"并集外轮廓",非"逐个内部面"**:⑥ 得到所有面后,⑦ 用路线 B 把面分为 Solid/Void,⑧ 仅输出 Solid 与非 Solid 之间(及孤立闭环)的实体边 = 外轮廓 + 孔洞。

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
    public double GapTolerance  { get; init; } = 0.001;  // 端点桥接容差(绘图单位)
    public double ClosingRadius { get; init; } = 0.0;    // 路线 B 闭合半径 r; 0=按 GapTolerance 自适应(≈2–5×)
    public double MaxSagitta    { get; init; } = 0.0;    // 0=按尺寸自适应; >0 固定最大弦高
    public double MaxArcStepDeg { get; init; } = 15.0;   // 线性化最大角步进
    public double GeomEpsilon   { get; init; } = 1e-9;   // 浮点比较基准(按包围盒缩放)
    public ExtractMode Mode { get; init; } = ExtractMode.SelectedObjects;
}

// SelectedObjects: 对所选对象整体出外轮廓+孔洞
// FragmentAtPoint: 仅对拾取点所在的连通线工片段出轮廓(TBPICK)
public enum ExtractMode { SelectedObjects, FragmentAtPoint }
```

> **闭合半径 ClosingRadius (r) 是路线 B 的关键参数**:有界面的最大内切空圆半径 > r 视为孔洞(Void),≤ r 视为被填充(Solid)。r 越大,轮廓越平滑、孔越少;r 越小,越多内部空腔被保留为孔。默认按 `GapTolerance` 自适应放大。

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

public interface IBoundaryExtractor   // 门面; 实现类 OutlineExtractorService
{
    BoundaryResult Extract(
        IReadOnlyList<InputCurve> curves,
        BoundaryOptions options,
        Point2d? pickPoint,                 // FragmentAtPoint 模式下的拾取点
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

### 5.6 ⑥ FaceTracer — 面遍历得到所有面

采用平面图面遍历(DCEL 标准做法,非 Hierholzer),目的是得到**平面细分的所有面**(供 ⑦ 分类),不是直接当作输出环。

```
for each half-edge he not visited:
    face = []
    cur = he
    repeat:
        mark cur visited
        face.append(cur)
        # 在 cur.Dest 处,选"紧邻 cur 反向边的下一条(顺时针)出边"
        twinAngle = angle(cur.Twin)                 # 即 Dest→Origin 方向
        outs = vertexOutgoing[cur.Dest]             # 已按角排序
        next = outs 中角度 == twinAngle 的项的"前一个"(环形, 顺时针下一条)
        cur = next
    until cur == he
    record Face(face)            # 记录构成该面的半边序列
# 为每条无向边登记其左面/右面(faceLeft, faceRight)
```

- 该规则遍历出平面细分的**所有面**(含最外无界面)。
- 每个 `Face` 用**有符号面积**(Shoelace)判定朝向并计算面积;**最外无界面**为有符号面积为负(或绝对值最大方向相反)者。
- 仅由开放链构成、无法回到起点的悬挂边不会形成有效面 → 自然排除(对应"忽略悬挂线")。
- **关键**:同时为每条几何边记录其两侧面(`faceLeft`/`faceRight`),供 ⑧ 轮廓提取判断"两侧是否皆 Solid"。

### 5.7 ⑦ FaceClassifier — 路线 B 面分类 (Solid / Void)

> TotalBoundary 的"外轮廓 + 孔洞"= 对线工做**形态学闭合(半径 r = ClosingRadius)**后的边界。等价地逐面分类:

```
for each face F:
    if F 是无界外面:  F.kind = Void          # 外部恒为 Void
    else:
        rho = MaxInscribedEmptyCircleRadius(F)   # 面内不触及任何边的最大空圆半径
        F.kind = (rho > r) ? Void : Solid        # 大空腔=孔洞; 细小单元=被填充
```

**`MaxInscribedEmptyCircleRadius(F)` 求法(基础版可由简到精逐步实现):**
- **近似法(MVP 首选)**:对面做约束三角剖分或栅格采样,取面内点到其边界的最大距离;或用面积/周长比 + 包围盒做快速近似。
- **精确法(可选优化)**:基于面边界的广义 Voronoi / 中轴,求最大内切圆。
- `r` 默认按 `GapTolerance` 自适应放大(经验 2–5×),用户可调以控制轮廓细节。

**正确性要点(几个判例):**
- **大矩形(4 线)**:内部面 ρ 很大 → Void;外部无界面 → Void。两侧皆 Void 的 4 条实体边在 ⑧ 被保留 → 输出即矩形(不会因"内部是 Void"而丢失,因为 ⑧ 只在"两侧皆 Solid"时丢边)。
- **稠密网格(# 形 N×N)**:每个小格 ρ ≤ r → Solid;内部边两侧皆 Solid → ⑧ 丢弃;仅外周界保留。
- **donut(外环内含大空腔)**:外环带状区的小单元=Solid,中心大空腔 ρ>r=Void → 外轮廓 + 1 孔洞。

### 5.8 ⑧ OutlineBuilder — 轮廓边提取与组装

```
keep = []
for each 几何边 e (仅实体边, 桥接边见下):
    Lf = e.faceLeft.kind;  Rf = e.faceRight.kind
    if not (Lf == Solid and Rf == Solid):   # 两侧皆 Solid 才丢弃(属实心内部)
        keep.add(e)
# 桥接边: 仅当其位于保留轮廓上、用于闭合缝隙时保留(否则丢弃)
# 将 keep 中的边按公共顶点拓扑串联为闭合环:
#   - 一个连通片段的外周界 → 外轮廓环(CCW)
#   - 实心与空腔之间的边 → 孔洞环(CW)
#   - 多个不连通片段 → 各自的外轮廓
assemble keep -> List<RawLoop>(标注 IsOuter / ParentLoopIndex)
```

**要点:**
- 判据"**两侧皆 Solid 才丢弃**"统一覆盖了所有判例(矩形、网格、donut、多片段)。
- 在每个顶点串联时,若度数 > 2(多边交汇),按角度选择保持轮廓走向的下一条边(类似面遍历的 next-edge 规则)。
- 用有符号面积归一方向:外轮廓 CCW、孔洞 CW;`ParentLoopIndex` 标注孔属于哪个外轮廓。

### 5.9 ⑨ LoopRebuilder — 还原 bulge 圆弧

把 ⑧ 得到的"直线边链"凭 provenance 还原为精确多段线:

```
for each RawLoop:
    合并连续同源边: 遍历环上相邻 Edge,若 CurveId 相同且该曲线是 Arc/Circle
        且参数(角度)单调连续 → 合并为一段圆弧
    输出 LoopSegment:
        - 源是 Line  → LoopSegment{Line, End=边终点, Bulge=0}
        - 源是 Arc   → LoopSegment{Arc,  End=子弧终点, Bulge=tan(Δangle/4)}
                       (Δangle 由合并后的起止角得到, 子弧也精确)
        - 源是 Ellipse/Spline → 保留为多条 Line 段(已线性化)
        - 源是桥接边 → 输出为直线段(缝隙以直线闭合)
```
- **bulge 符号**取决于子弧转向(CCW 为正)与环的遍历方向,据实际走向定号。
- Arc/Circle 在输出中**精确无损**,Spline/Ellipse 为受控精度折线,桥接缝以直线连接。

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

### 7.1 `TOTALBOUNDARY` / `TB` — 选对象出轮廓(主命令)

对齐真实产品的 `Select objects or [SEttings/ABout]` 交互。

```
1. PromptSelectionOptions: 允许窗口/交叉/点选对象; 关键字 SEttings(改容差/半径/属性)、ABout
2. 过滤支持的实体类型
3. 锁文档 + 读事务 → EntityExtractor 取 InputCurve[]
4. BoundaryExtractor.Extract(Mode=SelectedObjects)
5. PolylineWriter 写回外轮廓 + 孔洞多段线
6. 报告: "生成 N 条轮廓(含 H 个孔), 桥接 M 处间隙, 忽略 K 条悬挂线"
```

### 7.2 `TBALL` — 全图轮廓

```
1. 取当前空间(模型/图纸)全部支持实体(无需用户选择)
2. EntityExtractor → Extract(Mode=SelectedObjects)
3. 大图: 强制显示进度条 + Esc 中断
4. 写回外轮廓 + 孔洞
```

### 7.3 `TBPICK` — 点选片段轮廓

> 语义:对**拾取点所在的连通线工片段**出轮廓,而非 BPOLY 的"该点所在单个封闭面"(后者属 SuperBoundary,不在范围)。

```
1. PromptPoint: 取拾取点(可循环多次)
2. 候选实体: 当前空间全部支持实体(或拾取点邻域窗口预过滤)
3. EntityExtractor → BoundaryExtractor.Extract(Mode=FragmentAtPoint, pickPoint)
   - Core 内:
       a. 构建平面图后, 找拾取点所在面;
       b. 由该面回溯其所属的连通分量(片段);
       c. 仅对该片段执行 ⑦ 面分类 + ⑧ 轮廓提取
   - 若拾取点落在无界外面且不属任何片段 → 报"该点不在任何图形片段内"
4. 写回该片段轮廓; 支持连续点选累积
```

---

## 八、容差与健壮性模型

| 容差 | 默认 | 作用 |
|------|------|------|
| `GapTolerance` | 0.001 | 端点聚类桥接(5.4),用户主调参数 |
| `ClosingRadius` (r) | ≈2–5×GapTolerance(自适应)| 路线 B 面分类:ρ>r→孔洞,ρ≤r→填充。控制轮廓细节 |
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
| TB.Core | `Geometry/PlanarGraphBuilder.cs` | 半边结构 + 极角排序 + 边两侧面登记 | ✅ |
| TB.Core | `Geometry/FaceTracer.cs` | 面遍历得到所有面 | ✅ |
| TB.Core | `Geometry/FaceClassifier.cs` | **路线 B: Solid/Void 分类(内切空圆 vs r)** | ✅ |
| TB.Core | `Geometry/OutlineBuilder.cs` | **轮廓边提取 + 组装外轮廓/孔洞环** | ✅ |
| TB.Core | `Geometry/LoopRebuilder.cs` | 还原 bulge 圆弧 | ✅ |
| TB.Core | `Geometry/RobustPredicates.cs` | 叉积/朝向/共线判定 | ✅ |
| TB.Core | `OutlineExtractorService.cs` | 门面(IBoundaryExtractor),串联流水线 | ✅ |
| TB.AutoCAD | `Services/EntityExtractor.cs` | DBObject→InputCurve(投影/块展开) | 集成 |
| TB.AutoCAD | `Services/PolylineWriter.cs` | BoundaryLoop→LWPolyline | 集成 |
| TB.AutoCAD | `Services/UcsService.cs` | UCS↔WCS 变换 | 集成 |
| TB.AutoCAD | `Services/{Selection,Layer,Progress}Service.cs` | 选择/图层/进度 | 集成 |
| TB.AutoCAD | `Commands/{TotalBoundaryCmd,TBAll,TBPick}.cs` | 命令入口 | 集成 |

---

## 十、开发顺序与验收关卡 (DoD)

| 步骤 | 实现 | 验收样例(可单测/集成) |
|------|------|------|
| 1 | 数据模型 + OutlineExtractorService 空壳 | 项目编译,门面可调用 |
| 2 | Tessellator | 圆/弧/样条展平点数随 sagitta 收敛;Line 不变 |
| 3 | UniformGrid + RobustPredicates | 邻域查询正确;叉积/朝向单测通过 |
| 4 | SegmentIntersector | 1 个十字交叉 → 4 条子边,交点唯一 |
| 5 | VertexMerger | 0.5mm 间隙的矩形 4 边 → 4 顶点(桥接成功) |
| 6 | PlanarGraphBuilder + FaceTracer | 矩形→1 面;两相邻矩形→3 面;边两侧面登记正确 |
| 7 | **FaceClassifier(路线 B)** | 大空腔=Void;小单元=Solid;闭合半径变化致分类切换 |
| 8 | **OutlineBuilder** | 稠密网格→仅外轮廓;donut→外轮廓+1 孔;多片段→多外轮廓 |
| 9 | LoopRebuilder | 含圆弧矩形 → 输出含 bulge,重建几何≈原弧 |
| 10 | EntityExtractor + PolylineWriter | 在 AutoCAD 中矩形/圆/带弧图形轮廓正确 |
| 11 | 三命令 | TOTALBOUNDARY/TBALL/TBPICK 均可用 |
| 12 | 进度/中断 + 诊断 | 10000+ 实体 ≤ 5s,可 Esc;诊断输出合理 |
| 13 | 对比验证 | 选定片段轮廓符合预期;带间隙图本插件成功而原生失败 |

---

## 十一、关键测试用例(TB.Core 单测,无需 AutoCAD)

```
[矩形-纯直线]      4 Line 首尾相接           → 1 条外轮廓, 4 段全 Line
[矩形-带间隙]      4 Line 各留 0.5mm 缝       → 容差 1mm: 1 条轮廓; 容差 0.1mm: 0 条
[圆]               1 Circle                   → 1 条外轮廓, 2 段 Arc(bulge=±1)
[圆角矩形]         4 Line + 4 Arc             → 1 条外轮廓, 8 段(4 Line+4 Arc bulge)
[十字交叉]         2 条交叉 Line              → 中段打断; 无面→无轮廓(纯悬挂)
[稠密网格]         # 形 N×N 线交叉(小格)     → 仅 1 条外周界(内部边不输出, 小格=Solid)
[donut]            外框 + 大空腔(r 内)        → 外轮廓 + 1 孔洞(空腔 ρ>r=Void)
[网格-小 r]        同上网格但 r 调小          → 小格变 Void → 出现多个孔(验证 r 敏感性)
[多片段]           两个不相连的矩形           → 2 条独立外轮廓
[嵌套环]           外矩形 + 内矩形(中间空)    → 外轮廓 + 1 孔(IsOuter=false)
[T 形搭接]         一线端点落在另一线中段     → 打断且顶点合并
[样条闭合]         1 闭合 Spline              → 1 条外轮廓, 多 Line 段(矢高内)
[片段点选]         多片段中点选其一           → 仅该片段轮廓
[退化]             零长线/重复线/孤立悬挂线   → 剔除/忽略, 进诊断, 不崩溃
```

---

## 十二、与原生命令的核心差异验证(基础版即体现)

| 场景 | 原生 BOUNDARY/BPOLY | 基础版预期(TotalBoundary 语义) |
|------|--------------------|-----------|
| 目标 | 点选点的单个封闭面 | 选定对象集合的**整体外轮廓 + 孔洞** |
| 复杂图整体轮廓 | 需手动加包围框/逐个拼 | 选中即出外轮廓 ✅ |
| 有 0.5mm 缝隙的"封闭"图 | 报"未找到有效边界" | 容差内自动桥接 ✅ |
| 中段交叉但视觉封闭 | 依赖显示精度,易失败 | 显式打断 ✅ |
| 含圆弧边界 | 可成功 | 成功且 bulge 无损 ✅ |
| 悬挂/多余线 | 常需预清理 | 自动忽略 ✅ |
| 容差/细节调节 | 不可控 | GapTolerance + ClosingRadius 全参数化 ✅ |

> 基础版即应在"**整体外轮廓提取 + 间隙桥接 + 中段相交 + 免预清理**"上**明显优于**原生命令,这正是 TotalBoundary 的核心卖点(轮廓生成器,而非逐面提取)。

---

> 本设计与 `totalboundary-dev-plan-v2.md`(v2.1)配套:主方案描述总体架构与分级,本文件给出基础版可直接编码的详细设计(数据模型、算法伪代码、命令流程、验收关卡)。核心已对齐 TotalBoundary 的"外轮廓 + 孔洞"语义(路线 B)。后续可据此生成 TB.Core / TB.AutoCAD 的解决方案骨架代码。
