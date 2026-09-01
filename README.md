# LTE 辐射 参考灵敏度自动探底测试工具（LTE RefSens Radiated Emission Auto-Search）

一款基于 **.NET 8 + WPF** 的桌面自动化测试工具，通过 **SCPI over TCP** 驱动 **R&S CMW500** 综测仪，对 LTE 终端逐 Band、逐信道自动探测 **参考灵敏度（Reference Sensitivity, RefSens）**，并输出 Excel 报表与完整 SCPI 指令日志。

> 项目根命名空间/程序集名：`LteRefSensTester`　工程：`src/LTESenSearchRE.csproj`　解决方案：`src/LTESenSearchRE.sln`

---

## 目录

- [功能特性](#功能特性)
- [运行环境与硬件要求](#运行环境与硬件要求)
- [快速开始](#快速开始)
- [使用流程](#使用流程)
- [探底算法（RE4.3）](#探底算法re43)
- [连接与断连恢复机制](#连接与断连恢复机制)
- [支持的 Band 列表](#支持的-band-列表)
- [输出产物](#输出产物)
- [项目结构](#项目结构)
- [架构说明](#架构说明)
- [受保护的文件与代码段](#受保护的文件与代码段)
- [参考文档](#参考文档)

---

## 功能特性

- **全自动探底**：一次运行遍历所选 Band × {Low / Mid / High} 信道，自动收敛到参考灵敏度电平。
- **TPC 双通道测试**：分别在上行 **TPC = MAX（MAXP）** 与 **TPC = MIN（MINP）** 两种功控状态下探底，并给出两者差值（MAX − MIN）判定 PASS/FAIL。
- **三段式探底算法**：探底（±1 dB）→ 寻值（±0.5 dB）→ 确认（±0.1 dB），确认环节使用 3000 帧 Extended BLER 校验，确认轮数可选 1/2/3。
- **UE 上行发射功率采样**（可选）：在收敛电平处通过 `PMONitor:AVERage` 读取 UE TX Power。
- **每信道独立初始电平**：支持导入/导出 Excel 模板，为每个 Band 的 Low/Mid/High 单独设定探底起点。
- **CMW500 线损表管理**（可选）：从 CSV 导入频率-损耗表并上传、激活到 CMW500（RF1C，RX + TX）；另附独立的线损表编辑窗口，可编辑/新建多张表并以 `*.loss.json` 持久化到 `%AppData%/LteRefSens/LossTables`。
- **稳健的断连恢复**：仅依据 RRC 状态判定连接，掉线后自动执行小区重启 + SIM 卡软复位（`AT+CFUN=1`）的多轮恢复。
- **实时日志与进度**：滚动日志面板、进度计数、暂停/继续、可选记录完整 SCPI 交互。
- **自动落盘**：每次运行结果与 SCPI 日志自动保存到独立时间戳子目录。

---

## 运行环境与硬件要求

| 项 | 要求 |
|----|------|
| 操作系统 | Windows 10/11（x64） |
| 运行时 | .NET 8 Desktop Runtime（`net8.0-windows`，WPF） |
| 构建 | .NET 8 SDK + Visual Studio 2022 / `dotnet` CLI |
| 综测仪 | Rohde & Schwarz CMW500（LTE Signaling + LTE Measurement 选件） |
| 连接 | 以太网 TCP（SCPI Raw Socket），默认 `172.29.0.3:5025` |
| SIM 复位（可选） | 通过 USB CDC 串口（230400 8N1）连接的 SIM 模拟器 |

### NuGet 依赖

- `ClosedXML` `0.105.0` —— Excel 报表/模板读写
- `Microsoft.Extensions.DependencyInjection` `8.0.0` —— 依赖注入容器

---

## 快速开始

```bash
# 克隆后进入工程目录
cd src

# 还原并构建
dotnet build LTESenSearchRE.csproj -c Release

# 运行
dotnet run --project LTESenSearchRE.csproj
```

或使用 Visual Studio 2022 直接打开 `src/LTESenSearchRE.sln` 后 F5 运行（主窗口标题为 `LTESenSearchRE4.3`）。

发布（自包含/框架依赖）可参考 `src/Properties/PublishProfiles/FolderProfile.pubxml`。

### 自动构建与发布（CI）

`.github/workflows/release.yml` 提供「打 tag 自动发布单文件 EXE」流程：推送 `v*` / `V*` 版本标签（或手动触发）后，在 `windows-latest` 上执行 `dotnet publish -c Release -r win-x64 --self-contained -p:PublishSingleFile=true`，产出压缩后的单文件、自包含 EXE（目标机无需安装 .NET 运行时），并按版本号命名后发布到 GitHub Releases。

---

## 使用流程

1. **连接仪表**
   - 在「IP Address」中选择或输入 CMW500 地址（默认 `172.29.0.3`，端口 `5025`），点击 **Connect**。
   - 连接成功后程序自动下发全局初始化：`MSUBframes 0,10,2` 与 MEV 触发源 `FrameTrigger`（后续流程不再改动）。

2. **配置测试参数**
   - **TPC 设置**：勾选 `MAX` / `MIN`（至少勾选其一方可开始）。
   - **探底确认次数**：1 / 2 / 3（确认环节连续 BLER < 5% 的要求次数）。
   - **功率测试**（可选）：勾选后在收敛点采样 UE TX Power（每点约增加 4 秒）。
   - **初始 RefSens**（可选）：`导出模板` 生成 Excel，填入各信道起始电平后 `导入电平`；未导入时默认起点 −110 dBm。
   - **CMW500 线损表**（可选）：勾选后 `导出模板` 生成 CSV，填写后 `导入线损` 上传并激活。

3. **选择 Band**：在 FDD / TDD 分组中勾选目标 Band，可用 `All` / `None` 快速全选/清空。

4. **开始测试**
   - **Start (All Channels)**：遍历所选 Band 的 Low/Mid/High 三个信道。
   - **Mid Channel Only**：仅测每个 Band 的 Mid 信道（快速验证）。
   - 运行中可 **Stop**、**暂停/继续**。
   - 信道执行顺序：FDD 优先、TDD 其后，并按 Band 号排序。

5. **查看/导出结果**
   - 结果实时写入表格：`RefSens`、`Sen`（= RefSens + 27.8 dBm，10 MHz 满带宽等效功率）、`UE TX`、`MAX−MIN`、`Probes`、`Result`、`Note`。
   - `Export Excel` 手动导出报表；`导出 LOG` 导出运行日志；`记录 SCPI 指令` 勾选后可保存底层 SCPI 交互。
   - 每次运行结束自动落盘（见 [输出产物](#输出产物)）。

> **判定规则**：仅当某信道正常完成（DONE）且 MAX、MIN 均有效时，按 `(MAX − MIN) ∈ [−1, 1] dB` 判 **PASS**，否则 **FAIL**；`ERROR` / `SKIP` 透传状态。

---

## 探底算法（RE4.3）

算法对应参考文档 `docs/探底算法RE4.3.xlsx`，对每个信道、每个 TPC 相位（MAX/MIN）独立执行：

| 环节 | 步进 | 判定阈值 | 测量帧数 |
|------|------|----------|----------|
| **Step 1 探底** | ±1.0 dB | 2% / 20% | 1000 |
| **Step 2 寻值** | ±0.5 dB | 2% / 20% | 1000 |
| **Step 3 确认** | ±0.1 dB | 5% | 1000（收敛）+ 3000（出值校验） |

- **Step 1 探底**：以起始电平首测 BLER。
  - `2% ≤ BLER ≤ 20%`：直接进入 Step 3。
  - `BLER < 2%`：每次 −1 dB 下探，直到首次 `BLER ≥ 2%`。
  - `BLER > 20%`：每次 +1 dB 上抬，直到首次 `BLER ≤ 20%`。
- **Step 2 寻值**：仅当 Step 1 落点仍在 `[2%, 20%]` 之外时进入，以 ±0.5 dB 逼近。
- **Step 3 确认**：以 ±0.1 dB 收敛到 5% 附近，随后以 **3000 帧 Extended BLER** 连续校验 `confirmCount` 次（要求全部 `BLER < 5%`）方可出值；未通过则 +0.1 dB 重试。
- **悬崖检测（Cliff）**：若相邻两步出现「上一步 BLER < 1% 且本步 BLER > 90%」的突变，判定为掉线，触发重连后原点复测。
- **TPC 双相位**：先跑 MAX（若启用），MIN 相位（若启用）默认从 MAX 收敛值续测以缩短时间。

> 探底测量默认使用单次（SINGleshot）Extended BLER，通过轮询 `FETCh:...:EBLer:STATe?` 直到 `RDY` 后取 `FETCh:...:EBL:REL?` 计算 `BLER = (100 − Throughput) / 100`。

---

## 连接与断连恢复机制

连接与信道切换策略对应 `docs/CONNECTION_TEST_PROCEDURE.md`（只读参考）：

- **连接判据**：唯一依据 `SENSe:LTE:SIGN1:RRCState?` 返回 `CONN…` 视为已连接。
- **信道切换**：
  - **同制式**（FDD→FDD / TDD→TDD）：仅重设 BAND/信道/RB，不断开小区。
  - **跨制式**（FDD↔TDD）：断开 UE → 关小区 → 切双工模式与子帧偏移（SOFF 0↔2）→ 重配参数 → 完整 EBL 初始化 → 重启小区。
- **断连恢复（唯一恢复路径）**：记录当前电平 → 置 −85 dBm → 小区 OFF/ON → 最多 **16 轮**、每轮 10 秒内以 1 秒间隔轮询 RRC；每 4 轮（4/8/12/16）升级为「小区 OFF/ON + SIM 卡软复位」。恢复成功则还原电平续测，16 轮仍失败则跳过（SKIP）当前信道。
- **SIM 卡软复位**：枚举 USB CDC 串口（230400 8N1），逐口尝试下发 `cd uart3_proxy` → `AT+ECSIMCFG="SimSimulator",1` → `AT+CFUN=1`（逻辑移植自 `sim_simulator_tool.exe`）。

---

## 支持的 Band 列表

内置 22 个 Band，全部按 **10 MHz（50 DL RB）** 配置：

- **FDD（17 个）**：B1、B2、B3、B4、B5、B7、B8、B12、B13、B17、B18、B19、B20、B25、B26、B28、B66
- **TDD（5 个）**：B34、B38、B39、B40、B41

每个 Band 内置 Low/Mid/High 三个 EARFCN 及对应的 DL/UL RB 配置，详见 `src/ViewModels/MainViewModel.cs` 中的 `AllBands` 表。

---

## 输出产物

每次测试运行结束后自动生成独立时间戳目录：

```
<程序目录>/TESTDATA/Run_YYYYMMDD_HHmmss/
├── RefSens.xlsx     # 结构化结果报表（Band/Mode/Channel/EARFCN/RefSens/Sen/UE TX/Diff/Result…）
└── SCPI.log         # 本次运行的完整 SCPI 收发记录
```

- 初始电平 / 线损模板默认读写目录：`D:\RFtest\RFtestCabloss\`
  - 初始电平模板：`初始电平模板.xlsx`
  - 线损模板：`LossTable_Template.csv`
- `docs/RefSens_Example.csv` 为结果/模板格式示例。

---

## 项目结构

```
LTESensitivitySearchRE/
├── README.md                      # 本文档
├── .github/workflows/release.yml  # 打 tag 自动发布单文件 EXE（CI）
├── docs/                          # 参考文档（部分为只读）
│   ├── AGENTS.md                  # AI 协作约束（只读文件 / 锁定代码段清单）
│   ├── CONNECTION_TEST_PROCEDURE.md   # 连接测试操作流程（只读）
│   ├── RefSens_Example.csv        # 结果/模板格式示例
│   ├── 探底算法RE4.2 / RE4.3.xlsx  # 探底算法规格
│   └── 连接操作流程.xlsx           # 连接流程源表
├── research/                      # 算法/流程技术交底书生成脚本（python-docx）
│   ├── gen_patent_doc.py
│   └── gen_patent_doc_v2.py
└── src/                           # 应用源码（WPF, .NET 8）
    ├── App.xaml(.cs)              # 入口 + DI 容器装配（无 StartupUri，代码驱动启动）
    ├── MainWindow.xaml(.cs)       # 主界面
    ├── LossEditorWindow.cs        # 线损表编辑窗口（纯代码构建）
    ├── SimpleInputDialog.cs       # 通用单值输入对话框
    ├── Commands/                  # RelayCommand / AsyncRelayCommand
    ├── Models/                    # BandConfig / ChannelPoint / DuplexMode / TestConfig / RefSensRow / LossTableEntry
    ├── ViewModels/                # MainViewModel / ViewModelBase
    ├── Services/                  # CmwService(ICmwService) / TestService(ITestService) / LogService(ILogService)
    ├── Helpers/                   # BuildInfo 等
    └── LTESenSearchRE.csproj/.sln
```

---

## 架构说明

- **MVVM**：`MainWindow`（View）↔ `MainViewModel`（ViewModel，数据绑定 + 命令）↔ Models。
- **依赖注入**：`App.OnStartup` 通过 `Microsoft.Extensions.DependencyInjection` 将服务与 ViewModel 注册为单例（`src/App.xaml.cs`）。
- **服务层**：
  - `CmwService`：CMW500 的 TCP/SCPI 通信（异步收发、`*OPC?` 同步、超时重试、`SemaphoreSlim` 串行化、可选 SCPI 记录）。
  - `TestService`：探底算法、信道切换、EBL/OCNG 管理、UE 功率采样、断连恢复、SIM 复位。
  - `LogService`：日志事件分发。
- **命令**：`AsyncRelayCommand` / `RelayCommand` 承载 UI 交互。

---

## 受保护的文件与代码段

依据 `docs/AGENTS.md`，以下内容为锁定项，**修改需谨慎**（AI 协作时禁止改动）：

- **只读参考文档**：`CMW500_CONNECTION_SPEC.md`、`CONNECTION_TEST_PROCEDURE.md`。
- **锁定代码段**（`src/Services/TestService.cs`）：`InitEblFullAsync()`、`RefreshEblAsync()` 及其信道切换/EBL 管理规则（同制式仅换参、跨制式重启小区、首通道 CEST 跳过初始化等）。

---

## 参考文档

- `docs/探底算法RE4.3.xlsx` —— 当前生效的探底算法规格（本工具实现依据）。
- `docs/CONNECTION_TEST_PROCEDURE.md` —— 连接与信道切换操作流程。
- `docs/AGENTS.md` —— 协作约束与锁定项清单。
- `research/gen_patent_doc*.py` —— 由算法/流程生成技术交底书（Word）的脚本。
