# PSP-UFU 软件架构分析与功能模块说明

> 分析对象：开源项目 **PSP-UFU**（Power Systems Platform of Federal University of Uberlândia）
> 仓库路径：`D:\github-repo\PSP`
> 分析目的：为在其基础上二次开发"电力系统仿真教学工具"提供架构基线
> 许可协议：GPL v2（见文末「许可与合规」一节，二次开发前必读）

---

## 1. 项目概述

PSP-UFU 是巴西乌贝兰迪亚联邦大学（UFU）开源的**跨平台、多语言、免费电力系统仿真软件**，定位为面向**科研、教学与工业应用**的图形化（CAD 风格）电力系统分析平台。作者 Thales Lima Oliveira，首发 2017 年，有正式学术论文支撑（*Int. Trans. Electr. Energ. Syst.* 2019）。

它允许用户通过可视化元件"搭积木"式地构建任意输电网络与控制系统，并提供潮流、短路、谐波、暂态/动态稳定四大类分析，结果以屏幕联动文本、表格、曲线、热力图等形式呈现。

**代码规模（实测）**

| 指标 | 数值 |
|------|------|
| 语言 | C / C++（GUI 基于 wxWidgets） |
| C/C++ 源文件行数 | ~74,000 行 |
| `Project/` 下文件数 | 349 |
| 电力元件类型 | 10（含母线 Bus） |
| 控制元件类型 | 14 |
| 仿真求解器类 | 5（1 基类 + 4 求解器） |
| 表单（对话框） | ~50（每个元件 + 设置/报告） |

---

## 2. 技术栈

| 类别 | 选型 | 用途 |
|------|------|------|
| GUI 框架 | **wxWidgets 3.1.6 / 3.2.2.1** | 跨平台桌面 UI（Ribbon 菜单、Notebook、对话框） |
| 界面设计 | wxFormBuilder（`.wxcp` 工程） | 表单代码生成（`*Base.cpp/.h`） |
| 持久化 | **rapidxml** | 工程文件 `.psp` 的 XML 读写 |
| 数学表达式 | **fparser**（自带） | 控制元件中的用户自定义公式解析 |
| 傅里叶变换 | **FFTW 3.3.5** | 谐波/频谱分析 |
| 绘图 | **wxChartDir** 或 **wxMathPlot**（二选一，可配置） | 结果曲线、频率响应 |
| 图形渲染 | `wxGraphicsContext`（GDI+ 路径）；GLFW+GLEW（OpenGL 部分接入但大多注释） | 画布绘制 |
| 文档 | Doxygen + Docusaurus | API 文档与用户手册 |
| 构建 | Visual Studio `.sln` / `.vcxproj`（也支持 CMake 思路的跨平台编译） | — |

> 注意：项目**默认仅 Windows 构建配置完备**（`.vcxproj` 含绝对路径 `wxWidgets-3.1.6` 等）。跨平台（Linux/macOS）需自行配置 wxWidgets 环境。

---

## 3. 整体架构（分层）

PSP 采用**单体桌面应用 + 面向对象分层**结构。自上而下分为五层，依赖方向基本单向（上层依赖下层，仿真层反依赖元素模型）：

```
┌──────────────────────────────────────────────────────────────┐
│  L5  应用层 (Application)                                      │
│   main.cpp (MainApp : wxApp)  →  MainFrame (Ribbon/Notebook)   │
└───────────────────────────────┬──────────────────────────────┘
                                 │ 调用/编排
┌───────────────────────────────▼──────────────────────────────┐
│  L4  画布/编辑器层 (Editors)                                    │
│   Workspace（电力网络画布）   ControlEditor（控制框图编辑器）    │
│   Camera / GraphAutoLayout / HMPlane（缩放、自动布局、热力图）   │
└───────────────────────────────┬──────────────────────────────┘
                                 │ 持有/管理
┌───────────────────────────────▼──────────────────────────────┐
│  L3  元素模型层 (Element Model)  —— 数据与图形一体的领域对象     │
│   Element (抽象基类)                                            │
│     ├─ PowerElement (抽象) → Bus/Line/Transformer/Generator…   │
│     └─ ControlElement (抽象) → Gain/Sum/TransferFunction…      │
│   Text / GCText（注释与联动文本）                               │
└───────────────────────────────┬──────────────────────────────┘
                                 │ 传入 std::vector<Element*>
┌───────────────────────────────▼──────────────────────────────┐
│  L2  仿真计算引擎 (Simulation Engine)                           │
│   ElectricCalculation (基类: YBus/求逆/ABC-DQ0/分类)            │
│     ├─ PowerFlow      (潮流: GS / NR / Hybrid)                 │
│     ├─ Fault          (短路: 对称/不对称/母线短路容量)          │
│     ├─ Electromechanical (暂稳/动稳: 同步机模型1-5, 梯形积分)   │
│     └─ PowerQuality   (谐波: 谐波YBus, THD, 频扫)              │
└───────────────────────────────┬──────────────────────────────┘
                                 │ 读取/回写
┌───────────────────────────────▼──────────────────────────────┐
│  L1  基础设施层 (Utilities & Persistence)                      │
│   PropertiesData(SimulationData) · XMLParser · FileHanding     │
│   ElementPlotData · DegreesAndRadians · 多语言(i18n) · 主题     │
└──────────────────────────────────────────────────────────────┘

外部依赖（extLibs / vendors）：rapidxml, fparser, fftw, wxChartDir,
wxMathPlot, glfw, artProvider, chatdir(FFT二进制)
```

