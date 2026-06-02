# BUILD-KICKOFF — BSV 扑克平台（给 Claude Code 的指令）

你是 **Claude Code**。请依据本仓库中的规格说明构建 BSV 扑克平台。这些规格说明是
**权威性的**；你负责实现它们。**所有内容都由你来构建和运行** —— 代码、测试、向量、
真实的 Script 解释器、VM 镜像、安装程序 —— 并且你要**提交（commit）**你的成果。
规格说明的作者不运行任何东西；那是你的工作。

## 权威输入 —— 在编写任何代码之前请完整阅读这些文件
- `bsv-poker-spec.md` —— 协议/密码学/交易核心：引擎、mental-poker 密码学、BSV
  交易/Script 模型、SDK（§15）、钱包/托管、网络/发现、自包含的 VM（§10）、依赖契约（§2）、
  需求登记册（§19.F）。
- `bsv-poker-app-architecture.md` —— 应用层：Windows 桌面端 + Web 客户端、
  **多游戏**平台（Hold'em、Omaha、Stud、Draw、Razz，以及计划中的 Blackjack）、各屏幕、
  发现、NFT/微支付接缝、NFR、安全、错误/故障处理、可观测性、打包、
  **测试规格**（§A16）、文档标准（§A15）、需求登记册 + 可追溯性 + 验收门禁（§A18），以及
  **作者与构建者的职责划分**（§A19 —— 每一项"运行"任务都是你的）。
- `bsv-poker-spec-redteam-01.md` —— 红队评审 01（已应用；**请勿**重新引入
  B1 的 `w_j` 重建不一致问题，或 B2 的发牌与选牌错误）。
- `HANDOVER.md`、`bsv-poker-spec-HANDOVER.md` —— 上下文与委托人的规则。
- `handeval_oracle.py` —— 手牌评估预言机；你要**运行**它来生成 §19.D
  向量。

凡是规格说明标注 `DECISION REQUIRED` 之处，遵循其陈述的默认值并记录一条 ADR。
不要臆造协议。如果你发现规格说明中存在真正的歧义，请记录它并选择规格说明的
默认值 —— 不要停下来询问。

## 不可商量的规则 —— 违反即为构建缺陷，而非风格选择
1. **仅限 BSV，后 Genesis（post-Genesis）。** 没有 BTC 代码，没有 BTC 假设。
   `OP_CHECKLOCKTIMEVERIFY`/`OP_CHECKSEQUENCEVERIFY` 是 **NO-OPS**（空操作）—— 所有计时都在
   **交易层级**强制执行（在原始替换规则下使用 `nLockTime` + `nSequence`），
   绝不在脚本内进行。
2. **OP_RETURN 在任何地方都被禁止。** 每一个承诺/锚点都是**活动**脚本中的 push-data
   （`<data> OP_DROP` 前缀，或解锁脚本中的一次 push）。任何 `OP_RETURN` 输出都属于
   拒绝项。添加一个 CI/lint 检查：如果 `OP_RETURN` 操作码（0x6a）出现在任何锁定或解锁脚本中，
   就让构建失败。
3. **零捏造。** 每一个数字、向量和字节大小都**由运行代码生成**，
   并作为可复现的向量提交 —— 绝不凭记忆写出。`reproduce` 命令
   会重新生成全部这些内容，并在任何不匹配时以非零状态退出。
4. **真实解释器测试。** Script 花费通过真实的 BSV Script 解释器运行，使用
   **Genesis 规则**；否定测试**必须**在解释器**内部**失败，绝不在包装器
   守卫中失败；签名抽查不是可接受的替代方案。
5. **不夸大；枚举信任面。不隐藏假设** —— 将决策记录为
   ADR。**不道歉。**
6. **可追溯性完整。** 每一项需求（核心 `REQ-*` 和应用 `REQ-APP-*`）都映射到
   代码和一个通过的测试；CI 在任何未测试的需求或未追溯的共识/安全
   源文件上失败。
7. **工程标准：** NASA NPR 7150.2 保障实践 + 一份**有文档记录的** Power-of-Ten
   适配（规则 3 和原始指针规则在 GC 的 TS/Go 运行时中为 N/A —— 明确说明这一点）+
   Microsoft SDL。

