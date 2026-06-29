# TotalBoundary Clone — AI 自动编程分阶段执行计划

> 版本: v1.0 | 日期: 2026-06-29  
> 目的: 指导 AI 按阶段完成基础版 Foundation 的编码、测试、提交与验证。  
> 配套文档:
> - `docs/totalboundary-dev-plan-v2.md`
> - `docs/totalboundary-foundation-design.md`
> - `docs/totalboundary-foundation-ai-implementation-plan.md`

---

## 0. 执行总原则

### 0.1 每轮只做一个清晰目标

AI 每轮只实现一个 Phase 或一个 Phase 内的小任务。禁止一次性实现多个高风险算法模块。

每轮必须遵循:

```text
读取相关文档
  → 实现最小代码
  → 编写/更新测试
  → 运行测试
  → 修复失败
  → git status
  → git add
  → git commit
  → git push
  → 更新 PR
```

### 0.2 不承诺一次完成整个商业级插件

本计划目标是让 AI 以**可验证的小步迭代**完成基础版 MVP，再逐步扩展。复杂计算几何必须依靠测试用例逐步收敛。

### 0.3 严格防跑偏

每轮开始前 AI 必须复述本轮仍遵守:

```text
TotalBoundary = 选定对象集合的外轮廓 / 周界 + 内部孔洞
不是 SuperBoundary / BOUNDARY 的逐面提取
```

禁止:

- 输出所有 FaceTracer 面；
- 把稠密网格中的每个小格输出为 polyline；
- 依赖 `Hatch.GenerateBoundary`；
- 在 `TB.Core` 引用 AutoCAD DLL；
- 在 Core 阶段编写 AutoCAD 代码；
- 不写测试直接推进下一阶段。

---

## 1. 分支与提交策略

### 1.1 工作分支

编码建议新建独立实现分支:

```bash
git checkout develop
git checkout -b cursor/totalboundary-foundation-core-e23d
```

如果继续沿当前文档分支开发，也必须保持提交粒度清晰。推荐代码实现与文档 PR 分离。

### 1.2 提交规则

每一轮至少一个提交，提交信息格式:

```text
feat(core): add geometry primitives
test(core): cover segment intersection cases
feat(core): add route-b face classifier
feat(autocad): add TOTALBOUNDARY command
```

每个提交应满足:

- 代码可编译，或明确记录因环境缺失无法编译；
- 相关测试已运行；
- 不混入无关重构；
- 不修改已通过测试的算法行为，除非同步更新测试说明。

---

## 2. 环境检查阶段

## Phase E0 — 环境与可构建性检查

### 目标

确认当前机器是否具备 .NET SDK、测试框架、AutoCAD SDK / ObjectARX 引用路径。

### 执行命令

```bash
dotnet --info
dotnet --list-sdks
```

如存在 AutoCAD SDK:

```bash
echo "$ACAD2013"
echo "$ACAD2025"
```

### 输出

新增或更新:

```text
docs/totalboundary-env-notes.md
```

记录:

- 可用 .NET SDK；
- 是否能构建 `net40`；
- 是否具备 ObjectARX 2013/2025；
- 是否具备 accoreconsole；
- 当前阶段能运行哪些测试。

### 完成标准

- 明确 Core 单测是否可运行；
- 明确 AutoCAD 集成是否可在当前环境验证；
- 若缺少 AutoCAD SDK，不阻塞 Phase C0–C10 的 `TB.Core` 开发。

### 提交

```bash
git add docs/totalboundary-env-notes.md
git commit -m "docs: record TotalBoundary build environment notes"
git push -u origin <branch>
```

---

## 3. Core 编码阶段

## Phase C0 — 解决方案与测试骨架

### 目标

创建可测试的 Core 工程骨架。

### 输入文档

- `totalboundary-foundation-ai-implementation-plan.md` Phase 0

### 产出文件

```text
TB.sln
src/TB.Core/TB.Core.csproj
tests/TB.Core.Tests/TB.Core.Tests.csproj
tests/TB.Core.Tests/SmokeTests.cs
```

### 实现要求

- `TB.Core` 不引用 AutoCAD DLL；
- 测试项目可以先只目标 `net8.0`；
- 如果 `net40` 在当前环境无法构建，应记录在 env notes 中，不得伪造。

### 最小测试

```text
Smoke_CoreProjectLoads
```

### 验证命令

```bash
dotnet test
```

### 完成标准

- `dotnet test` 可运行；
- 有 1 个 smoke test 通过；
- 仓库中出现标准 `src/` 与 `tests/` 结构。

---

## Phase C1 — 几何基础模型