**关键耦合点**：仿真层（`ElectricCalculation`）通过 `GetElementsFromList(std::vector<Element*>)` 直接持有**具体元件对象指针**，并调用 `Bus*`、`Line*`、`SyncGenerator*` 等强类型列表。这意味着**仿真引擎与元件模型强耦合**，无法零依赖地单独抽取为"纯数值库"。这是二次开发时需要重点改造的结构点（详见第 9 节）。

---

## 4. 目录结构与各层落点

```
PSP/
├─ PSP.sln / Project/PSP-UFU.vcxproj    # 构建
├─ Project/
│  ├─ main.cpp                          # L5 入口 (MainApp)
│  ├─ MainFrame.{h,cpp}                 # L5 主窗口/Ribbon 编排
│  ├─ elements/                         # L3 元素模型
│  │  ├─ Element.{h,cpp}                #   抽象基类（图形+CAD属性）
│  │  ├─ GraphicalElement.*             #   图形基类
│  │  ├─ Text.* / GCText.*              #   文本/联动文本
│  │  ├─ powerElement/                  #   10 种电力元件
│  │  │  ├─ PowerElement.* (抽象)
│  │  │  ├─ Bus, Line, Transformer, SyncGenerator, SyncMotor,
│  │  │  │  IndMotor, Load, Capacitor, Inductor, HarmCurrent,
│  │  │  │  EMTElement, Branch, Shunt, Machines
│  │  └─ controlElement/                #   14 种控制元件
│  │     ├─ ControlElement.* (抽象) + Node
│  │     ├─ Gain, Sum, Divider, Multiplier, MathOperation,
│  │     │  MathExpression, Limiter, RateLimiter, Saturation,
│  │     │  TransferFunction, Constant, IOControl, Exponential,
│  │     │  ConnectionLine, ControlElementContainer, ControlElementSolver
│  ├─ simulation/                       # L2 仿真引擎
│  │  ├─ ElectricCalculation.* (基类)
│  │  ├─ PowerFlow.*  Fault.*  Electromechanical.*  PowerQuality.*
│  ├─ forms/                            # L5 所有对话框（wxFormBuilder 生成）
│  │  ├─ MainFrameBase, BusForm, LineForm, SyncMachineForm,
│  │  │  DataReport, StabilityEventList, SimulationsSettingsForm …
│  ├─ utils/                            # L1 基础设施
│  │  ├─ PropertiesData.* (SimulationData)
│  │  ├─ XMLParser.*  FileHanding.*  Camera.*  GraphAutoLayout.*
│  │  ├─ HMPlane.* (热力图)  ElementPlotData.*  editors/Workspace.*
│  ├─ data/                             # 图标、翻译(po/mo)、示例工程(.psp)
│  │  ├─ samples/ (IEEE 9/14/30/57/118 等标准算例)
│  │  └─ lang/ (en / pt_BR / zh_CN 翻译)
│  └─ extLibs/                         # 第三方库源码/头文件
├─ docs/   docusaurus/                  # 文档与官网
└─ vendors/                             # chatdir / fftw 二进制
```

---

## 5. 核心模块详解

