# TotalBoundary Clone — 基础版 AI 编码执行计划

> 版本: v1.0 | 日期: 2026-06-29  
> 配套文档:
> - `docs/totalboundary-dev-plan-v2.md` (v2.1)
> - `docs/totalboundary-foundation-design.md` (v1.1)
>
> 本文档用于指导 AI 直接编码基础版 Foundation。目标是把方案拆成**可执行、可验证、可逐步提交**的小任务，避免 AI 自行发挥导致偏离 TotalBoundary 的真实语义。

---

## 0. AI 编码总原则

### 0.1 目标不可跑偏

本项目复刻 **TotalBoundary**，不是 SuperBoundary。

必须始终牢记:

```text
TotalBoundary = 选定对象集合的外轮廓 / 周界 + 内部孔洞
SuperBoundary / BOUNDARY = 点选或提取每个封闭内部面
```

基础版最终输出应是:

- 一个或多个**外轮廓环**；
- 必要时输出内部**孔洞环**；
- 不应输出稠密网格中的每个小格子；
- 不应把 FaceTracer 得到的所有 face 直接写成多段线。

### 0.2 实现顺序

必须先实现并验证 `TB.Core`，再实现 `TB.AutoCAD`。

```text
先纯几何、纯单测 → 再接 AutoCAD API → 再做命令和集成测试
```

理由:

- AutoCAD `DBObject` 不可跨线程访问，也不适合 mock；
- 几何算法必须在无 CAD 环境下可重复单测；
- AI 每实现一个小模块都要有单测作为回归护栏。

### 0.3 兼容性约束

基础代码需兼容 `net40` 与 `net8.0-windows`。

因此核心项目中应避免:

- `init` 属性；
- collection expression `[]`；
- nullable reference types 依赖；
- 只在高版本 .NET 可用的 API；
- 未加 polyfill 的 C# 新语法。

建议 DTO 使用:

```csharp
public sealed class BoundaryOptions
{
    public double GapTolerance { get; set; }
    public double ClosingRadius { get; set; }
}
```

不要写:

```csharp
public double GapTolerance { get; init; }
public IReadOnlyList<Edge> Edges { get; init; } = [];
```

### 0.4 每步必须可验证

每完成一个模块，必须:

1. 增加或更新对应单元测试；
2. 运行相关测试；
3. 不能把多个算法模块一次性堆完再测；
4. 不能跳过失败测试继续写后续模块。

---

## 1. 推荐解决方案结构

```text
TotalBoundaryClone/
├── src/
│   ├── TB.Core/
│   │   ├── Model/
│   │   ├── Geometry/
│   │   ├── Pipeline/
│   │   └── TB.Core.csproj
│   └── TB.AutoCAD/
│       ├── Commands/
│       ├── Services/
│       └── TB.AutoCAD.csproj
├── tests/
│   └── TB.Core.Tests/
│       ├── Geometry/
│       ├── Pipeline/
│       └── TB.Core.Tests.csproj
├── bundle/
│   └── PackageContents.xml
└── TB.sln
```

### 1.1 项目依赖方向

```text
TB.Core        无 CAD 依赖
TB.Core.Tests  仅依赖 TB.Core
TB.AutoCAD     依赖 TB.Core + AutoCAD .NET API
```

严禁:

```text
TB.Core 引用 acmgd.dll / acdbmgd.dll / accoremgd.dll
```

---

## 2. 最小可运行 MVP 边界

AI 第一轮编码只做 MVP，先确保核心语义闭环。

### 2.1 MVP 输入实体

第一阶段只支持:

- Line；
- Arc；
- Circle；
- LWPolyline 中的直线段和 bulge 圆弧段。

Spline / Ellipse / BlockReference 可以先留接口和诊断，不阻塞 MVP。

### 2.2 MVP 输出

输出 `BoundaryResult`，包含:

- `BoundaryLoop[] Loops`
- 每个 loop:
  - `IsOuter`
  - `ParentLoopIndex`
  - `SignedArea`
  - `LoopSegment[]`

MVP 可以只在 `TB.Core.Tests` 验证输出，不需要立刻写 AutoCAD 实体。

### 2.3 MVP 路线 B 实现策略