### 目标

实现所有后续算法依赖的基础 DTO。

### 产出文件

```text
src/TB.Core/Model/Point2.cs
src/TB.Core/Model/Vector2.cs
src/TB.Core/Model/BoundingBox2.cs
src/TB.Core/Model/BoundaryOptions.cs
src/TB.Core/Model/BoundaryResult.cs
src/TB.Core/Model/BoundaryLoop.cs
tests/TB.Core.Tests/Model/Point2Tests.cs
tests/TB.Core.Tests/Model/BoundingBox2Tests.cs
```

### 接口要求

使用 `get; set;`，避免 `init` 和 collection expression。

### 必须测试

```text
Point2_DistanceTo_ReturnsExpected
BoundingBox_Expand_UpdatesMinMax
BoundingBox_Inflate_ExpandsBothDirections
BoundingBox_Intersects_DetectsOverlap
```

### 验证命令

```bash
dotnet test --filter "Point2|BoundingBox"
dotnet test
```

### 完成标准

- 基础 DTO 单测全通过；
- 不包含任何 CAD 引用；
- 后续模块可复用这些类型。

---

## Phase C2 — RobustPredicates

### 目标

实现浮点几何判定统一入口。

### 产出文件

```text
src/TB.Core/Geometry/RobustPredicates.cs
tests/TB.Core.Tests/Geometry/RobustPredicatesTests.cs
```

### 必须测试

```text
Orientation_ReturnsCounterClockwise
Orientation_ReturnsClockwise
Orientation_ReturnsCollinearWithinEpsilon
IsBetween_ReturnsTrueForPointOnSegment
IsBetween_ReturnsFalseForOutsideCollinearPoint
AreClose_RespectsEpsilon
```

### 验证命令

```bash
dotnet test --filter RobustPredicates
dotnet test
```

### 完成标准

- 所有几何比较后续必须调用该模块；
- 不允许各模块自行散落 eps 判断逻辑。

---

## Phase C3 — Edge 与 Tessellator

### 目标

把 Line / Arc / Circle 转为带 provenance 的直线边。

### 产出文件

```text
src/TB.Core/Model/InputCurve.cs
src/TB.Core/Model/Edge.cs
src/TB.Core/Geometry/Tessellator.cs
tests/TB.Core.Tests/Geometry/TessellatorTests.cs
```

### MVP 支持

- Line；
- Arc；
- Circle 拆为两个半圆或多段 Arc；
- Polyline 暂由 AutoCAD 层拆成 Line/Arc，因此 Core 不需要知道 Polyline。

### 必须测试

```text
Tessellate_Line_ReturnsSingleEdge
Tessellate_Arc_ReturnsMultipleEdgesWithSameCurveId
Tessellate_Circle_ReturnsClosedEdges
Tessellate_Arc_RespectsMaxArcStep
Tessellate_UnsupportedCurve_AddsDiagnosticOrThrowsKnownException
```

### 验证命令

```bash
dotnet test --filter Tessellator
dotnet test
```

### 完成标准

- 每条 Edge 记录 `CurveId`、`TA`、`TB`；
- 圆弧最终输出仍可通过 provenance 还原 bulge；
- Tessellator 不负责轮廓判断。

---

## Phase C4 — UniformGrid 空间索引

### 目标

为求交和近邻端点查找提供空间索引。

### 产出文件

```text
src/TB.Core/Geometry/UniformGrid.cs
tests/TB.Core.Tests/Geometry/UniformGridTests.cs
```

### 必须测试

```text
Grid_Query_ReturnsInsertedItem
Grid_Query_ReturnsItemsAcrossMultipleCells
Grid_Query_DoesNotReturnFarItem
Grid_Query_DeduplicatesMultiCellItem
Grid_HandlesNegativeCoordinates
```

### 验证命令

```bash
dotnet test --filter UniformGrid
dotnet test
```

### 完成标准

- 支持负坐标；
- cell key 不使用浮点；
- Query 结果确定性排序或由调用方排序。

---

## Phase C5 — SegmentIntersector

### 目标

实现线段求交和交点打断。

### 产出文件

```text
src/TB.Core/Geometry/SegmentIntersection.cs
src/TB.Core/Geometry/SegmentIntersector.cs
tests/TB.Core.Tests/Geometry/SegmentIntersectorTests.cs
```

### 必须测试