### 5.1 应用与 UI 层（L5）
- **`main.cpp` / `MainApp`**：程序入口。负责初始化 wxImage、加载 `config.ini`（语言/主题/绘图库/ATP 路径）、加载多语言 catalog（`zh_CN` 已内置）、解析命令行（可直接打开 `.psp`，`--test` 跑自测）。
- **`MainFrame`**：核心编排者，继承 wxFormBuilder 生成的 `MainFrameBase`。管理 **Ribbon 菜单**（添加元件、潮流、短路、稳定、谐波、导入、设置）与 **AuiNotebook**（多工程页，每页一个 `Workspace`）。所有仿真按钮的 `OnXxxClick` 在此触发，实例化对应求解器并传入当前画布元件列表，最后调度结果展示。

### 5.2 画布 / 编辑器层（L4）
- **`Workspace`**：电力网络的可视化与交互核心。持有 `std::vector<Element*>` 元件集合，负责鼠标交互（添加/拖拽/旋转/连线/拾取框）、`Camera`（平移缩放）、`DrawDC`（用 `wxGraphicsContext` 绘制所有元件与潮流箭头）、自动布局 `GraphAutoLayout`、热力图 `HMPlane`、以及调用仿真类。
- **`ControlEditor`**：独立的控制框图编辑器。用于搭建励磁器/AVR/PSS/调速器等控制系统的**方块图**，由 `ControlElementSolver` 在暂稳仿真中按时间步长求解。

### 5.3 元素模型层（L3）—— 领域对象
设计采用**"图形即数据"**的单一对象模型：每个元件既是图形实体（坐标、角度、连线点、拾取框），又携带电气参数与求解所需状态。

- **`Element`（抽象基类）**：定义位置/尺寸/旋转/选择/父子关系/XML 存取的通用接口；纯虚 `Contains()`、`Intersects()`、`DrawDC()`、`ShowForm()`、`SaveElement()`/`OpenElement()`。
- **`PowerElement`（抽象）**：扩展电气属性——标称电压与单位、投切数据（`SwitchingData`）、潮流箭头方向、动态事件标记、绘图数据接口 `GetPlotData()`。
- **具体电力元件（10 类）**：`Bus`（母线，网络的连接枢纽）、`Line`（线路）、`Transformer`（变压器）、`SyncGenerator`/`SyncMotor`（同步机）、`IndMotor`（感应电机）、`Load`（负荷）、`Capacitor`/`Inductor`（无功补偿）、`HarmCurrent`（谐波电流源）、`EMTElement`（电磁暂态占位）。
- **`ControlElement`（抽象）**：控制方块基类，含输入/输出 `Node`、纯虚 `Solve(double* input, double timeStep)` 与 `Initialize()`、`GetOutput()`，实现"按步求解"的模块契约。
- **具体控制元件（14 类）**：`Gain`、`Sum`、`Divider`、`Multiplier`、`MathOperation`、`MathExpression`（fparser 公式）、`Limiter`、`RateLimiter`、`Saturation`、`TransferFunction`（传递函数，状态空间）、`Constant`、`IOControl`（与电力侧量测/控制量对接）、`Exponential`、`ConnectionLine`、`ControlElementContainer`。

### 5.4 仿真计算引擎（L2）
所有求解器继承自 **`ElectricCalculation`**，复用其电网解析与线性代数能力：

- **`ElectricCalculation`（基类）**
  - `GetElementsFromList()`：按类型将 `Element*` 拆分为 Bus/Line/... 强类型列表；
  - `GetYBus()`：构建节点导纳矩阵，支持**正序/负序/零序**三序，可选计入同步机/负荷等效阻抗；
  - `InvertMatrix()` / `GaussianElimination()` / `GetLUDecomposition()` / `LUEvaluate()`：复矩阵求逆与求解；
  - `ABCtoDQ0()` / `DQ0toABC()`：三相↔dq0 坐标变换；
  - `GetMachineModel()`：自动选择同步机模型（1–5 阶）。

- **`PowerFlow`（潮流）**
  - 三种算法：`RunGaussSeidel()`、`RunNewtonRaphson()`、`RunGaussNewton()`（混合 NR-GS，带惯性因子）；
  - 支持无功越限处理 `CheckReactiveLimits()`、含感应电机负荷 `CalculateMotorsReactivePower()`；
  - 计算结果通过 `UpdateElementsPowerFlow()` 回写元件（用于屏幕联动文本与潮流箭头）。

- **`Fault`（短路）**
  - `RunFaultCalculation()`：对称/不对称短路（三相、两相、两相接地、单相接地），基于序网与故障复合；
  - `RunSCPowerCalcutation()`：计算各母线短路容量（SCC）。