`FaceClassifier` 使用 **Polylabel / adaptive grid** 近似最大内切空圆半径。

暂不实现:

- 精确 Voronoi / 中轴；
- 复杂椭圆和样条；
- 多层块；
- Hatch / DXF / GUI。

---

## 3. 分阶段编码计划

## Phase 0 — 工程骨架

### 任务 0.1 创建解决方案和项目

#### 产出

- `TB.sln`
- `src/TB.Core/TB.Core.csproj`
- `tests/TB.Core.Tests/TB.Core.Tests.csproj`

#### 要求

- `TB.Core` 多目标:

```xml
<TargetFrameworks>net40;net8.0-windows</TargetFrameworks>
```

- `TB.Core.Tests` 可先只跑在现代测试框架可支持的目标框架，例如 `net8.0`。
- 测试框架可用 xUnit 或 NUnit。

#### 验证

```bash
dotnet test
```

若本地 SDK 不支持 `net40` 构建，可先让 `TB.Core.Tests` 引用 `net8.0` 构建的 `TB.Core`，并在文档中记录环境限制。

---

## Phase 1 — 基础几何模型

### 任务 1.1 实现基本 DTO

#### 文件

```text
src/TB.Core/Model/Point2.cs
src/TB.Core/Model/Vector2.cs
src/TB.Core/Model/BoundingBox2.cs
src/TB.Core/Model/InputCurve.cs
src/TB.Core/Model/BoundaryOptions.cs
src/TB.Core/Model/BoundaryLoop.cs
src/TB.Core/Model/BoundaryResult.cs
```

#### 接口契约

```csharp
public struct Point2
{
    public double X { get; set; }
    public double Y { get; set; }

    public Point2(double x, double y);
    public double DistanceTo(Point2 other);
}

public struct BoundingBox2
{
    public Point2 Min { get; set; }
    public Point2 Max { get; set; }

    public void Expand(Point2 p);
    public void Inflate(double amount);
    public bool Intersects(BoundingBox2 other);
}

public enum CurveKind
{
    Line,
    Arc,
    EllipseArc,
    Spline
}

public sealed class InputCurve
{
    public int Id { get; set; }
    public CurveKind Kind { get; set; }
    public Point2 Start { get; set; }
    public Point2 End { get; set; }
    public bool IsClosed { get; set; }

    public Point2 Center { get; set; }
    public double Radius { get; set; }
    public double StartAngle { get; set; }
    public double EndAngle { get; set; }

    public long SourceHandle { get; set; }
}
```

#### 测试

```text
Point2_DistanceTo_ReturnsExpected
BoundingBox_Expand_UpdatesMinMax
BoundingBox_Inflate_ExpandsBothDirections
BoundingBox_Intersects_DetectsOverlap
```

---

### 任务 1.2 实现 `RobustPredicates`

#### 文件

```text
src/TB.Core/Geometry/RobustPredicates.cs
```

#### 接口契约

```csharp
public static class RobustPredicates
{
    public static double Cross(Point2 a, Point2 b, Point2 c);
    public static int Orientation(Point2 a, Point2 b, Point2 c, double eps);
    public static bool AreClose(Point2 a, Point2 b, double eps);
    public static bool IsBetween(Point2 a, Point2 b, Point2 p, double eps);
}
```

#### 测试

```text
Orientation_ReturnsCounterClockwise
Orientation_ReturnsClockwise
Orientation_ReturnsCollinearWithinEpsilon
IsBetween_ReturnsTrueForPointOnSegment
IsBetween_ReturnsFalseForOutsideCollinearPoint
```

---

## Phase 2 — 曲线展平 Tessellator

### 任务 2.1 实现 `Edge`

#### 文件

```text
src/TB.Core/Model/Edge.cs
```

#### 接口契约

```csharp
public enum EdgeKind
{
    Real,
    Bridge
}

public sealed class Edge
{
    public int Id { get; set; }
    public Point2 A { get; set; }
    public Point2 B { get; set; }
    public int CurveId { get; set; }
    public double TA { get; set; }
    public double TB { get; set; }
    public EdgeKind Kind { get; set; }
    public bool IsDegenerate(double eps);
    public BoundingBox2 GetBounds();
}
```

