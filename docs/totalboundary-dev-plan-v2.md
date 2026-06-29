# TotalBoundary Clone — AutoCAD .NET 插件开发方案

> 版本: **v2.1** | 日期: 2026-06-29 | 状态: 待确认
>
> **v2.1 修订要点（核心目标纠偏）**：
> - 🔴 **纠正核心目标**：经核实 TotalBoundary 的招牌能力是**提取选定对象的"外轮廓 / 周界"（并集剪影 + 内部孔洞）**，而非"提取所有最小内部面"（后者是姊妹产品 SuperBoundary / 原生 BOUNDARY 的语义）。本版本将核心算法的最终阶段从"输出每个面"改正为"**内/外面分类 + 轮廓提取**"。
> - 🔴 **空腔判定采用路线 B（形态学闭合 / alpha-shape 风格）**：以闭合半径 `ClosingRadius` 控制，空旷区域超阈值即成孔洞，细小网格单元被填充覆盖。
> - 🔴 **命令体系**改以 `TOTALBOUNDARY`（选对象 → 出轮廓）为主命令，对齐真实产品。
> - 🔴 **性能基准上调**：真实标杆为**上万对象级 / 秒级**（如 ~12500 对象约 4–5 秒）。
> - 🟢 SuperBoundary 式"全区域提取"明确**不在本次复刻范围**（可作为未来可选模式）。
>
> **v2.0 既有修订（保留有效）**：
> - 多目标构建（net40 + net8.0-windows），最低版本 **AutoCAD 2013**；主线程 STA + 纯几何层并行；线段相交打断（Bentley–Ottmann）；圆弧 **bulge** 保留；Autoloader Bundle 多版本加载；术语表/数据模型/NFR/验收标准/构建矩阵/可测试性。

---

## 一、项目概述

复刻 AutoCAD 知名插件 **TotalBoundary（Debalance Research Group）** 的核心功能，使用 AutoCAD .NET API 自主开发一个**轮廓（Outline）提取插件**。采用**分级产品策略**，分三阶段交付，每阶段可独立编译和部署。

### 产品定位（务必锚定，勿跑偏）
TotalBoundary 是**轮廓线生成器**：对用户选定的一组对象，沿其**外周界**生成闭合多段线（"draws enclosed polylines around the outside of selected objects"），并在内部存在较大空腔时生成相应的**孔洞**边界（"one outside, one inside"）。它**不是**逐个内部面的提取工具。

> 区分:**TotalBoundary = 外轮廓/剪影**；**SuperBoundary = BOUNDARY 替代 / 全区域**。本方案复刻前者。

### 核心能力
- 对选定对象集合提取**外轮廓 + 内部孔洞**的闭合多段线
- 支持 Line、Arc、Circle、Ellipse、Spline、Polyline、BlockReference 等实体类型
- 强大的**间隙容差（Gap Tolerance）**处理能力：自动桥接近似相接处的缝隙，远超原生 `BOUNDARY` / `BPOLY`
- 正确处理**线段中段相交**（X/T 交叉），而不仅是端点连接
- 用**闭合半径（ClosingRadius，路线 B）**区分"应填充的细小单元"与"应保留的内部孔洞"
- 无需预清理：自动忽略多余的悬挂线（开放链不参与轮廓）
- 样条/椭圆**分段线性化**，圆弧用 **bulge** 精确保留；可设线宽/颜色/图层
- 可选 **Solid 填充**（TotalBoundary 宣传特性，本方案放在 Pro，基础版预留接口）

### 与原生命令的本质差异
| 能力 | 原生 BOUNDARY/BPOLY | 本插件（TotalBoundary 复刻）|
|------|--------------------|--------|
| 目标 | 点选点所在的单个封闭区域 | 选定对象集合的**整体外轮廓 + 孔洞** |
| 间隙桥接 | 几乎不容忍 | 可配置容差，主动桥接 |
| 中段交叉处理 | 依赖屏幕显示精度 | 显式求交打断 |
| 大图性能 | 复杂图缓慢/失败 | 上万对象秒级 |
| 预清理 | 常需手动加包围框/清理 | 无需，自动忽略悬挂线 |
| 容差/细节可控 | 黑盒 | 全参数化（GapTolerance / ClosingRadius / 采样）|

---

## 二、技术栈与兼容性