- **`Electromechanical`（暂态/动态稳定）**
  - 时间域逐步积分（梯形法 `IntegrationConstant`，步长默认 1e-2 s）；
  - 同步机多模型（模型 1–5 自动选择，含饱和 `CalculateSyncMachineSaturation`）；
  - 感应电机暂态 `CalculateIndMachinesTransientValues()`；
  - **事件/投切系统**：`SetEventTimeList()` / `HasEvent()` / `SetEvent()` 实现故障、开关动作的时间序列；
  - 与控制编辑器联动：在每个步长调用 `ControlElementSolver` 求解励磁/调速/PSS 等控制环（控制步长 = 主步长 / `controlTimeStepRatio` 默认 10）；
  - 可选 COI（中心 of inertia）参考、母线频率估计（相角微分 / washout 滤波器）。

- **`PowerQuality`（谐波）**
  - `CalculateHarmonicYbusList()` / `CalculateHarmonicYbus()`：各次谐波频率下的导纳矩阵；
  - 谐波电压与 **THD** 计算、`FrequencyResponseForm` 频扫（依赖 FFTW）。

### 5.5 数据与持久化（L1）
- **`PropertiesData`**：集中管理 `SimulationData`（基值、潮流方法/容差/迭代、稳定步长/时长/容差、ZIP 负荷、谐波接法）、`GeneralData`（语言/主题/绘图库）、`FreqResponseData`。
- **`XMLParser` + `FileHanding`**：将整个工程（元件几何 + 电气参数 + 控制框图 + 仿真设置）序列化为 `.psp`（rapidxml）。每个 `Element` 实现 `SaveElement/OpenElement`，实现"所见即所存"。

---

## 6. 四大仿真功能模块（用户视角）

| 模块 | 入口(Ribbon) | 求解器 | 教学价值 |
|------|-------------|--------|---------|
| **潮流计算** | Power Flow | `PowerFlow` | 理解 GS/NR 迭代收敛、PV/PQ/Slack 节点、无功越限 |
| **短路计算** | Fault / SCC | `Fault` | 对称分量法、序网、母线短路容量 |
| **谐波分析** | Harmonics / Freq. Response | `PowerQuality` | 谐波阻抗、THD、频响扫描 |
| **暂态/动态稳定** | Stability | `Electromechanical` | 同步机模型、摇摆曲线、PSS/AVR 作用、故障时序 |

外加 **控制编辑器（Control Editor）**：用方块图搭建并仿真任意控制系统，是"展示调节器原理"的绝佳教学载体。

---

## 7. 数据流 / 一次典型运行流程

```
[用户在 Workspace 搭网] ──拖入 Bus/Line/...，连线，双击设参(Forms)
        │
        ▼  (点击 Ribbon 按钮)
[MainFrame::OnPowerFlowClick]
        │  取 m_workspaceList[i] 的元件 vector<Element*>
        ▼
[PowerFlow pf(elementList)]
   ├─ GetElementsFromList()      // 拆分强类型列表
   ├─ GetYBus()                  // 建导纳矩阵(用元件参数)
   ├─ RunNewtonRaphson()         // 迭代求解
   └─ UpdateElementsPowerFlow()  // 回写电压/功率
        │
        ▼
[Workspace 刷新] 联动文本(电压/相角)、潮流箭头、DataReport 表格、ChartView 曲线
        │
        ▼
[FileHanding / XMLParser] 可保存为 .psp 供下次/学生作业复用
```

---

## 8. 文档与示例资源

- **用户手册 / 官网**：`docusaurus/`（含 `installation`、`powerFlow`、`stability`、`controlEditor`、`harmonics` 等 40+ 篇 Markdown，已含中文页面骨架）。
- **API 文档**：`docs/doxygen/`（Doxygen 生成，含类继承图）。
- **标准算例**：`Project/data/samples/`（IEEE 9/14/30/57/118 母线、OMIB with/without PSS 等），可直接作为教学素材。
- **多语言**：已内置 `en` / `pt_BR` / `zh_CN` 翻译文件（`.po/.mo`），中文界面基础已具备。

---

## 9. 面向"二次开发教学工具"的评估（重点）