### 任务 2.2 实现 `Tessellator`

#### 文件

```text
src/TB.Core/Geometry/Tessellator.cs
```

#### 输入

```text
InputCurve[]
BoundaryOptions
```

#### 输出

```text
List<Edge>
```

#### MVP 规则

- Line → 1 条 `Real` edge。
- Arc → 按角步细分为多条 edge，保留 `CurveId`、`TA/TB` 角度。
- Circle → 拆为两个半圆 arc，再展平。
- Spline / Ellipse → MVP 可抛出 `NotSupportedDiagnostic` 或用预留接口。

#### 测试

```text
Tessellate_Line_ReturnsSingleEdge
Tessellate_Arc_ReturnsMultipleEdgesWithSameCurveId
Tessellate_Circle_ReturnsClosedEdges
Tessellate_Arc_RespectsMaxArcStep
```

---

## Phase 3 — 空间索引

### 任务 3.1 实现 `UniformGrid`

#### 文件

```text
src/TB.Core/Geometry/UniformGrid.cs
```

#### 接口契约

```csharp
public sealed class UniformGrid
{
    public UniformGrid(double cellSize);
    public void Insert(int itemId, BoundingBox2 bounds);
    public IList<int> Query(BoundingBox2 bounds);
}
```

#### 规则

- cell key 使用 `(long ix, long iy)`，不要用浮点作为字典 key。
- `Query` 可以返回重复 id，但调用方必须去重；或在内部去重。

#### 测试

```text
Grid_Query_ReturnsInsertedItem
Grid_Query_ReturnsItemsAcrossMultipleCells
Grid_Query_DoesNotReturnFarItem
Grid_Query_DeduplicatesMultiCellItem
```

---

## Phase 4 — 线段求交与打断

### 任务 4.1 实现线段求交

#### 文件

```text
src/TB.Core/Geometry/SegmentIntersection.cs
src/TB.Core/Geometry/SegmentIntersector.cs
```

#### 接口契约

```csharp
public enum IntersectionKind
{
    None,
    Point,
    Overlap
}

public sealed class SegmentIntersection
{
    public IntersectionKind Kind { get; set; }
    public Point2 Point { get; set; }
    public Point2 OverlapStart { get; set; }
    public Point2 OverlapEnd { get; set; }
    public double TOnFirst { get; set; }
    public double TOnSecond { get; set; }
}

public static class SegmentIntersector
{
    public static SegmentIntersection Intersect(Edge a, Edge b, double eps);
    public static IList<Edge> SplitAtIntersections(IList<Edge> edges, double eps);
}
```

#### 要求

必须支持:

- 交叉 X；
- T 形搭接；
- 端点接触；
- 平行不相交；
- 共线重叠。

#### 测试

```text
Intersect_Cross_ReturnsPoint
Intersect_TShape_ReturnsPointOnMiddleOfSegment
Intersect_EndpointTouch_ReturnsPoint
Intersect_Parallel_ReturnsNone
Intersect_CollinearOverlap_ReturnsOverlap
SplitAtIntersections_Cross_SplitsBothEdges
SplitAtIntersections_TShape_SplitsOnlyHitEdgeIfOtherEndpointAlreadyExists
```

---

## Phase 5 — 间隙桥接

### 任务 5.1 统一语义:不要先移动端点

本阶段必须实现为:

```text
真实端点保持原坐标；
距离 <= GapTolerance 的开放端之间插入 BridgeEdge；
仅在距离 <= GeomEpsilon 时可以合并为同一 Vertex。
```

这样更接近 TotalBoundary 的"自动消除缝隙"表现，也避免破坏圆弧 provenance。

### 任务 5.2 实现 `GapBridger`

#### 文件

```text
src/TB.Core/Geometry/GapBridger.cs
```

#### 接口契约

```csharp
public sealed class GapBridgeResult
{
    public IList<Edge> Edges { get; set; }
    public int BridgeCount { get; set; }
}

public sealed class GapBridger
{
    public GapBridgeResult AddBridgeEdges(IList<Edge> realEdges, BoundaryOptions options);
}
```

#### 规则

