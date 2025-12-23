# MAGICAL 项目工作流程与分析

## 项目概述

MAGICAL（Machine Generated Analog IC Layout）是一个开源项目，用于生成模拟 IC 布局。它采用模块化架构，通过顶层 Python 流程集成各个子模块。项目强调模拟电路的对称性、匹配性和寄生效应控制。

### 主要特性
- **模块化设计**：子模块处理约束生成、放置、路由等。
- **模拟电路重点**：优先考虑对称性和寄生效应，而非数字 IC 的面积/时序。
- **依赖**：C++ 库（Boost、Limbo、LPSolve、Lemon）、Python 3.7+。
- **构建**：使用 Docker 简化设置。

### 机器学习集成
MAGICAL 在多个阶段融入机器学习，以提升自动化和性能：
- **WellGAN**：基于GAN的井结构生成，用于优化器件布局。
- **GeniusRoute**：使用VAE的路由路径优化，预测最佳布线方案。
- **ML驱动放置评分**：在放置前使用神经网络预测布局质量。
- **贝叶斯优化**：迭代调优布局性能，减少手动迭代。

### 工作流程步骤
1. **输入准备**：网表（.sp）、PDK 文件（.lef、.techfile）、约束（.sym、.symnet）、JSON 配置。
2. **约束生成**：自动生成对称约束。
3. **分层处理**：
   - 器件生成。
   - 对称放置。
   - 路由（全局和详细）。
4. **输出**：最终 GDS 布局和中间文件。

### 文本流程图
```
MAGICAL Analog IC Layout Flow
=============================

START
  ↓
[1] Input Files Preparation
  │
  ├── JSON Config (adc1.json)
  │   ├── Netlist (.sp)
  │   ├── PDK Files (.lef, .techfile, .simple)
  │   └── Constraints (.sym, .symnet)
  ↓
[2] Parse Inputs (Python Flow)
  │
  ├── Load JSON Config
  ├── Parse Netlist (HSPICE/Spectre)
  ├── Parse PDK (LEF, Techfile)
  └── Parse Constraints
  ↓
[3] Constraint Generation (ConstGen)
  │
  ├── Auto-Generate Symmetry (.sym, .symnet)
  │   └── Manual Override if Needed
  ↓
[4] Hierarchical Processing (Bottom-Up)
  │
  ├── Device Generation (device_generation)
  │   ├── Generate Primitive Devices (MOS, Cap, Res)
  │   └── Output Device GDS
  ↓
  ├── Placement (IdeaPlaceEx)
  │   ├── Setup Hierarchy
  │   ├── Run Placer with Constraints
  │   ├── Output Placement Layout (.place.gds)
  │   └── Generate Floorplan (.floorplan.gds)
  ↓
  ├── Routing (anaroute)
  │   ├── Parse Placement Results
  │   ├── Global Routing (.gr, .gr.spec)
  │   ├── Detailed Routing
  │   └── Output Routed Layout (.route.gds)
  ↓
[5] Output Generation
  │
  ├── Final GDS (.route.gds)
  ├── Intermediate Files
  │   ├── Pin Files (.pin, .iopin)
  │   ├── Boundary (.bound)
  │   ├── Connections (.con)
  │   └── Specs (.spec)
  ↓
END
```

#### 流程说明
- **步骤 1-2**：准备和解析文件。Python 脚本加载配置，构建内部数据库。
- **步骤 3**：生成对称约束（自动或手动）。
- **步骤 4**：分层处理：
  - 器件生成：为底层器件创建布局。
  - 放置：优化位置，确保对称。
  - 路由：连接网络，考虑模拟要求。
- **步骤 5**：输出最终布局和中间文件。
- **运行命令**：`cd examples/BENCH; source run.sh`（调用顶层 Python 脚本）。
- **时间**：取决于电路复杂度，几分钟到小时。
- **注意**：流程是端到端的，但某些组件（如 anaroute）可单独运行调试。

