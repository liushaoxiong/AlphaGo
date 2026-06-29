# TotalBoundary Clone — AutoCAD .NET 插件开发方案

> 版本: **v2.0** | 日期: 2026-06-29 | 状态: 待确认
>
> **v2.0 修订要点**：
> - 🔴 修正目标框架策略——单一 .NET 4.8 无法覆盖目标版本，改为**多目标构建（net40 + net8.0-windows）**
> - 🔴 最低支持版本由 2008 上调为 **AutoCAD 2013**（.NET 4.x 起点，工程成本与装机量的最佳平衡）
> - 🔴 修正线程模型矛盾：主线程 STA + 纯几何层并行
> - 🟠 算法补强：新增**线段相交打断（Bentley–Ottmann）**、修正环路算法为**平面图面遍历**、圆弧改用 **bulge** 而非线性化
> - 🟠 加载机制改为 **Autoloader Bundle**，自动适配多版本
> - 🟡 补充术语表、数据模型、NFR、验收标准、构建矩阵、可测试性策略

---

## 一、项目概述

复刻 AutoCAD 市场知名插件 **TotalBoundary / SuperBoundary** 的核心功能，使用 AutoCAD .NET API 自主开发一个边界提取插件。采用**分级产品策略**，分三阶段交付，每阶段可独立编译和部署。

### 核心能力
- 从封闭或近似封闭的几何集合中提取闭合多段线（Boundary Polyline）
- 支持 Line、Arc、Circle、Ellipse、Spline、Polyline、BlockReference 等实体类型
- 强大的**间隙容差（Gap Tolerance）**处理能力，远超 AutoCAD 原生 `BOUNDARY` / `BPOLY` 命令
- 正确处理**线段中段相交**（X/T 交叉），而不仅是端点连接
- 可选生成填充图案（Hatch）、计算面积/周长（含孔洞扣除）、导出 DXF

### 与原生命令的本质差异
| 能力 | 原生 BOUNDARY/BPOLY | 本插件 |
|------|--------------------|--------|
| 间隙桥接 | 几乎不容忍 | 可配置容差，主动桥接 |
| 中段交叉处理 | 依赖屏幕显示精度 | 显式求交打断 |
| 容差/采样可控 | 黑盒 | 全参数化 |
| 输出形式 | Polyline/Region | Polyline(bulge)/Region/Hatch/DXF |

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

> 最小可行产品 (MVP)，解决核心痛点：**提取闭合边界多段线**

| 模块 | 功能描述 |
|------|----------|
| 实体分析器 | 识别并分类选择集中的所有实体类型 (Line, Arc, Circle, Ellipse, Spline, Polyline, BlockReference) |
| 预处理/清理 | 去重、删除零长线段、合并重叠共线段（类 OVERKILL）|
| 间隙检测器 | 基于空间索引计算端点近邻距离，判断是否在容差范围内 |
| 相交求解器 | Bentley–Ottmann 扫描线求线段交点并在交点处打断 |
| 平面图构建器 | 顶点/边邻接结构，记录边方向 |
| 面遍历器 | 极角排序 + next-edge 面遍历，找出所有闭合环路 |
| 环层级分析 | 岛屿/孔洞包含树（奇偶规则）|
| 多段线构建器 | 环路 → ClosedPolyline；**圆弧用 bulge**，仅样条/椭圆自适应线性化 |
| 命令集 | `TBBoundary`(窗口选择)、`TBQuickPick`(点选区域)、`TBAll`(全图提取) |
| 属性设置 | 颜色、线宽、图层分配 |
| 进度指示 | 大型图纸处理时显示进度条，支持 Esc 中断 |

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
│   │   ├── Vertex.cs / Edge.cs     # 平面图基本元素
│   │   ├── Loop.cs                 # 闭合环 (顶点序列 + bulge)
│   │   └── LoopHierarchy.cs        # 岛屿/孔洞包含树
│   ├── Geometry/
│   │   ├── SpatialIndex.cs         # 网格/R-tree 空间索引
│   │   ├── GapDetector.cs          # 近邻端点 + 容差比较
│   │   ├── SegmentIntersector.cs   # Bentley–Ottmann 求交并打断
│   │   ├── RobustPredicates.cs     # 浮点健壮判定 (共线/朝向)
│   │   ├── PlanarGraphBuilder.cs   # 邻接图 + 边方向
│   │   ├── FaceTracer.cs           # 极角排序 + next-edge 面遍历
│   │   └── Linearizer.cs           # Spline/Ellipse 自适应线性化 (矢高控制)
│   ├── Output/
│   │   └── PolylineSpec.cs         # 与 CAD 无关的多段线规格 (含 bulge)
│   └── TB.Core.csproj              # 多目标 net40;net8.0-windows
├── TB.AutoCAD/                     # CAD 集成层 (薄适配器)
│   ├── Commands/
│   │   ├── TBoundary.cs            # TBBoundary — 窗口选择
│   │   ├── TBQuickPick.cs          # TBQuickPick — 点选 (射线/区域生长)
│   │   └── TBAll.cs                # TBAll — 全图提取
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
用户选择实体/区域
    │
    ▼