1. 计算所有 edge 端点；
2. 查找距离 ≤ `GapTolerance` 的端点对；
3. 只连接**开放端**或低度端点；
4. 每个端点最多选择最近的一个桥接对象；
5. 插入 `EdgeKind.Bridge`；
6. BridgeEdge 不带 `CurveId`，输出时作为直线段。

#### 测试

```text
GapBridger_AddsBridgeForSmallGap
GapBridger_DoesNotBridgeLargeGap
GapBridger_ChoosesNearestEndpoint
GapBridger_DoesNotCreateDuplicateBridges
GapBridger_DoesNotMoveOriginalEndpoints
```

---

## Phase 6 — 平面图 DCEL

### 任务 6.1 实现 Vertex 与 HalfEdge

#### 文件

```text
src/TB.Core/Model/Vertex.cs
src/TB.Core/Model/HalfEdge.cs
src/TB.Core/Geometry/PlanarGraph.cs
src/TB.Core/Geometry/PlanarGraphBuilder.cs
```

#### 接口契约

```csharp
public sealed class Vertex
{
    public int Id { get; set; }
    public Point2 Point { get; set; }
}

public sealed class HalfEdge
{
    public int Id { get; set; }
    public int OriginVertexId { get; set; }
    public int DestinationVertexId { get; set; }
    public int TwinId { get; set; }
    public int EdgeId { get; set; }
    public double Angle { get; set; }
    public int FaceId { get; set; }
    public bool Visited { get; set; }
}

public sealed class PlanarGraph
{
    public IList<Vertex> Vertices { get; set; }
    public IList<Edge> Edges { get; set; }
    public IList<HalfEdge> HalfEdges { get; set; }
    public IDictionary<int, IList<int>> OutgoingHalfEdgeIdsByVertex { get; set; }
}
```

#### Vertex 归并规则

- 距离 ≤ `GeomEpsilon` 的端点归为同一 Vertex；
- 不用 `GapTolerance` 归并顶点，因为 GapTolerance 已通过 BridgeEdge 表达；
- 顶点代表坐标采用确定性规则，例如 `(minX, minY)` 或最先出现端点。

#### 测试

```text
GraphBuilder_CreatesTwoHalfEdgesPerEdge
GraphBuilder_AssignsTwinIds
GraphBuilder_SortsOutgoingByAngle
GraphBuilder_MergesOnlyEpsilonCloseVertices
```

---

## Phase 7 — FaceTracer

### 任务 7.1 实现 Face

#### 文件

```text
src/TB.Core/Model/Face.cs
src/TB.Core/Geometry/FaceTracer.cs
```

#### 接口契约

```csharp
public enum FaceKind
{
    Unknown,
    Solid,
    Void,
    UnboundedVoid
}

public sealed class Face
{
    public int Id { get; set; }
    public IList<int> HalfEdgeIds { get; set; }
    public double SignedArea { get; set; }
    public FaceKind Kind { get; set; }
    public bool IsUnbounded { get; set; }
}

public sealed class FaceTraceResult
{
    public IList<Face> Faces { get; set; }
    public PlanarGraph Graph { get; set; }
}
```

#### 面遍历规则

在当前半边 `he` 到达终点 `v` 后:

1. 找到 `he.Twin` 在 `v` 的出边排序列表中的位置；
2. 取其顺时针方向的下一条出边；
3. 重复直到回到起始 half-edge。

必须固定规则，不可一会儿取前一个、一会儿取后一个。

#### 无界面识别

MVP 规则:

- 按固定 traversal 方向，signed area < 0 的最大绝对面积 face 标为 `UnboundedVoid`；
- 其余 face 初始 `Unknown`；
- 后续如发现多连通分量问题，再扩展为按连通分量识别。

#### 测试

```text
FaceTracer_Rectangle_FindsOneBoundedFaceAndOneUnboundedFace
FaceTracer_TwoAdjacentRectangles_FindsTwoBoundedFaces
FaceTracer_Grid2x2_FindsFourSmallBoundedFaces
FaceTracer_AssignsFaceIdToHalfEdges
```

---

## Phase 8 — 路线 B FaceClassifier

### 任务 8.1 实现 Polylabel 近似最大内切圆

#### 文件