| 项目 | 规格 |
|------|------|
| 语言 | C#（多目标编译）|
| 目标框架 (TFM) | `net40`（AutoCAD 2013–2024）+ `net8.0-windows`（AutoCAD 2025–2026）|
| AutoCAD API | AutoCAD .NET API (`acdbmgd.dll` + `acmgd.dll` + `accoremgd.dll`) |
| **最低版本** | **AutoCAD 2013 (AC19.0, .NET 4.0)** |
| 最高版本 | AutoCAD 2026 (AC25.x, .NET 8) |
| 平台 | x64（AutoCAD 2015+ 仅 64 位；如需支持 2013–2014 附带 x86）|
| 构建工具 | Visual Studio 2022 / MSBuild，SDK 风格项目 |
| 加载机制 | **Autoloader Bundle**（`.bundle` + `PackageContents.xml`）+ 按需加载 |
| 打包方式 | WiX MSI（职责：把 bundle 部署到 ApplicationPlugins）|
| 线程模型 | **主线程 STA**；纯几何计算可工作线程并行，**禁止跨线程访问 DBObject** |

### 2.1 AutoCAD 版本 ↔ 运行时 ↔ 构建矩阵

| AutoCAD 版本 | 内部版本号 | 绑定运行时 | 本插件 TFM | 引用 SDK |
|------|------|------|------|------|
| 2013–2014 | 19.0 / 19.1 | .NET 4.0 / 4.5 | `net40` | ObjectARX 2013 |
| 2015–2016 | 20.0 / 20.1 | .NET 4.5 | `net40` | ObjectARX 2013 |
| 2017 | 21.0 | .NET 4.6 | `net40` | ObjectARX 2013 |
| 2018–2020 | 22.0 / 23.0 / 23.1 | .NET 4.7 | `net40` | ObjectARX 2013 |
| 2021–2024 | 24.0–24.3 | .NET 4.8 | `net40` | ObjectARX 2013 |
| **2025–2026** | **25.x** | **.NET 8** | **`net8.0-windows`** | ObjectARX 2025 |

> **设计原则**：.NET Framework 一侧统一以 `net40` + **AutoCAD 2013 ObjectARX** 编译。由于 .NET 4.x 系列共享同一 CLR4 且 AutoCAD API 在该区间向后兼容，单套 net40 二进制即可运行于 2013–2024。AutoCAD 2025 迁移到 .NET 8，二进制不兼容，必须单独出 `net8.0-windows` 构建。
>
> 引用程序集统一设置 **Copy Local = false**、**Specific Version = false**，避免把 ObjectARX DLL 误打包。

### 2.2 多目标项目配置示例

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFrameworks>net40;net8.0-windows</TargetFrameworks>
    <UseWindowsForms>true</UseWindowsForms>
    <Platforms>x64</Platforms>
    <LangVersion>latest</LangVersion>
  </PropertyGroup>

  <!-- AutoCAD 2013–2024 -->
  <ItemGroup Condition="'$(TargetFramework)' == 'net40'">
    <Reference Include="acmgd"><HintPath>$(ACAD2013)\acmgd.dll</HintPath>
      <Private>false</Private><SpecificVersion>false</SpecificVersion></Reference>
    <Reference Include="acdbmgd"><HintPath>$(ACAD2013)\acdbmgd.dll</HintPath>
      <Private>false</Private><SpecificVersion>false</SpecificVersion></Reference>
  </ItemGroup>

  <!-- AutoCAD 2025–2026 -->
  <ItemGroup Condition="'$(TargetFramework)' == 'net8.0-windows'">
    <Reference Include="acmgd"><HintPath>$(ACAD2025)\acmgd.dll</HintPath>
      <Private>false</Private><SpecificVersion>false</SpecificVersion></Reference>
    <Reference Include="acdbmgd"><HintPath>$(ACAD2025)\acdbmgd.dll</HintPath>
      <Private>false</Private><SpecificVersion>false</SpecificVersion></Reference>
  </ItemGroup>
</Project>
```

### 2.3 运行时版本检测（正确写法）

```csharp
// Application.VersionNumber 形如 "24.3s (LMS Tech)"，不能直接做字符串比较
var raw = Application.VersionNumber;
var token = System.Text.RegularExpressions.Regex.Match(raw, @"^\d+\.\d+").Value;
var ver = Version.Parse(token);          // 例: 24.3 / 19.0 / 25.0