```text
Intersect_Cross_ReturnsPoint
Intersect_TShape_ReturnsPointOnMiddleOfSegment
Intersect_EndpointTouch_ReturnsPoint
Intersect_Parallel_ReturnsNone
Intersect_CollinearOverlap_ReturnsOverlap
SplitAtIntersections_Cross_SplitsBothEdges
SplitAtIntersections_TShape_SplitsHitEdge
SplitAtIntersections_DoesNotCreateZeroLengthEdges
```

### 验证命令

```bash
dotnet test --filter SegmentIntersector
dotnet test
```

### 完成标准

- X / T / 端点 / 共线重叠均有测试；
- 打断后不产生零长边；
- 交点参数稳定、可排序。

---

## Phase C6 — GapBridger

### 目标

实现 TotalBoundary 的间隙容差能力。

### 关键约束

GapTolerance 不应直接移动真实端点。

```text
距离 <= GapTolerance 的开放端之间插入 BridgeEdge
距离 <= GeomEpsilon 才可视为同一点
```

### 产出文件

```text
src/TB.Core/Geometry/GapBridger.cs
tests/TB.Core.Tests/Geometry/GapBridgerTests.cs
```

### 必须测试

```text
GapBridger_AddsBridgeForSmallGap
GapBridger_DoesNotBridgeLargeGap
GapBridger_ChoosesNearestEndpoint
GapBridger_DoesNotCreateDuplicateBridges
GapBridger_DoesNotMoveOriginalEndpoints
GapBridger_BridgeEdgeHasKindBridge
```

### 验证命令

```bash
dotnet test --filter GapBridger
dotnet test
```

### 完成标准

- 小缝能闭合；
- 大缝不闭合；
- BridgeEdge 可被后续 OutlineBuilder 识别。

---

## Phase C7 — PlanarGraphBuilder / DCEL

### 目标

构建半边图，作为面遍历基础。

### 产出文件

```text
src/TB.Core/Model/Vertex.cs
src/TB.Core/Model/HalfEdge.cs
src/TB.Core/Geometry/PlanarGraph.cs
src/TB.Core/Geometry/PlanarGraphBuilder.cs
tests/TB.Core.Tests/Geometry/PlanarGraphBuilderTests.cs
```

### 必须测试

```text
GraphBuilder_CreatesTwoHalfEdgesPerEdge
GraphBuilder_AssignsTwinIds
GraphBuilder_SortsOutgoingByAngle
GraphBuilder_MergesOnlyEpsilonCloseVertices
GraphBuilder_PreservesBridgeEdgeKind
```

### 验证命令

```bash
dotnet test --filter PlanarGraph
dotnet test
```

### 完成标准

- 每条 Edge 两条 HalfEdge；
- outgoing 按角度排序；
- 所有 tie-break 使用确定性规则。

---

## Phase C8 — FaceTracer

### 目标

从 DCEL 中遍历所有面，并给边登记左右面。

### 产出文件

```text
src/TB.Core/Model/Face.cs
src/TB.Core/Geometry/FaceTracer.cs
tests/TB.Core.Tests/Geometry/FaceTracerTests.cs
```

### 必须测试

```text
FaceTracer_Rectangle_FindsOneBoundedFaceAndOneUnboundedFace
FaceTracer_TwoAdjacentRectangles_FindsTwoBoundedFaces
FaceTracer_Grid2x2_FindsFourSmallBoundedFaces
FaceTracer_AssignsFaceIdToHalfEdges
FaceTracer_OpenCross_ReturnsNoBoundedFace
```

### 验证命令

```bash
dotnet test --filter FaceTracer
dotnet test
```

### 完成标准

- 不能输出为最终轮廓；
- 只是为 FaceClassifier 和 OutlineBuilder 提供面数据；
- 无界面识别有测试。

---

## Phase C9 — Polylabel / FaceClassifier 路线 B

### 目标

实现路线 B 的孔洞/填充分类。

### 产出文件

```text
src/TB.Core/Geometry/Polylabel.cs
src/TB.Core/Geometry/FaceClassifier.cs
tests/TB.Core.Tests/Geometry/PolylabelTests.cs
tests/TB.Core.Tests/Geometry/FaceClassifierTests.cs
```

### 必须测试

```text
Polylabel_Rectangle_ReturnsHalfOfShortSide
Polylabel_NarrowCorridor_ReturnsSmallRadius
Polylabel_PointOutsidePolygon_HasNegativeDistance
FaceClassifier_LargeRectangleClassifiedVoid
FaceClassifier_SmallGridCellClassifiedSolid
FaceClassifier_ChangingClosingRadiusChangesClassification
UnboundedFace_ClassifiedUnboundedVoid
```

### 验证命令

```bash
dotnet test --filter "Polylabel|FaceClassifier"
dotnet test
```