```text
src/TB.Core/Geometry/Polylabel.cs
src/TB.Core/Geometry/FaceClassifier.cs
```

#### Polylabel 输入

```text
Polygon ring: Point2[]
precision: double
```

#### Polylabel 输出

```csharp
public sealed class PolylabelResult
{
    public Point2 Center { get; set; }
    public double Distance { get; set; } // 近似最大内切圆半径 rho
}
```

#### 算法要求

使用 adaptive grid:

1. 取 polygon bbox；
2. 以 bbox 较小边作为初始 cell size；
3. 每个 cell 保存:
   - center；
   - halfSize；
   - distanceToPolygonBoundary；
   - maxPotential = distance + halfSize * sqrt(2)；
4. 用 maxPotential 优先队列；
5. 每次取最有潜力 cell；
6. 若 `cell.maxPotential - best.distance <= precision` 停止细分；
7. 否则拆成四个子 cell；
8. best.distance 即 ρ(F)。

#### 距离计算

`SignedDistanceToPolygon(point, ring)`:

- point 在 polygon 内部 → 正距离；
- point 在外部 → 负距离；
- 距离值为 point 到所有边的最短距离。

#### FaceClassifier 接口

```csharp
public sealed class FaceClassifier
{
    public void ClassifyFaces(PlanarGraph graph, IList<Face> faces, BoundaryOptions options);
}
```

#### 分类规则

```text
if face.IsUnbounded:
    face.Kind = UnboundedVoid
else:
    rho = Polylabel(faceRing).Distance
    r = options.ClosingRadius > 0
        ? options.ClosingRadius
        : options.GapTolerance * 3.0
    face.Kind = rho > r ? Void : Solid
```

#### 测试

```text
Polylabel_Rectangle_ReturnsHalfOfShortSide
Polylabel_NarrowCorridor_ReturnsSmallRadius
FaceClassifier_LargeRectangleClassifiedVoid
FaceClassifier_SmallGridCellClassifiedSolid
FaceClassifier_ChangingClosingRadiusChangesClassification
UnboundedFace_ClassifiedUnboundedVoid
```

---

## Phase 9 — OutlineBuilder

### 任务 9.1 实现轮廓边保留规则

#### 文件

```text
src/TB.Core/Geometry/OutlineBuilder.cs
```

#### 接口契约

```csharp
public sealed class RawLoop
{
    public IList<int> EdgeIds { get; set; }
    public bool IsOuter { get; set; }
    public int ParentLoopIndex { get; set; }
    public double SignedArea { get; set; }
}

public sealed class OutlineBuilder
{
    public IList<RawLoop> BuildOutlines(PlanarGraph graph, IList<Face> faces);
}
```

#### 保留规则

```text
KeepEdge(e):
  left = face on one side of e
  right = face on other side of e

  if left == Solid && right == Solid:
      return false

  if e.Kind == Bridge:
      return IsBridgeNeededForLoopClosure(e)

  return true
```

#### BridgeEdge 规则

MVP 简化:

- 如果 BridgeEdge 两侧不是 Solid/Solid，先保留；
- 若后续组环时形成开放链或孤立短线，再丢弃；
- 后续可优化为只保留确实参与闭环的 bridge。

#### 组环规则

1. 将 keep edges 转为无向邻接；
2. 从未使用 edge 开始；
3. 在当前顶点选择下一条 edge:
   - 优先选择使轮廓转角最平滑的 edge；
   - 若有多选，按角度和 edge id 做确定性 tie-break；
4. 回到起点即形成 loop；
5. 无法闭合的链丢弃并记录诊断。

#### 方向规则

- signed area > 0 → CCW；
- 外轮廓归一为 CCW；
- 孔洞归一为 CW；
- 孔洞 parent 通过点在多边形测试确定。

#### 测试

```text
OutlineBuilder_Rectangle_ReturnsOneOuterLoop
OutlineBuilder_TwoSeparateRectangles_ReturnsTwoOuterLoops
OutlineBuilder_Grid2x2_ReturnsOnlyOuterLoop
OutlineBuilder_Donut_ReturnsOuterAndHole
OutlineBuilder_DiscardsOpenChain
OutlineBuilder_KeepsBridgeEdgeWhenNeededToCloseGap
```