## 架构和子模块

MAGICAL 使用 Git 子模块进行独立开发。

### 核心子模块
- **anaroute（模拟路由器）**：
  - 作用：路由，包括全局/详细路由、对称感知路由。
  - 独立运行：是（二进制和 Python API）。
  - 关键文件夹：`gr/`（全局路由）、`dr/`（详细路由）、`drc/`（DRC 检查）。

- **ConstGen（约束生成器）**：
  - 作用：生成器件和网络的对称约束。
  - 独立运行：部分（脚本/工具可用）。
  - 关键文件夹：`sym_detect/`（对称检测）。

- **IdeaPlaceEx（放置探索器）**：
  - 作用：模拟放置，具有对称约束。
  - 独立运行：可能（CMake 构建）。
  - 关键文件夹：`api/`（接口）。

- **device_generation（器件生成器）**：
  - 作用：生成基本器件（MOS、Cap、Res）。
  - 独立运行：可能（Python 包）。
  - 关键文件夹：如 `Mosfet.py` 的 Python 文件。

- **flow（顶层流程）**：
  - 作用：通过 Python 脚本集成子模块。
  - 独立运行：否（仅编排）。

### 其他组件
- **examples**：示例电路（ADC、OTA）。
- **build.sh、Dockerfile**：构建脚本。
- **mockPDK**：模拟 PDK 用于测试。

### 克隆和更新
```
git clone https://github.com/magical-eda/MAGICAL.git
cd MAGICAL
git submodule init
git submodule update
```

## 使用的算法

基于项目结构和 EDA 文献，关键算法包括：

### 核心算法
1. **模拟退火 (SA)**：
   - **原理**：通过模拟物理退火过程（温度逐渐降低）接受次优解，避免局部最优。使用概率函数决定是否接受劣解。
   - **IC 设计应用**：在放置阶段优化器件位置。目标函数包括面积、线长、对称违反和寄生效应。适用于 NP-hard 问题，如模拟 IC 的对称约束布局。
   - **实现**：代码中可能涉及温度调度和随机扰动。依赖 LPSolve 用于约束求解。

2. **力导向放置**：
   - **原理**：将布局建模为物理系统，器件为节点，连接为弹簧。应用力平衡位置。
   - **IC 设计应用**：在放置中处理器件间关系，如匹配对吸引力和拥塞排斥。适用于模拟 IC 的对称轴约束。
   - **实现**：可能使用数值方法求解。

3. **图同构**：
   - **原理**：比较图结构，通过节点/边映射检测相同性。
   - **IC 设计应用**：在约束生成中识别网表对称器件对和网络。网表可建模为图。
   - **实现**：`sym_detect/` 文件夹可能实现图匹配。

4. **迷宫路由**：
   - **原理**：网格化布线区域，使用 BFS 或 A* 寻找路径。
   - **IC 设计应用**：全局路由阶段分配网络到通道。模拟 IC 考虑对称和寄生。
   - **实现**：`gr/` 文件夹支持。Limbo 库提供工具。

5. **撕裂重路由 (RAR)**：
   - **原理**：迭代移除冲突路径，然后重新路由。
   - **IC 设计应用**：详细路由处理 DRC 违规。模拟 IC 用于对称网络重新布线。
   - **实现**：`dr/` 和 `drc/` 文件夹集成 DRC 检查。

6. **Steiner 树**：
   - **原理**：为多终端网络构建最小树，添加 Steiner 点。
   - **IC 设计应用**：优化网络拓扑最小化寄生。
   - **实现**：可能在 `graph/` 中。

7. **整数线性规划 (ILP)**：
   - **原理**：优化线性目标，受整数变量和线性约束。
   - **IC 设计应用**：处理放置/路由硬约束，如对称轴。
   - **实现**：LPSolve 求解。