### 9.1 可直接复用的资产（高价值）
1. **仿真内核算法**：潮流（GS/NR/混合）、短路（序网）、谐波、暂稳（同步机模型 1–5 + 梯形积分）均为**成熟、有论文背书、可直接运行**的实现，是教学工具最核心的资产。
2. **标准算例库**：IEEE 系列与 OMIB 算例可直接当教案/作业。
3. **控制编辑器**：方块图搭建+求解机制，非常适合演示"励磁系统/调速系统/PSS 如何镇定系统"。
4. **多语言与主题框架**：`zh_CN` 已就绪，主题（亮/暗）可配置。
5. **完整用户文档**：Docusaurus 手册可改造成教学指南。

### 9.2 架构耦合与改造难点（需提前规划）
1. **仿真层 ↔ 元件模型强耦合**：求解器直接依赖具体 `Element` 派生类指针。若想做一个**轻量/Web 版教学前端**，无法直接调用其引擎，需先做一次"解耦重构"——把数值计算抽成接收**纯数据结构（如导纳矩阵、参数表）**的库，与 GUI 脱钩。
2. **桌面单体架构**：基于 wxWidgets 的原生桌面应用。学生分发、跨平台体验、在线教学（浏览器访问）均有门槛。**教学工具常见更优形态是 Web（前端交互 + 后端仿真服务）**，这要求把 L2 引擎用 C API / WASM / 后端服务封装。
3. **UI 渲染以 GDI+ 为主**：OpenGL 路径大多注释掉，3D/炫酷可视化有限；若要做"沉浸式教学可视化"需自行补充。
4. **缺少异步/线程化仿真**：长时暂稳仿真在 GUI 主线程运行（`RunStabilityCalculation` 为同步调用），大系统可能卡 UI。教学场景下需加后台线程/进度反馈。
5. **测试覆盖薄弱**：仅见 `RunPSPTest()` 自测入口，无现代单元测试框架（gtest 等）。重构前建议先补引擎层回归测试，避免改坏算法。
6. **构建配置偏 Windows**：`.vcxproj` 含本地绝对路径，跨平台/CI 需重新配置 wxWidgets。

### 9.3 许可与合规（⚠️ 必读）
- PSP-UFU 采用 **GPL v2**（强 copyleft）。**基于其代码修改/衍生的作品，若对外分发，必须以 GPL v2 开源**——无法闭源或转专有授权。
- 这意味着：若你们的"教学工具"计划**闭源商业化或仅内部闭源部署**，直接 fork/修改源码会触发 GPL 义务。
- **合规路径建议**：
  - (a) 整个教学工具以 GPL v2 开源（最省心，与上游一致）；
  - (b) **仅借鉴算法思路、自行独立重新实现**数值内核（不复制其源码），则不受 GPL 传染，可自由定许可——但工作量与"避免代码相似"的法律边界需评估；
  - (c) 与作者联系获取**单独的商业/专有授权**（GPL 项目作者可双重授权）。
- 无论如何，**保留原始版权声明与 LICENSE** 是底线。

### 9.4 建议演进路线（教学工具）
```
阶段1  熟悉与验证：用现有 PSP 跑通标准算例，建立"答案基准"，补充引擎回归测试
阶段2  解耦内核：将 L2 仿真引擎重构为"输入纯数据、输出结果"的独立库（C/C++ API）
阶段3  教学化前端：
       方案A（低成本）：在 wxWidgets 上增教学功能（步骤提示、理论弹窗、作业导出）
       方案B（推荐）：内核封装为后端服务/ WASM，配 Web 前端（Laravel/Livewire 等）
阶段4  教学增强：动画化摇摆曲线、控制环实时演示、算例库、自动批改、多语言完善
```

---

## 10. 总结

PSP-UFU 是一个**算法完整、工程扎实、文档齐全**的开源电力系统仿真器，其四大分析模块 + 控制编辑器几乎覆盖了电工类专业的核心仿真教学需求，标准算例与中文界面基础也降低了起步成本。

对"在其基础上做教学工具"而言，**真正的价值在 L2 仿真内核与算例库**，而**最大的改造点在"桌面单体 + 引擎/UI 强耦合 + GPL 许可"三件事上**。建议把重心放在"内核解耦 + 现代化教学前端 + 合规策略"上，而不是在 wxWidgets 桌面壳里缝缝补补。

---

*文档由代码静态分析生成，结合 `main.cpp` / `MainFrame` / `Element` / `PowerElement` / `ControlElement` / `ElectricCalculation`(+4 求解器) / `Workspace` / `PropertiesData` / `.vcxproj` 等核心文件。如需对某模块（如暂稳积分器、控制求解器、序列化格式）做逐函数级别的深读，可继续指定。*