[主线程] EntityExtractor ── 解析 DBObject → 纯 POCO Segment (含 UCS→WCS)
    │
    ▼
[可并行] Preprocess ──────── 去重 / 删零长 / 合并共线
    │
    ▼
[可并行] SegmentIntersector ─ Bentley–Ottmann 求交，在交点处打断   ★新增关键步骤
    │
    ▼
[可并行] GapDetector ──────── 空间索引近邻 + 容差桥接
    │
    ▼
[可并行] PlanarGraphBuilder ─ 构建带方向邻接图
    │
    ▼
[可并行] FaceTracer ───────── 极角排序 + next-edge 面遍历 (非 Hierholzer) ★修正
    │
    ▼
[可并行] LoopHierarchy ────── 岛屿/孔洞包含树
    │
    ▼
[主线程] PolylineWriter ───── PolylineSpec → ClosedPolyline (圆弧用 bulge)
    │
    ▼
   输出结果
```

> 关键：CAD 对象的读写都在主线程 (STA)；中间无 CAD 依赖的纯几何步骤可用 PLINQ 并行。

### 6.2 关键算法细节

#### 预处理 (Preprocess)
- 删除零长线段、完全重复实体；
- 合并重叠共线段，避免平面图退化；
- 等价于轻量级 OVERKILL，是健壮性的第一道防线。

#### 相交求解 (SegmentIntersector) ★新增
- **Bentley–Ottmann 扫描线**求所有线段交点，复杂度 O((n+k) log n)，k 为交点数；
- 在交点处打断线段，生成平面细分的边；
- 圆弧/样条参与求交时先按容差处理为可求交表示，交点回投到原曲线参数。

#### 间隙检测 (GapDetector)
- 用**空间索引**（网格或 R-tree）做端点近邻查询，避免 O(n²)；
- 距离 ≤ 容差 → 桥接为同一顶点；否则标记为开放端；
- 容差默认 0.001 绘图单位，用户可调。

#### 面遍历 (FaceTracer) ★修正算法
- 在每个顶点把出边按**极角排序**；
- 进入一条边到达对端后，**总是选择最靠左（或最靠右）的下一条出边**（wavefront / next-clockwise-edge）；
- 遍历平面图所有内部面，即所有闭合边界；
- 注：原 Hierholzer 用于欧拉回路，不适用于面提取，已替换。

#### 点选提取 (TBQuickPick)
- 不做全图面遍历，而是从拾取点**射线投射 / 区域生长**，定位包围该点的最小环；
- 命中后局部构建平面图并提取该面，性能更高、交互更快。

#### 环层级 (LoopHierarchy)
- 用点-在-多边形 + 包含关系建**包含树**；
- 区分外环（逆时针）与内孔（顺时针），供面积扣除与 Hatch 岛屿使用。

#### 多段线构建 (PolylineWriter)
- **Arc / Circle → bulge 段，精确无损**（不线性化）；
- Spline / Ellipse → 按**矢高/弦高 (sagitta) 自适应**细分为折线，用户可控最大弦高与最大角度；
- 可选输出 `Region` 对象。

---

## 七、数据模型与术语表

| 术语 | 定义 |
|------|------|
| `Segment` | 几何线段的统一抽象，三态：直线段 / 圆弧段（含 bulge）/ 样条段；不含任何 CAD 类型 |
| `Vertex` | 平面图顶点，承载坐标与容差桥接后的合并身份 |
| `Edge` | 连接两顶点的有向边，记录方向与来源 Segment |
| `Loop` | 闭合环，顶点序列 + 每段 bulge + 朝向（CW/CCW）|
| `LoopHierarchy` | 环的包含树，表达外环-孔洞-岛屿层级 |
| `PolylineSpec` | 与 CAD 无关的多段线规格，供 PolylineWriter 落地为 LWPolyline |
| 容差 (Tolerance) | 端点视为连接的最大间隙距离 |
| 矢高 (Sagitta) | 弦与弧之间的最大垂距，控制线性化精度 |

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
| 6 | PlanarGraphBuilder + FaceTracer | 单环/交叉样例提取正确 |
| 7 | LoopHierarchy | 外框+内孔提取出 2 环且层级正确 |
| 8 | PolylineWriter（bulge）| 含圆弧样例输出无失真 |
| 9 | 三命令 (TBBoundary/QuickPick/All) | 三命令可用，QuickPick 点选准确 |
| 10 | 属性设置 + 进度/中断 | 大图可显示进度且可 Esc 取消 |
| 11 | 对比验证 | 与原生 BOUNDARY 在干净图上结果一致 |

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
- [ ] Preprocess：重复 / 零长 / 共线重叠
- [ ] GapDetector：零间隙 / 小间隙 / 大间隙 / 无重叠
- [ ] SegmentIntersector：X 交叉 / T 搭接 / 多重共点 / 端点重合
- [ ] FaceTracer：单环 / 嵌套环 / 共享边 / 孤岛 / 开放链（不应成环）
- [ ] LoopHierarchy：外框+孔 / 多层岛屿 / 并列环
- [ ] PolylineWriter：纯直线 / 含圆弧(bulge) / 含样条(线性化) / 混合
- [ ] RobustPredicates：近共线 / 近重合的浮点边界

### 集成测试（accoreconsole.exe 脚本化，无界面）
- [ ] 简单矩形 (4 Line) → 1 ClosedPolyline
- [ ] 圆 (Circle) → 1 ClosedPolyline（bulge 表示）
- [ ] 带 0.5mm 间隙封闭图形 → 容差内自动闭合
- [ ] 十字交叉 → 打断后正确成面
- [ ] 嵌套环（外框+内孔）→ 2 Polyline + 层级正确
- [ ] 复杂工程图 (1000+ 实体) → < 5 秒，内存可控

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
| 嵌套环误判 | 提取错误边界 | 包含树 + 内外环朝向校验 |
| 块引用膨胀 | 内存溢出 | 递归深度限制 + 增量展开 |
| UCS/3D 标高 | 结果错位 | 统一投影到工作平面 + UcsService 变换 |

---

## 十一、非功能需求（NFR）

| 项目 | 要求 |
|------|------|
| 性能 | 1000+ 实体 < 5 秒；提供进度反馈 |
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
| 间隙容差 | 0.001 绘图单位 | 用户可在 INI/GUI 中修改 |
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
<ApplicationPackage SchemaVersion="1.0" Name="TotalBoundary" AppVersion="2.0">
  <Components Description="AutoCAD 2013-2024">
    <RuntimeRequirements OS="Win64" Platform="AutoCAD"
                         SeriesMin="R19.0" SeriesMax="R24.3"/>
    <ComponentEntry AppName="TB" ModuleName="./net40/TB.AutoCAD.dll"
                    LoadOnAutoCADStartup="False">
      <Commands GroupName="TB">
        <Command Global="TBBOUNDARY" Local="TBBOUNDARY"/>
        <Command Global="TBQUICKPICK" Local="TBQUICKPICK"/>
        <Command Global="TBALL" Local="TBALL"/>
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

1. ✅ 方案确认（v2.0，最低版本 AutoCAD 2013）
2. ⏳ 搭建多目标 VS 解决方案 + Directory.Build.props
3. ⏳ 实现 TB.Core 算法（含相交打断与面遍历）
4. ⏳ 实现 TB.AutoCAD 薄适配层与三命令
5. ⏳ 实现 Pro 功能
6. ⏳ 实现 ProPlus 功能
7. ⏳ 集成测试（accoreconsole 多版本冒烟）与优化
8. ⏳ 打包 bundle + WiX 发布

---

> 文档修订说明：本 v2.0 在 v1.0 框架基础上修正了目标框架兼容性（多目标 net40+net8）、线程模型、核心算法（新增 Bentley–Ottmann 求交、改用平面图面遍历、圆弧 bulge），并补充了数据模型、NFR、验收标准、加载机制与依赖授权。最低支持版本定为 AutoCAD 2013。