### 完成标准

- `ClosingRadius` 能改变分类；
- 不需要精确中轴，但结果必须满足测试；
- `FaceKind` 只由 FaceClassifier 写入。

---

## Phase C10 — OutlineBuilder

### 目标

提取最终外轮廓 + 孔洞环。

### 产出文件

```text
src/TB.Core/Geometry/OutlineBuilder.cs
tests/TB.Core.Tests/Geometry/OutlineBuilderTests.cs
```

### 必须测试

```text
OutlineBuilder_Rectangle_ReturnsOneOuterLoop
OutlineBuilder_TwoSeparateRectangles_ReturnsTwoOuterLoops
OutlineBuilder_Grid2x2_ReturnsOnlyOuterLoop
OutlineBuilder_Donut_ReturnsOuterAndHole
OutlineBuilder_DiscardsOpenChain
OutlineBuilder_KeepsBridgeEdgeWhenNeededToCloseGap
```

### 验证命令

```bash
dotnet test --filter OutlineBuilder
dotnet test
```

### 完成标准

- 稠密网格只输出外轮廓；
- donut 输出外轮廓 + 孔洞；
- 不输出所有 face。

---

## Phase C11 — LoopRebuilder

### 目标

将 RawLoop 转成包含 line/bulge 的 BoundaryLoop。

### 产出文件

```text
src/TB.Core/Geometry/LoopRebuilder.cs
tests/TB.Core.Tests/Geometry/LoopRebuilderTests.cs
```

### 必须测试

```text
LoopRebuilder_LineRectangle_ReturnsFourLineSegments
LoopRebuilder_Circle_ReturnsArcSegmentsWithBulge
LoopRebuilder_ReversedArc_UsesNegativeBulge
LoopRebuilder_BridgeEdge_ReturnsLineSegment
```

### 验证命令

```bash
dotnet test --filter LoopRebuilder
dotnet test
```

### 完成标准

- 圆弧尽量还原为 bulge；
- bridge 输出为直线；
- 孔洞方向 CW，外轮廓方向 CCW。

---

## Phase C12 — Pipeline 端到端

### 目标

串联 Core 全流程，得到可使用的基础版 Core MVP。

### 产出文件

```text
src/TB.Core/Pipeline/OutlineExtractorService.cs
tests/TB.Core.Tests/Pipeline/OutlineExtractorPipelineTests.cs
```

### 必须测试

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

### 验证命令

```bash
dotnet test --filter Pipeline
dotnet test
```

### 完成标准

Core MVP 完成。此阶段完成前不进入 AutoCAD 集成。

---

## 4. AutoCAD 集成阶段

## Phase A0 — AutoCAD 项目骨架

### 目标

创建 `TB.AutoCAD`，但只做最小命令加载验证。

### 产出文件

```text
src/TB.AutoCAD/TB.AutoCAD.csproj
src/TB.AutoCAD/Commands/TotalBoundaryCmd.cs
bundle/PackageContents.xml
```

### 验证

如果有 AutoCAD:

```text
NETLOAD TB.AutoCAD.dll
执行 TBABOUT 或 TOTALBOUNDARY
```

如果有 accoreconsole:

```bash
accoreconsole.exe /i test.dwg /s scripts/smoke.scr
```

### 完成标准

- DLL 可加载；
- 命令能输出版本信息；
- 不执行复杂几何逻辑。

---

## Phase A1 — EntityExtractor

### 目标

将 AutoCAD 实体转成 `InputCurve[]`。

### 支持顺序

1. Line；
2. Arc；
3. Circle；
4. Polyline 直线段；
5. Polyline bulge 段；
6. BlockReference 单层；
7. Ellipse / Spline 诊断占位。

### 验证

使用简单 DWG 或命令内统计:

```text
矩形 4 Line -> InputCurve count = 4
Circle -> 2 arc curves
Polyline bulge -> 包含 Arc curve
```

---

## Phase A2 — PolylineWriter

### 目标

把 `BoundaryLoop[]` 写成 AutoCAD `Polyline`。

### 验证

```text
手工构造 BoundaryLoop -> 写入 1 条 Closed Polyline
带 bulge loop -> AutoCAD 中显示圆弧
Undo 一次可撤销全部输出
```

---

## Phase A3 — TOTALBOUNDARY / TB 主命令

### 目标

选对象 → EntityExtractor → Core Pipeline → PolylineWriter。

### 验证图形

```text
4 Line 矩形
带小缝矩形
Circle
2x2 网格
donut
```

### 完成标准

- 输出符合 Core 端到端预期；
- 出错事务回滚；
- Editor 输出统计信息。