bool isCore2025Plus = ver >= new Version(25, 0);   // .NET 8 线
bool hasFeatureX     = ver >= new Version(21, 0);  // 21.0 = AutoCAD 2017
// 注意：23.0 = AutoCAD 2019（非 2017），编号需按官方对照
```

---

## 三、Hatch.GenerateBoundary 评估结论

### 问题
`Hatch.GenerateBoundary` 是 AutoCAD 2013+ 引入的内部 API，可从填充图案反向提取边界。

### 为什么不将其作为核心算法？

| 维度 | 评估 |
|------|------|
| **稳定性** | 对间隙极其敏感，微小缝隙即失败 |
| **性能** | 需先创建 Hatch 再提取，开销大 |
| **可控性** | 黑盒 API，无法精细控制容差和提取逻辑 |
| **可移植性** | 行为随版本变化，跨 2013–2026 一致性无保证 |

### 最终决策
**不依赖 `Hatch.GenerateBoundary` 作为核心算法。** 仅在 Pro+ 版本中作为**辅助验证手段**：当自研算法提取的边界与 `Hatch.GenerateBoundary` 结果一致时，提高置信度标记。由于最低版本已为 2013，该 API 在全版本区间均可作为可选验证。

---

## 四、分级产品架构

### Tier 1 — 基础版 (Foundation)

> 最小可行产品 (MVP)，解决核心痛点：**对选定对象提取外轮廓 + 孔洞闭合多段线**

| 模块 | 功能描述 |
|------|----------|
| 实体分析器 | 识别并分类选择集中的所有实体类型 (Line, Arc, Circle, Ellipse, Spline, Polyline, BlockReference) |
| 预处理/清理 | 去重、删除零长线段、合并重叠共线段（类 OVERKILL）|
| 间隙桥接器 | 基于空间索引计算端点近邻距离，≤ 容差则插入桥接边 |
| 相交求解器 | Bentley–Ottmann 扫描线求线段交点并在交点处打断 |
| 平面图构建器 | 顶点/边邻接结构（半边/DCEL），记录边方向 |
| 面遍历器 | 极角排序 + next-edge 面遍历，得到所有面（含无界外面）|
| **面分类器（路线 B）** | 按闭合半径将面分为 Solid（填充）/ Void（孔洞）；无界面恒为外部 |
| **轮廓提取器** | 取"实边且非两侧皆 Solid"的边组装为闭合轮廓环（外轮廓 + 孔洞）|
| 多段线构建器 | 轮廓环 → ClosedPolyline；**圆弧用 bulge**，仅样条/椭圆自适应线性化 |
| 命令集 | `TOTALBOUNDARY`/`TB`(选对象出轮廓)、`TBALL`(全图)、`TBPICK`(点选片段轮廓) |
| 属性设置 | 颜色、线宽、图层分配 |
| 进度指示 | 大型图纸处理时显示进度条，支持 Esc 中断 |

> 说明：SuperBoundary 式"逐个内部面全提取"不在基础版范围；面分类器若令所有有界面=Solid，则退化为外轮廓，可作为该模式开关的预留。

### Tier 2 — 升级版 (Pro)

| 模块 | 功能描述 |
|------|----------|
| 面积/周长计算 | 计算每个边界面积和周长，**孔洞面积自动扣除**，输出到选项板或日志 |
| 实体填充 (Solid Hatch) | 为提取的边界生成 Solid Hatch（正确处理岛屿）|
| DXF 导出 | 将提取的边界导出为独立 DXF 文件 |
| 批量处理 | 多区域一次性提取 |
| INI 配置文件 | 持久化用户偏好（容差、采样、图层规则等）|
| 图层智能管理 | 按原始实体类型自动分类到新图层 |

### Tier 3 — 旗舰版 (Pro Plus)

| 模块 | 功能描述 |
|------|----------|
| GUI 配置对话框 | 可视化参数面板（容差滑块、图层映射表、快捷键设置）|
| 递归块扩展 | 自动展开 BlockReference 内部几何（含 MInsert/嵌套，深度限制）|
| GenerateBoundary 验证 | 与自研结果交叉比对，标记置信度 |
| 性能监控面板 | 实时显示耗时、内存、提取效率 |
| 撤销/重做集成 | 与 AutoCAD 事务系统深度集成 |
| 可卸载安装程序 | WiX MSI，一键安装/卸载，部署 bundle |
| 许可证系统 | 可选许可证验证框架（为商业化预留）|

---

## 五、项目目录结构

```
TotalBoundaryClone/
├── TB.sln                          # Visual Studio 解决方案
├── Directory.Build.props           # 统一 TFM、平台、ObjectARX 路径变量
├── TB.Core/                        # 纯算法层 (无 CAD 依赖, 100% 可单测)
│   ├── Model/
│   │   ├── Segment.cs              # 统一线段抽象 (直线/圆弧/样条三态)
│   │   ├── Vertex.cs / Edge.cs     # 平面图基本元素 (实边/桥接边)
│   │   ├── Face.cs                 # 平面细分面 (Solid/Void 分类)
│   │   └── BoundaryLoop.cs         # 轮廓环 (顶点序列 + bulge + 外轮廓/孔)
│   ├── Geometry/
│   │   ├── SpatialIndex.cs         # 网格/R-tree 空间索引
│   │   ├── GapBridger.cs           # 近邻端点 + 容差桥接边
│   │   ├── SegmentIntersector.cs   # Bentley–Ottmann 求交并打断
│   │   ├── RobustPredicates.cs     # 浮点健壮判定 (共线/朝向)
│   │   ├── PlanarGraphBuilder.cs   # 半边(DCEL) + 边方向
│   │   ├── FaceTracer.cs           # 极角排序 + next-edge 面遍历(得到所有面)
│   │   ├── FaceClassifier.cs       # 路线 B: Solid/Void 分类(闭合半径)
│   │   ├── OutlineExtractor.cs     # 轮廓边组装为外轮廓+孔洞环
│   │   └── Linearizer.cs           # Spline/Ellipse 自适应线性化 (矢高控制)
│   ├── Output/
│   │   └── PolylineSpec.cs         # 与 CAD 无关的多段线规格 (含 bulge)
│   └── TB.Core.csproj              # 多目标 net40;net8.0-windows
├── TB.AutoCAD/                     # CAD 集成层 (薄适配器)
│   ├── Commands/
│   │   ├── TotalBoundaryCmd.cs     # TOTALBOUNDARY/TB — 选对象出轮廓(主命令)
│   │   ├── TBAll.cs                # TBALL — 全图轮廓
│   │   └── TBPick.cs               # TBPICK — 点选片段轮廓
│   ├── Services/
│   │   ├── EntityExtractor.cs      # DBObject → TB.Core.Segment (主线程)
│   │   ├── PolylineWriter.cs       # PolylineSpec → DBObject (主线程)
│   │   ├── SelectionService.cs     # 选择集管理
│   │   ├── LayerService.cs         # 图层操作
│   │   ├── UcsService.cs           # UCS↔WCS 变换
│   │   └── ProgressService.cs      # 进度指示 + Esc 中断
│   ├── Config/{INIConfig.cs, Settings.cs}
│   ├── Diagnostics/Logger.cs       # 结构化日志
│   └── TB.AutoCAD.csproj
├── TB.Pro/                         # Pro 功能层
│   ├── AreaCalculator.cs / HatchGenerator.cs
│   ├── DxfExporter.cs / BatchProcessor.cs
│   └── TB.Pro.csproj
├── TB.ProPlus/                     # ProPlus 功能层
│   ├── ConfigDialog.cs / BlockExpander.cs
│   ├── GenerateBoundaryValidator.cs
│   ├── PerformanceMonitor.cs / TransactionManager.cs
│   └── TB.ProPlus.csproj
├── bundle/                         # Autoloader 资源
│   └── PackageContents.xml         # 多版本运行时映射
├── TB.Installer/                   # WiX 安装包 (部署 bundle)
│   └── TB.Installer.wixproj
├── TB.Tests/                       # 单元 + 集成测试
│   ├── CoreTests/                  # 针对 TB.Core (xUnit, 主力)
│   └── IntegrationTests/           # accoreconsole 脚本化
└── README.md
```

---

## 六、核心算法设计

### 6.1 整体流程（修订）

```
用户选择实体集合
    │
    ▼