---

## Phase 10 — LoopRebuilder

### 任务 10.1 实现 bulge 还原

#### 文件

```text
src/TB.Core/Geometry/LoopRebuilder.cs
```

#### 输入

```text
RawLoop[]
PlanarGraph
InputCurve[]
```

#### 输出

```text
BoundaryLoop[]
```

#### 规则

- 连续同 `CurveId` 且参数连续的 Arc 边合并；
- bulge = `tan(deltaAngle / 4)`；
- 如果 loop 方向与 arc 原方向相反，bulge 取负；
- BridgeEdge 输出为直线段；
- Line 输出为直线段；
- Spline/Ellipse 展平段输出为直线段。

#### 测试

```text
LoopRebuilder_LineRectangle_ReturnsFourLineSegments
LoopRebuilder_Circle_ReturnsArcSegmentsWithBulge
LoopRebuilder_ReversedArc_UsesNegativeBulge
LoopRebuilder_BridgeEdge_ReturnsLineSegment
```

---

## Phase 11 — Pipeline 门面

### 任务 11.1 实现 `OutlineExtractorService`

#### 文件

```text
src/TB.Core/Pipeline/OutlineExtractorService.cs
```

#### 流程

```text
InputCurve[]
  -> Tessellator
  -> SegmentIntersector.SplitAtIntersections
  -> GapBridger.AddBridgeEdges
  -> PlanarGraphBuilder.Build
  -> FaceTracer.Trace
  -> FaceClassifier.ClassifyFaces
  -> OutlineBuilder.BuildOutlines
  -> LoopRebuilder.Rebuild
  -> BoundaryResult
```

#### 测试

```text
Pipeline_Rectangle_ReturnsOneLoop
Pipeline_GappedRectangle_ReturnsOneLoopWhenToleranceAllows
Pipeline_GappedRectangle_ReturnsNoLoopWhenToleranceTooSmall
Pipeline_Grid2x2_ReturnsOneOuterLoop
Pipeline_Donut_ReturnsOuterAndHole
Pipeline_TwoSeparateRectangles_ReturnsTwoLoops
```

---

## Phase 12 — AutoCAD 集成

必须在 `TB.Core` 通过上述测试后再做。

### 任务 12.1 EntityExtractor

#### 文件

```text
src/TB.AutoCAD/Services/EntityExtractor.cs
```

#### 支持顺序

1. Line；
2. Arc；
3. Circle；
4. Polyline 直线段；
5. Polyline bulge 段；
6. Ellipse / Spline 诊断暂不支持；
7. BlockReference 单层展开。

#### 验证

在 AutoCAD / accoreconsole 内加载测试 DWG:

```text
Line rectangle -> InputCurve count = 4
Circle -> InputCurve count = 2 arcs
Polyline bulge -> InputCurve includes Arc
```

### 任务 12.2 PolylineWriter

#### 文件

```text
src/TB.AutoCAD/Services/PolylineWriter.cs
```

#### 要求

- 使用 `Autodesk.AutoCAD.DatabaseServices.Polyline`；
- `AddVertexAt(index, point2d, bulge, startWidth, endWidth)`；
- 设置 `Closed = true`；
- 设置图层、颜色、线宽；
- 一个命令内统一事务，支持一次 Undo。

### 任务 12.3 Commands

#### 文件

```text
src/TB.AutoCAD/Commands/TotalBoundaryCmd.cs
src/TB.AutoCAD/Commands/TBAll.cs
src/TB.AutoCAD/Commands/TBPick.cs
```

#### 命令

```text
TOTALBOUNDARY
TB
TBALL
TBPICK
```

#### 命令流程

```text
选择 / 点选
  -> EntityExtractor
  -> OutlineExtractorService
  -> PolylineWriter
  -> Editor.WriteMessage 输出统计
```

---

## 4. AI 每轮编码提交规则

每轮建议只做一个 Phase 或一个小任务。

### 每轮完成后必须执行

```bash
dotnet test
git status --short
```

若本环境缺少 .NET SDK 或 AutoCAD SDK:

- 记录无法运行的原因；
- 至少保证文档和代码结构一致；
- 不得伪造测试通过。