---

## Phase A4 — TBALL 与 TBPICK

### 目标

实现全图轮廓和点选片段轮廓。

### 验证

```text
TBALL 当前空间简单图 -> 输出所有片段轮廓
TBPICK 两个矩形中点选一个 -> 只输出该片段
```

---

## 5. 测试图形坐标规范

AI 写 Core 单测时优先使用以下坐标，保证可复现。

### 5.1 Rectangle

```text
(0,0) -> (10,0)
(10,0) -> (10,5)
(10,5) -> (0,5)
(0,5) -> (0,0)
Expected: 1 outer loop
```

### 5.2 GappedRectangle

```text
(0,0) -> (9.9,0)
(10,0) -> (10,5)
(10,5) -> (0,5)
(0,5) -> (0,0)
gap = 0.1
GapTolerance = 0.2 => 1 outer loop
GapTolerance = 0.05 => 0 loop
```

### 5.3 Circle

```text
center = (0,0)
radius = 5
Expected: 1 outer loop with arc/bulge segments
```

### 5.4 Grid2x2

外框:

```text
(0,0)-(10,0)-(10,10)-(0,10)-(0,0)
```

内部线:

```text
(5,0)->(5,10)
(0,5)->(10,5)
```

参数:

```text
ClosingRadius = 4
Expected: only 1 outer loop, no four cell loops
```

### 5.5 Donut

外框:

```text
(0,0)-(20,0)-(20,20)-(0,20)-(0,0)
```

内空腔框:

```text
(7,7)-(13,7)-(13,13)-(7,13)-(7,7)
```

连接/填充线可用一组狭长小格模拟实心带。

Expected:

```text
1 outer loop + 1 hole loop
```

### 5.6 OpenCross

```text
(0,0)->(10,10)
(0,10)->(10,0)
Expected: 0 loops
```

### 5.7 TwoRectangles

Rectangle A:

```text
(0,0)-(5,0)-(5,5)-(0,5)-(0,0)
```

Rectangle B:

```text
(10,0)-(15,0)-(15,5)-(10,5)-(10,0)
```

Expected:

```text
2 outer loops
```

---

## 6. 每轮 AI 任务提示模板

后续每次让 AI 编码时，可使用以下模板。

```text
请按 docs/totalboundary-ai-phased-execution-plan.md 执行 Phase Cx。

要求:
1. 只实现本 Phase，不做后续 Phase。
2. 不改变 TotalBoundary 外轮廓 + 孔洞语义。
3. TB.Core 不得引用 AutoCAD DLL。
4. 为本 Phase 增加文档指定的单元测试。
5. 运行 dotnet test；如果环境无法运行，说明原因。
6. 提交并推送，提交信息使用计划中的格式。
```

---

## 7. 阶段准入/退出表

| 阶段 | 进入条件 | 退出条件 |
|------|----------|----------|
| C0 | 仓库干净 | 解决方案和测试骨架可运行 |
| C1 | C0 通过 | 基础 DTO 测试通过 |
| C2 | C1 通过 | RobustPredicates 测试通过 |
| C3 | C2 通过 | Tessellator 测试通过 |
| C4 | C3 通过 | UniformGrid 测试通过 |
| C5 | C4 通过 | SegmentIntersector 测试通过 |
| C6 | C5 通过 | GapBridger 测试通过 |
| C7 | C6 通过 | PlanarGraphBuilder 测试通过 |
| C8 | C7 通过 | FaceTracer 测试通过 |
| C9 | C8 通过 | Polylabel / FaceClassifier 测试通过 |
| C10 | C9 通过 | OutlineBuilder 测试通过 |
| C11 | C10 通过 | LoopRebuilder 测试通过 |
| C12 | C11 通过 | Pipeline 端到端测试通过 |
| A0 | C12 通过且有 AutoCAD SDK | DLL 可加载 |
| A1 | A0 通过 | EntityExtractor 验证通过 |
| A2 | A1 通过 | PolylineWriter 验证通过 |
| A3 | A2 通过 | TOTALBOUNDARY 验证通过 |
| A4 | A3 通过 | TBALL/TBPICK 验证通过 |

---

## 8. 当前建议的第一条编码任务

第一条实际编码任务应为:

```text
执行 Phase E0 + Phase C0。

目标:
- 检查 .NET / AutoCAD SDK 环境。
- 创建 TB.sln、TB.Core、TB.Core.Tests。
- 添加 smoke test。
- 运行 dotnet test。
- 提交并推送。
```

不要一开始就写几何算法。先保证工程和测试循环稳定。