8. **遗传算法**：
   - **原理**：模拟进化，选择、交叉、变异。
   - **IC 设计应用**：优化器件尺寸。
   - **实现**：device_generation 中可能。

9. **gm/ID 方法**：
   - **原理**：基于跨导效率建模器件。
   - **IC 设计应用**：快速尺寸晶体管，确保性能。
   - **实现**：Python 文件如 `Mosfet.py`。

10. **生成对抗网络 (GAN)**：
    - **原理**：生成器和判别器对抗训练，生成逼真数据。
    - **IC 设计应用**：在WellGAN中生成井结构，优化器件布局。
    - **实现**：用于自动生成符合DRC的几何结构。

11. **变分自编码器 (VAE)**：
    - **原理**：通过编码器和解码器学习数据分布，生成新样本。
    - **IC 设计应用**：在GeniusRoute中优化路由路径，预测布线方案。
    - **实现**：用于路由优化，减少拥塞和寄生。

12. **贝叶斯优化**：
    - **原理**：使用概率模型选择下一评估点，最小化目标函数。
    - **IC 设计应用**：迭代调优布局性能，如线长和对称。
    - **实现**：在放置和路由中减少手动迭代次数。

### 项目结构文件夹
- **geo/**：几何处理（多边形、DRC）。
- **graph/**：图算法（路由、Steiner 树）。
- **sym_detect/**：对称检测。
- **gr/**：全局路由。
- **dr/**：详细路由。
- **acs/**：模拟约束满足。

## 全局路由 vs. 详细路由

### 全局路由 (GR)
- **过程**：将芯片区域划分为网格，使用迷宫路由将网络分配到区域。
- **约束**：容量、粗略 DRC、对称。
- **目标**：最小化拥塞/线长。
- **输出**：`.gr`（指南）、`.gr.spec`（规格）。
- **在 MAGICAL 中**：`gr/` 文件夹。

### 详细路由 (DR)
- **过程**：基于 GR 指南生成实际线/通孔，确保 DRC 合规。
- **约束**：精确 DRC、几何、寄生。
- **目标**：DRC 无违规布局，寄生最小化。
- **输出**：最终 GDS。
- **在 MAGICAL 中**：`dr/`、`drc/` 文件夹。

### 关键差异
- GR：粗略规划，快速；DR：精细实现，准确。
- 模拟强调：两者中的对称/寄生，但 DR 更严格。

## 依赖和库

### 关键库
- **Limbo**：EDA 库，用于路由/几何。
- **Boost**：C++ 扩展（图、I/O）。
- **LPSolve**：ILP 求解器。
- **Lemon**：图算法。
- **Flex/Zlib**：解析/压缩。
- **Python 3.7+**：脚本/集成。

使用 Docker 简化设置。

## 文件格式和生成

### 输入文件
- **网表** (.sp)：HSPICE 或 Spectre 网表，描述电路拓扑、器件参数、连接。
- **PDK 文件**：
  - .lef：定义工艺层、单元尺寸、引脚位置。
  - .techfile：层名称到编号映射。
  - .simple：简化的层编号列表。
  - 标准单元 GDS 库：预定义单元 GDS。
- **约束**：
  - .sym：器件对称约束（对称组、对称对、自对称器件）。
  - .symnet：网络对称约束（对称网络对、自对称网络）。
- **配置**：JSON 文件，指定路径、参数、PDK 路径。

### 输出文件
- **.gr**：全局路由指南。文件头包括 gridStep（网格步长）、Offset（偏移）、symAxis（对称轴）。然后每行一个引脚形状：NetName pinNameIdx Layer xLo yLo xHi yHi isPower iopinshapeIsPowerStripe。Layer 从 1 开始（M1），坐标为 DBU 单位。
- **.gr.spec**：网络规格。每行一个网络：NetName width cuts isPower rows cols。width 为线宽（DBU），cuts 为通孔切割数，isPower 为电源标志，rows/cols 为布局参数。
- **最终 GDS**：布线布局。

### 在 PnR.py 中的生成
在 `routeParsePin` 函数中：
- `.gr`：写入网格信息（gridStep, Offset, symAxis），然后记录每个网络的引脚形状。
- `.gr.spec`：写入 "NET_SPEC:\n"，然后为每个网络写入一行规格。

### 使用新网表和新 PDK 需要的文件
要运行 MAGICAL，需要准备：
- 网表 (.sp)：HSPICE 或 Spectre 网表。
- PDK 文件：
  - .lef：定义工艺层、单元尺寸、引脚位置。
  - .techfile：层名称到编号映射。
  - .simple：简化的层编号列表。
- 约束 (.sym, .symnet)：对称约束文件。
- JSON 配置：指定路径、参数、PDK 路径。
- 可选：初始 GDS、IO 文件等。

**注意**：MAGICAL 的 device_generation 子模块基于规则生成基本器件（如 MOS、Cap、Res），使用 gm/ID 方法或几何规则，不依赖 PDK 的 pcell 库。对于新 PDK，可能需要调整 device_generation 中的参数（如尺寸、层映射）或代码，以匹配工艺规则。示例中使用 mockPDK 作为模板。

#### 为新 PDK 修改 device_generation 的具体步骤
1. **修改 glovar.py**：
   - 创建新类，如 `new_pdk_glovar`，复制 `tsmc40_glovar` 的结构。
   - 更新 `min_w`（最小宽度）、`layer`（层映射）、`sp`（间距）、`en`（包围）、`ex`（扩展）等字典，以匹配新 PDK 的工艺规则。
   - 添加新层定义，如新金属层或特殊层（e.g., `layer['NEW_LAYER'] = 99`）。
   - 示例：如果新 PDK 有不同网格，修改 `GRID = 0.001`。

2. **修改器件类（如 Mosfet.py、Capacitor.py）**：
   - 更改导入：从 `from .glovar import new_pdk_glovar as glovar`。
   - 调整特殊规则：如 `dummy_l`、`crit_l` 等，根据新 PDK 的设计规则手册 (DRM) 更新。
   - 如果器件类型不同，修改 `__init__` 方法中的属性处理。

3. **创建新器件类**：
   - 如果新 PDK 有独特器件（如新型电容或电阻），复制现有类（如 Capacitor.py）为模板，创建新文件 `NewDevice.py`。
   - 定义新类继承 `basic`，添加特定几何生成逻辑。
   - 示例：对于新型 MOS，添加新属性如 `self.new_attr = True`，并在 `generate_layout` 中实现。

4. **添加定义**：
   - 在 glovar.py 中添加新全局变量，如 `new_rule = 0.05`。
   - 在器件类中添加新方法，如 `def custom_method(self):` 用于特定布局。
   - 更新 `basic.py` 如果需要通用功能。

5. **测试和验证**：
   - 运行示例，检查 GDS 输出是否符合新 PDK 规则。
   - 使用 DRC 检查工具验证布局。

#### 为新 PDK 修改其他模块的具体步骤
除了 device_generation，其他模块通常无需修改代码，只需提供正确的输入文件。但 flow 模块中的 PnR.py 需要更新导入，以使用新的 glovar 类。以下以引入 SKY130 PDK 作为示例（假设你已创建 sky130_glovar 类）：

1. **修改 flow/python/PnR.py**：
   - 更改导入语句：从 `from device_generation.glovar import tsmc40_glovar as glovar` 改为 `from device_generation.glovar import sky130_glovar as glovar`。
   - 示例：如果 SKY130 的最小金属宽度不同，PnR.py 中的 `self.halfMetWid = int(glovar.min_w['M1']*1000)/2` 会自动使用新值。

2. **anaroute 模块**：
   - 无需修改代码：该模块通过解析输入的 .lef 和 .techfile 文件来处理 PDK。
   - 准备 SKY130 的 .lef 和 .techfile 文件，确保层映射正确（e.g., M1 对应层编号）。

3. **ConstGen 和 IdeaPlaceEx 模块**：
   - 无需修改代码：这些模块基于约束和几何，不依赖特定 PDK 代码。
   - 如果新 PDK 有不同对称要求，调整 .sym 和 .symnet 文件。

4. **新电路设计（非新 PDK）**：
   - 无需修改任何模块代码：只需准备新的网表 (.sp)、约束文件 (.sym/.symnet) 和 JSON 配置。
   - 示例：对于新 ADC 电路，复制 adc1/ 目录，替换网表和约束。

5. **输入文件准备（SKY130 示例）**：
   - 下载 SKY130 PDK 文件（.lef, .techfile, .simple）。
   - 更新 JSON 配置中的 PDK 路径。
   - 确保 .techfile 中的层名称（如 OD, M1）与 glovar.py 中的字典匹配。

## 寄生感知

MAGICAL 不集成商业 PEX 工具，而是通过估算实现寄生感知和控制。适用于早期设计，重点相对寄生。

### 寄生估算方法
- **线长和面积计算**：在 `CirDB::computeNSetNetStatistics()` 中，计算每个网络的 wireLength（曼哈顿距离）和 wireArea（基于宽度和长度）。公式：线长 = ∑ |Δx| + |Δy|，面积 = 线长 × 宽度。
- **层效应**：考虑金属层（越高寄生越低），但不精确建模。
- **Steiner 树优化**：在路由中最小化总线长，间接控制寄生。

### 寄生控制机制
- **对称和匹配**：通过对称约束确保匹配网络寄生相等（如差分对）。
- **约束驱动路由**：GR/DR 优先最小化线长和层切换。
- **评估和迭代**：`computeTotalStatistics()` 输出总线长和面积，允许手动调整。

### 局限性
- 估算不精确，忽略耦合电容、工艺变异。
- 适用于相对控制，不适合绝对寄生分析。

## 放置作为 ILP

放置是将器件分配到网格位置，满足约束（如对称、间距），最小化成本（如线长、面积）。这是 ILP 问题。

- **变量**：器件位置 (x, y) 是整数（网格坐标）。
- **约束**：线性，如器件间距 ≥ d，对称轴 x = axis，边界 x ≤ max。
- **目标函数**：线性，如 ∑ 线长（曼哈顿距离）。
- **求解**：使用 LPSolve 求解 ILP。模拟 IC 中添加对称约束，使问题复杂，但仍线性。

ILP 适用于优化，但 NP-hard；SA 用于大规模。

## 实验结果

MAGICAL 的性能通过基准电路验证，与手动布局相当：

### 两级OTA
- **增益**：69.7 dB (MAGICAL) vs. 69.8 dB (手动)。
- **UGB**：1496 MHz vs. 1567 MHz。
- **偏移**：0.027 mV vs. 0.012 mV。

### CTDSM
- **功耗**：几乎相同。
- **SNDR/SFDR**：略有下降，但仍可接受。

观察：自动化布局性能接近手动，大幅缩短设计周期。

## 当前局限性

- **寄生建模**：缺乏全面耦合/IR降分析。
- **商业EDA集成**：无原生Cadence/Synopsys/Mentor链接。
- **先进节点支持**：在40nm验证，亚10nm FinFET未证实。
- **性能调优开销**：贝叶斯优化仍需多次迭代。

## 未来发展路线图

- **增强AI驱动寄生感知优化**。
- **OpenROAD集成**：用于混合信号协同设计。
- **FinFET就绪器件生成**。
- **扩展模拟/混合信号基准库**。

## 运行流程

1. 在示例目录准备输入（e.g., `adc1/`）。
2. 运行：`cd examples/adc1; source run.sh`。
3. 输出：.gds 文件、中间文件。

对于新设计，从示例调整 PDK/约束。