[主线程] EntityExtractor ── 解析 DBObject → 纯 POCO Segment (含 UCS→WCS)
    │
    ▼
[可并行] Preprocess ──────── 去重 / 删零长 / 合并共线 / 忽略孤立悬挂线
    │
    ▼
[可并行] Tessellator ─────── 曲线→直线边(带 provenance, 圆弧记圆心便于还原)
    │
    ▼
[可并行] SegmentIntersector ─ Bentley–Ottmann 求交，在交点处打断
    │
    ▼
[可并行] GapBridger ───────── 空间索引近邻 + 容差插入桥接边(闭合缝隙)
    │
    ▼
[可并行] PlanarGraphBuilder ─ 半边(DCEL) + 各顶点出边极角排序
    │
    ▼
[可并行] FaceTracer ───────── next-edge 面遍历 → 所有面(含无界外面)
    │
    ▼
[可并行] FaceClassifier ───── ★路线 B: 每个有界面按闭合半径分 Solid/Void
    │
    ▼
[可并行] OutlineExtractor ─── ★取"实边且非两侧皆 Solid"的边 → 外轮廓 + 孔洞环
    │
    ▼
[可并行] LoopRebuilder ────── 凭 provenance 还原 bulge 圆弧
    │
    ▼
[主线程] PolylineWriter ───── → ClosedPolyline (圆弧 bulge), 反变换回 WCS
    │
    ▼
   输出结果(少数几条轮廓多段线)