### 每轮提交信息格式

```text
feat(core): add robust geometry primitives
test(core): cover segment intersection cases
feat(core): add route-b face classifier
feat(autocad): add TOTALBOUNDARY command
```

---

## 5. 禁止 AI 自行发挥清单

AI 编码时不得:

1. 不得把 FaceTracer 的所有 face 直接输出为 polyline；
2. 不得把项目目标改成 SuperBoundary / BPOLY 替代；
3. 不得依赖 `Hatch.GenerateBoundary` 作为核心算法；
4. 不得在 `TB.Core` 引用 AutoCAD DLL；
5. 不得跨线程访问 `DBObject`；
6. 不得提前删除看似悬挂但尚未求交打断的线；
7. 不得线性化圆弧作为最终输出，圆弧必须尽量还原为 bulge；
8. 不得用 `GapTolerance` 直接移动大间隙端点，应插入 BridgeEdge；
9. 不得一次性生成全部算法而不写单测；
10. 不得为了通过测试硬编码测试图形。

---

## 6. 最小验收套件

基础版第一阶段完成时，至少通过以下 `TB.Core.Tests`:

```text
Pipeline_Rectangle_ReturnsOneLoop
Pipeline_GappedRectangle_ReturnsOneLoopWhenToleranceAllows
Pipeline_GappedRectangle_ReturnsNoLoopWhenToleranceTooSmall
Pipeline_Circle_ReturnsBulgeLoop
Pipeline_Grid2x2_ReturnsOnlyOuterLoop
Pipeline_Donut_ReturnsOuterAndHole
Pipeline_TwoSeparateRectangles_ReturnsTwoLoops
Pipeline_OpenCross_ReturnsNoLoop
```

AutoCAD 集成第一阶段完成时，至少手工或 CoreConsole 验证:

```text
TOTALBOUNDARY 选 4 条矩形线 -> 生成 1 条闭合 Polyline
TOTALBOUNDARY 选带小缝矩形 -> 容差内生成 1 条闭合 Polyline
TOTALBOUNDARY 选 2x2 网格 -> 只生成外轮廓，不生成 4 个小格
TBALL 当前空间简单图 -> 生成轮廓
TBPICK 多片段图 -> 只生成被点选片段轮廓
```

---

## 7. 建议第一批 AI 编码任务

如果让 AI 开始写代码，建议第一批任务只做:

```text
任务 A:
创建解决方案骨架 + TB.Core + TB.Core.Tests
实现 Point2 / BoundingBox2 / RobustPredicates
补齐对应单测

任务 B:
实现 Edge / Tessellator(Line/Arc/Circle)
补齐对应单测

任务 C:
实现 UniformGrid + SegmentIntersector
补齐交叉/T形/端点/重叠单测

任务 D:
实现 GapBridger + PlanarGraphBuilder + FaceTracer
补齐矩形/网格面追踪单测

任务 E:
实现 Polylabel + FaceClassifier + OutlineBuilder
补齐路线 B 单测

任务 F:
实现 LoopRebuilder + Pipeline 端到端单测
```

只有任务 F 通过后，再开始 `TB.AutoCAD`。

---

## 8. 完成定义 (Definition of Done)

基础版 Core MVP 完成需满足:

- `TB.Core` 无 CAD 依赖；
- 所有 Phase 1–11 单测通过；
- 可以处理矩形、圆、带缝矩形、2x2 网格、donut、多片段；
- 输出不是所有 face，而是外轮廓 + 孔洞；
- 圆弧输出包含 bulge；
- `ClosingRadius` 改变会影响孔洞/填充分界；
- `GapTolerance` 改变会影响桥接结果；
- 开放悬挂线不会输出为轮廓；
- 诊断信息可说明被忽略/桥接的对象数量。

基础版 AutoCAD MVP 完成需满足:

- `TOTALBOUNDARY` / `TB` 可选对象并生成闭合 Polyline；
- `TBALL` 可对当前空间全部对象生成轮廓；
- `TBPICK` 可对点选片段生成轮廓；
- 命令失败时事务回滚；
- 单次命令生成结果可一次 Undo；
- 不跨线程访问 AutoCAD 对象。