## 依赖 —— 已存在于工作区中；**请勿**重新核验它们的存在
通过适配器/SDK 契约 **CT/BS/VA/OB**（核心 §2）针对 prof-faustus 仓库进行构建：
`cardtable`（mental-poker 基底 —— 使用其**原语**，而非其游戏）、
`bonded-subsat-channel`（亚聪通道 + **嵌入式 BSV 节点** = regtest 后端，
D6）、`verifiable-accounting` ×3、`overlay-broadcast`。绑定到契约，绝不绑定到内部实现。
单一的契约一致性套件**必须**针对**伪实现（fakes）**和**真实**
适配器**两者**都通过；安全关键路径（洗牌、揭示单次使用、公平博弈、签名）都针对
**真实**实现测试，绝不针对伪实现。

## 构建顺序 —— 在每个门禁处提交
按照分阶段路线图（核心 §17、应用 §A21.9）：
- **阶段 0 —— 基础。** 搭建 monorepo（核心 §16 / 应用 §A2.4）：`/spec`、
  `/packages/*`（protocol-types、engine、hand-eval、game-*、crypto-mentalpoker、
  script-templates-ts、tx-builder、wallet-custody、adapters、sdk、ui-core、app-services）、
  `/apps/{client-web,client-desktop,relay-go,indexer-go}`、`/vm`、`/tests`。适配器 +
  受一致性约束的伪实现，用于 CT/BS/VA/OB。VM 拉起 node(regtest)+relay+空客户端；
  `reproduce` 通过（绿色）；可追溯性骨架就绪。**门禁：** VM 端到端启动，自检
  通过，CI 全绿。**提交并打标签。**
- **阶段 1 —— 首个可玩：regtest 上的一对一无限注德州扑克（heads-up NL Texas Hold'em），
  Windows + Web，带发现。** 熵承诺/揭示；分布式洗牌；加密发牌；完整的
  翻牌前→河牌下注 FSM；最小揭示摊牌；结算；决策 + 恢复
  超时；**不揭示弃牌**；relay + LAN 发现；Tauri Windows 外壳以及
  Web 外壳，共享同一套 UI core；记录文本（transcript）+ 确定性回放。**通过运行
  `handeval_oracle.py` 生成 §19.D 向量。** 运行解释器层级的脚本测试（Genesis）、
  对抗性子集、VM 中的 E2E，以及 `reproduce`。**门禁：** 核心 §14.7 / 应用
  §A18.3，在桌面端和 Web 构建**两者**上。**提交并打标签。**
- **阶段 2+ 以及其他游戏**（Omaha、Stud、Draw、Razz；然后在其 §A21.7
  独特的无荷官模型确定之后再做 Blackjack）按应用 §A21.9 —— 每个一个 `GameModule`，每个带其
  变体的牌桌视图配置、**生成的**手牌评估向量、下注结构（PL/FL），以及
  其验收门禁。**每个游戏分别提交。**

## 运行与构建义务 —— 由你来执行
- 生成向量：`python3 handeval_oracle.py` → 嵌入已核验的输出；`reproduce` 检查
  它。为新向量扩展该预言机；绝不手写向量。
- 通过真实解释器构建并**测量** Script 模板（核心 §19.C）；将字节预算
  作为可复现向量提交。
- 构建自包含的 VM 镜像（容器；可选 VM 镜像），带一条命令的
  引导启动（核心 §10、应用 §A14）。
- 从同一个 commit 构建**已签名的 Windows 安装程序（Tauri）**和 **Web 包**；
  记录制品哈希（应用 §A14）。
- CI 阶段（应用 §A14.2）：typecheck → lint（含 OP_RETURN 缺失检查）→ 单元+属性
  → 解释器层级（Genesis）→ 集成 → 构建镜像 → 镜像内 E2E → `reproduce` →
  无障碍 + 安全 → 可追溯性。红色阶段阻止合并。

## 提交 / 加载纪律 —— 这就是"提交"步骤
- 如果仓库尚未初始化，执行 `git init`。以小的、可评审的单元提交，配以
  描述性的提交信息。**绝不提交红色 CI。**
- 在每个阶段/门禁处以及每个一致性/测试里程碑处提交。为阶段 0 和
  阶段 1 的验收 commit 打标签。
- 保持 ADR、需求登记册和可追溯性矩阵已提交且为最新。

## 报告
在每个阶段之后报告**诚实**的指标 —— 文件、测试、需求覆盖率、生成的
向量哈希、VM/安装程序哈希 —— 以及剩余内容。绝不声称你并不拥有的完成度。

**从阶段 0 开始。首先完整阅读 `bsv-poker-spec.md` 和 `bsv-poker-app-architecture.md`。**