```

> 关键：CAD 对象的读写都在主线程 (STA)；中间无 CAD 依赖的纯几何步骤可用 PLINQ 并行。
> **与 v2.0 的区别**：最终阶段由"输出每个面"改为"**面分类 + 轮廓提取**"，对齐 TotalBoundary 的外轮廓语义。

### 6.2 关键算法细节

#### 预处理 (Preprocess)
- 删除零长线段、完全重复实体；合并重叠共线段，避免平面图退化；
- **忽略孤立悬挂线**（不参与任何面的开放链），对应"无需预清理"特性；
- 等价于轻量级 OVERKILL，是健壮性的第一道防线。

#### 相交求解 (SegmentIntersector)
- **Bentley–Ottmann 扫描线**求所有线段交点，复杂度 O((n+k) log n)，k 为交点数；
- 在交点处打断线段，生成平面细分的边；展平后只需处理"线段-线段"求交。

#### 间隙桥接 (GapBridger)
- 用**空间索引**（网格）做端点近邻查询，避免 O(n²)；
- 端点距离 ≤ `GapTolerance` → 插入一条**桥接边**（标记为虚拟边，区别于实体边）；
- 桥接使近似相接的线工连通，是"间隙容差"特性的实现。容差默认 0.001，用户可调。

#### 面遍历 (FaceTracer)
- 在每个顶点把出边按**极角排序**；进入一条边到达对端后，沿"紧邻反向边的顺时针下一条出边"前进，遍历出**平面图的所有面**（含最外无界面）。
- 用有符号面积区分朝向并识别无界外面。该算法是平面细分面提取的标准做法（非 Hierholzer 欧拉回路）。

#### ★面分类 — 路线 B (FaceClassifier)
> TotalBoundary 的"外轮廓 + 孔洞"语义 = **对线工做形态学闭合（半径 r = ClosingRadius）后的边界**。等价地，逐面分类：
- **无界外面** → 恒为 `Void`（外部）。
- **有界面 F**：计算其**最大内切空圆半径** ρ(F)（面内不触及任何边的最大空圆）。
  - ρ(F) > r → `Void`（真正的内部空腔/孔洞，应保留为孔）；
  - ρ(F) ≤ r → `Solid`（细小单元/狭长缝，被闭合填充覆盖）。
- ρ(F) 可由面的距离变换采样或内切圆近似求得；r 默认取 `GapTolerance` 的若干倍，用户可调（控制轮廓"细节/平滑度"）。
- **要点**：简单封闭矩形(4 线)其内部面为 Void、外部亦 Void，但其 4 条实体边两侧皆 Void→均被保留→输出即矩形本身（见下条边规则），不会丢失。

#### ★轮廓提取 (OutlineExtractor)
- 逐条**实体边**(非桥接虚拟边)判定：**当且仅当其两侧的面都为 Solid 时丢弃**（属实心内部），否则保留（在轮廓上）。
  - 实心稠密网格的内部边 → 两侧 Solid → 丢弃，仅留外周界；
  - 矩形/孤立闭环的边 → 两侧 Void → 保留 → 输出该环；
  - 实心与空腔之间的边 → 保留 → 形成**孔洞**边界。
- 桥接虚拟边仅在闭合轮廓必需处保留。
- 将保留边按拓扑组装为闭合环：外轮廓(CCW) + 孔洞(CW)，可含多个不连通片段各自的外轮廓。

#### 点选提取 (TBPICK)
- 拾取一点 → 定位其所在的**连通线工片段** → 仅对该片段提取轮廓（外轮廓+孔洞）。
- 注：这是"对某片段出轮廓"，**非** BPOLY 的"该点所在单个封闭面"（后者属 SuperBoundary 语义，不在范围）。

#### 多段线构建 (LoopRebuilder + PolylineWriter)
- **Arc / Circle → bulge 段，精确无损**（凭 provenance 还原，不线性化）；
- Spline / Ellipse → 按**矢高/弦高 (sagitta) 自适应**细分为折线，用户可控最大弦高与最大角度；
- 可选输出 `Region` 对象。

---

## 七、数据模型与术语表

| 术语 | 定义 |
|------|------|
| 轮廓 (Outline) | 选定对象集合的外周界 + 内部孔洞闭合多段线，本插件的最终产物 |
| `Segment` | 几何线段的统一抽象，三态：直线段 / 圆弧段（含 bulge）/ 样条段；不含任何 CAD 类型 |
| `Edge` | 平面图的边；实体边(来自对象)或桥接边(容差填缝的虚拟边) |
| 面 (Face) | 平面细分的区域；分类为 Solid(实心/填充) 或 Void(空腔/外部) |
| Solid / Void | 路线 B 的面分类：内切空圆 ≤ r 为 Solid，> r 为 Void |
| `BoundaryLoop` | 一条闭合轮廓环：顶点序列 + 每段 bulge + 朝向；外轮廓 CCW / 孔洞 CW |
| 间隙容差 (GapTolerance) | 端点视为连接、插入桥接边的最大缝隙距离 |
| 闭合半径 (ClosingRadius, r) | 路线 B 的形态学闭合半径，区分"填充细单元"与"保留空腔" |
| 矢高 (Sagitta) | 弦与弧之间的最大垂距，控制样条/椭圆线性化精度 |

---

## 八、开发计划与里程碑

> 以**技术里程碑 + 可验收关卡**衡量进度（几何健壮性是主要不确定性来源），不绑定固定日历周期。

### Phase 1 — 基础版 (Foundation)

| 步骤 | 任务 | 验收关卡 (DoD) |
|------|------|------|
| 1 | VS 解决方案 + 多目标骨架 | net40/net8 两套二进制均可编译 |
| 2 | Segment 抽象 + EntityExtractor | 各实体类型正确转 POCO，UCS 变换正确 |
| 3 | Preprocess | 含重复/零长样例清理后无退化 |
| 4 | SpatialIndex + GapDetector | 0.5mm 间隙样例可桥接 |
| 5 | SegmentIntersector | 含 1 个十字交叉样例正确打断 |
| 6 | PlanarGraphBuilder + FaceTracer | 矩形→1 面；相邻两矩形→3 面 |
| 7 | FaceClassifier（路线 B）+ OutlineExtractor | 稠密网格→仅外轮廓；donut→外轮廓+1 孔 |
| 8 | LoopRebuilder + PolylineWriter（bulge）| 含圆弧样例输出无失真 |
| 9 | 三命令 (TOTALBOUNDARY/TBALL/TBPICK) | 三命令可用，点选片段轮廓准确 |
| 10 | 属性设置 + 进度/中断 | 大图可显示进度且可 Esc 取消 |
| 11 | 对比验证 | 选定片段轮廓与人工预期一致；间隙图优于原生 |

### Phase 2 — 升级版 (Pro)

| 步骤 | 任务 | 验收关卡 |
|------|------|------|
| 1 | AreaCalculator（含孔洞扣除）| 带孔多边形面积 = 外环 − 孔 |
| 2 | HatchGenerator（岛屿）| Solid Hatch 正确避开内孔 |
| 3 | DxfExporter | 导出 DXF 可被 AutoCAD 重新打开 |
| 4 | BatchProcessor | 多区域一次提取无遗漏 |
| 5 | INI 配置持久化 | 重启后偏好保留 |

### Phase 3 — 旗舰版 (Pro Plus)

| 步骤 | 任务 | 验收关卡 |
|------|------|------|
| 1 | ConfigDialog | 参数改动即时生效 |
| 2 | BlockExpander | 嵌套块几何参与计算，深度受限 |
| 3 | GenerateBoundaryValidator | 一致时标记高置信度 |
| 4 | PerformanceMonitor | 实时显示耗时/内存 |
| 5 | TransactionManager | 标准 Undo/Redo 可回滚 |
| 6 | WiX 安装包 | 在 2013–2026 任一版本安装即加载 |

---

## 九、测试计划

### 单元测试（TB.Core，主力，可在无 AutoCAD 环境运行）
- [ ] Preprocess：重复 / 零长 / 共线重叠 / 孤立悬挂线被忽略
- [ ] GapBridger：零间隙 / 小间隙(桥接) / 大间隙(不桥接)
- [ ] SegmentIntersector：X 交叉 / T 搭接 / 多重共点 / 端点重合
- [ ] FaceTracer：矩形→1 面 / 相邻矩形→共享边 / 无界面识别
- [ ] FaceClassifier（路线 B）：大空腔=Void / 小单元=Solid / 闭合半径敏感性
- [ ] OutlineExtractor：稠密网格→外轮廓 / donut→外轮廓+孔 / 多片段→多外轮廓
- [ ] LoopRebuilder：纯直线 / 含圆弧(bulge) / 含样条(线性化) / 混合
- [ ] RobustPredicates：近共线 / 近重合的浮点边界

### 集成测试（accoreconsole.exe 脚本化，无界面）
- [ ] 矩形 (4 Line) → 1 条外轮廓多段线
- [ ] 圆 (Circle) → 1 条外轮廓（bulge 表示）
- [ ] 带 0.5mm 间隙封闭图形 → 容差内自动闭合出轮廓
- [ ] 稠密网格(# 形 N×N) → 仅 1 条外周界(内部边不输出)
- [ ] donut(外框内挖空腔) → 外轮廓 + 1 孔洞
- [ ] 复杂工程图 (10000+ 实体) → ≤ 5 秒，内存可控

### 对比验证
- [ ] 与原生 `BOUNDARY` / `BPOLY` 结果对比
- [ ] 间隙容差处理能力对比
- [ ] 多版本冒烟：2013 / 2018 / 2024 / 2025 各加载并跑通核心命令

> **可测试性说明**：ObjectARX/RealDWG 托管类型难以 mock，因此**业务逻辑全部下沉 `TB.Core`**（纯 POCO，单测覆盖主力）；`TB.AutoCAD` 仅做薄适配器，集成测试用 CoreConsole 脚本（或 RealDWG，需单独授权）。

---

## 十、风险与缓解

| 风险 | 影响 | 缓解措施 |
|------|------|----------|
| **多运行时兼容** | 2025 .NET 8 与旧版不互通 | 多目标构建 + Autoloader 按版本加载 |
| **浮点健壮性** | 共线/交点误判致错环 | RobustPredicates + 容差统一管理 + 测试覆盖退化情形 |
| 跨线程访问 DBObject | 崩溃/数据损坏 | 严格主线程读写，仅 Core 并行 |
| Spline 线性化精度 | 边界失真 | 矢高自适应采样 + 用户可控；圆弧用 bulge 不线性化 |
| 大型图纸性能 | 处理超时 | 空间索引 + 扫描线求交 + Core 层 PLINQ |
| **孔洞/填充误判** | 该填的留洞或该留洞被填 | 路线 B 闭合半径可调 + 内切空圆稳健计算 + 测试覆盖 |
| 轮廓朝向错误 | bulge/孔洞方向反 | 有符号面积定向 + 外轮廓 CCW/孔 CW 归一 |
| 块引用膨胀 | 内存溢出 | 递归深度限制 + 增量展开 |
| UCS/3D 标高 | 结果错位 | 统一投影到工作平面 + UcsService 变换 |

---

## 十一、非功能需求（NFR）

| 项目 | 要求 |
|------|------|
| 性能 | **上万对象级秒级**（目标 ~12500 对象 ≤ 5 秒，对齐真实产品）；提供进度反馈 |
| 可中断 | 长任务支持 Esc 取消，状态可回滚 |
| 确定性 | 同输入产生同输出（含并行下结果稳定）|
| 内存 | 大图处理峰值可控，块展开有上限 |
| 健壮性 | 对脏数据（重复/零长/自交）不崩溃，降级处理 |
| 本地化 | 全局命令名保持英文；提示文本可中英 |
| 可诊断 | 结构化日志，异常含上下文 |

---

## 十二、假设与默认值

| 项目 | 默认值 | 说明 |
|------|--------|------|
| 间隙容差 GapTolerance | 0.001 绘图单位 | 端点桥接阈值，用户可在 INI/GUI 中修改 |
| 闭合半径 ClosingRadius | ≈ 2–5 × GapTolerance（自适应）| 路线 B 区分孔洞/填充；越大轮廓越平滑、孔越少 |
| 最大矢高 | 由绘图单位自适应 | Spline/Ellipse 线性化精度，取代固定节点数 |
| 最大角度步进 | 15° | 线性化角度上限，与矢高取严者 |
| 圆弧表示 | bulge（不线性化）| Arc/Circle 精确保留 |
| 输出图层 | 自动创建 "TB_Boundary" | 可按规则自定义 |
| 输出颜色 | ByLayer | 可改固定颜色 |
| 块展开深度 | 5 层 | 防止递归膨胀 |
| 线程模型 | 主线程 STA + Core 并行 | 禁止跨线程访问 DBObject |

---

## 十三、加载与部署

### 13.1 Autoloader Bundle（推荐）
将插件打包为 `TotalBoundary.bundle/` 文件夹，含 `PackageContents.xml`，部署到：
- 全局：`%ProgramFiles%\Autodesk\ApplicationPlugins\`
- 用户：`%APPDATA%\Autodesk\ApplicationPlugins\`

`PackageContents.xml` 通过 `<RuntimeRequirements>` 为不同 AutoCAD 版本指定不同 DLL：

```xml
<ApplicationPackage SchemaVersion="1.0" Name="TotalBoundary" AppVersion="2.1">
  <Components Description="AutoCAD 2013-2024">
    <RuntimeRequirements OS="Win64" Platform="AutoCAD"
                         SeriesMin="R19.0" SeriesMax="R24.3"/>
    <ComponentEntry AppName="TB" ModuleName="./net40/TB.AutoCAD.dll"
                    LoadOnAutoCADStartup="False">
      <Commands GroupName="TB">
        <Command Global="TOTALBOUNDARY" Local="TOTALBOUNDARY"/>
        <Command Global="TB" Local="TB"/>
        <Command Global="TBALL" Local="TBALL"/>
        <Command Global="TBPICK" Local="TBPICK"/>
      </Commands>
    </ComponentEntry>
  </Components>
  <Components Description="AutoCAD 2025+">
    <RuntimeRequirements OS="Win64" Platform="AutoCAD" SeriesMin="R25.0"/>
    <ComponentEntry AppName="TB" ModuleName="./net8/TB.AutoCAD.dll"
                    LoadOnAutoCADStartup="False"/>
  </Components>
</ApplicationPackage>
```

> 命令注册为**按需加载**（`LoadOnAutoCADStartup="False"`），首次调用命令时才载入 DLL。

### 13.2 WiX MSI
MSI 的职责仅是把 `.bundle` 部署到 ApplicationPlugins 并提供卸载，不自行处理多版本加载逻辑。

---

## 十四、依赖与授权

| 依赖 | 用途 | 备注 |
|------|------|------|
| ObjectARX SDK 2013 | net40 构建引用 | 取自 Autodesk 开发者门户 |
| ObjectARX SDK 2025 | net8 构建引用 | .NET 8 版本 |
| WiX Toolset | 安装包 | v4+ |
| xUnit | 单元测试 | TB.Core |
| RealDWG（可选）| 无宿主集成测试 | 需单独授权，否则用 accoreconsole |

---

## 十五、后续步骤

1. ✅ 方案确认（v2.1，最低版本 AutoCAD 2013，核心=外轮廓提取/路线 B）
2. ⏳ 搭建多目标 VS 解决方案 + Directory.Build.props
3. ⏳ 实现 TB.Core 算法（相交打断 → 面遍历 → 面分类(路线 B) → 轮廓提取）
4. ⏳ 实现 TB.AutoCAD 薄适配层与三命令（TOTALBOUNDARY/TBALL/TBPICK）
5. ⏳ 实现 Pro 功能
6. ⏳ 实现 ProPlus 功能
7. ⏳ 集成测试（accoreconsole 多版本冒烟）与优化
8. ⏳ 打包 bundle + WiX 发布

---

> 文档修订说明：v2.1 在 v2.0 基础上**纠正了核心目标** —— 经核实 TotalBoundary 是"外轮廓/周界提取器"，故将算法最终阶段从"输出每个内部面"改为"**面分类(路线 B 形态学闭合) + 轮廓提取**"，并对齐命令体系（TOTALBOUNDARY 主命令）与性能基准（上万对象秒级）。SuperBoundary 式全区域提取不在复刻范围。v2.0 的目标框架（多目标 net40+net8）、最低版本 2013、线程模型、相交打断、bulge、加载机制等修订继续有效。
