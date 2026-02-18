# Cael v0.1 实现方案（编译到 Haskell）

本计划描述 **编译器（Haskell 实现）** 与 **运行时库（Haskell 包）** 的完整实现方案，覆盖：
- 语言前端：布局/解析/AST/模块系统
- 语义层：去糖、名字解析
- 静态分析：类型推断、typeclass、效果行、阶段、流量、量纲、精炼片段
- 降级：从 Surface AST 到 Core IR
- 代码生成：Core IR 生成 Haskell 模块
- 构建与工具：CLI、GHC 调用、测试生成、verify/check
- 运行时库：Eff/Handler/Crystal/Temporal/Quantity/Ctx/并发等

> 约束：本文件不包含编译器代码。

---

## 1. 交付物与仓库结构

### 1.1 交付物
1) `cael` 可执行文件（编译器 + CLI）
2) `cael-prelude` Haskell 包（运行时/标准库）
3) 文档与示例工程

### 1.2 建议目录
- `compiler/`
  - `Cael.CLI`
  - `Cael.Parser`（layout + parser）
  - `Cael.Syntax`（CST/Surface AST）
  - `Cael.Rename`（名字解析）
  - `Cael.Desugar`
  - `Cael.Typecheck`（HM + 约束）
  - `Cael.Analysis`（flow/dimension/temporal/refinement）
  - `Cael.Core`（Core IR）
  - `Cael.Lower`（Surface/Typed -> Core）
  - `Cael.Codegen.HS`（Haskell 输出）
  - `Cael.Build`（GHC/cabal 调用）
  - `Cael.Diagnostics`
- `runtime/`（cael-prelude）
  - `Cael.Prelude`（基础类型）
  - `Cael.Eff`
  - `Cael.Crystal`
  - `Cael.Temporal`
  - `Cael.Quantity`
  - `Cael.Ctx`
  - `Cael.Concurrency`
  - `Cael.Testing`
- `examples/`

---

## 2. 编译管线总览

### 2.1 编译阶段（每个模块）
1) **读取源码**（含编码/换行归一）
2) **词法 + 布局（layout）**：产生 token 流（含 INDENT/DEDENT/NEWLINE）
3) **解析**：token -> CST/Surface AST（保留位置信息 span）
4) **模块加载**：解析 import，构建依赖图，拓扑排序
5) **名字解析（rename）**：
   - 建立符号表（导出/导入/局部）
   - 解析所有 Ident 到全限定名
6) **去糖（desugar）**：
   - 管道/条件管道/分支管道
   - 多子句/guard -> case/if
   - match -> case
   - do 语句规范化
   - `?`、`transition`、`parallel`、`select` 规范化成更少节点
7) **类型推断（HM + typeclass + 约束收集）**
8) **附加静态域检查**：
   - 效果行（membership constraints）
   - temporal stage 检查（状态转移、transition）
   - flow contracts（显式流）
   - dimension algebra（量纲一致性与插入转换）
   - refinement/evidence（可判定片段的蕴含）
   - inv（InvCore 检查或 inverse 必填）
   - synth（候选搜索与属性回归）
9) **Lowering 到 Core IR**
10) **生成 Haskell AST/文本**（每模块一个 `.hs`）
11) **构建**：调用 GHC（或 cabal/stack）编译生成可执行/库
12) **可选**：生成检查报告（flow 图、dimension 化简、temporal stage 图、crystal inspect 输出）

---

## 3. 解析器与布局规则（Layout）

### 3.1 布局目标
- 支持 `module ... where`、`data`/`effect`/`class`/`instance`/`handler`/`do`/`match`/`select` 的缩进块
- 允许顶层声明序列与嵌套块
- 错误信息要能指出“期望缩进/缺少 DEDENT”的位置

### 3.2 Layout 算法（建议）
- 词法阶段计算每行 leading spaces（tab 直接禁止或固定为 2/4 空格）
- 引入栈 `indentStack`：
  - 进入块：遇到触发布局的关键字后，下一行缩进更深则 push，否则空块报错/允许（按设计）
  - 换行：比较当前缩进与栈顶
    - 大于：发 INDENT 并 push
    - 等于：发 NEWLINE
    - 小于：反复发 DEDENT 直到匹配或报错
- 括号 `()[]{}` 内禁用布局（常见规则）

### 3.3 解析技术选型
- 组合子解析器（Megaparsec）适合表达式优先级与良好错误信息
- 或 Happy/Alex（更接近传统编译器），但错误体验通常更差

---

## 4. AST 设计与中间表示

### 4.1 三层表示
1) **Surface AST**：贴近语法（包含管道、match、handler、synth、inv 等）
2) **Typed AST**：为每个表达式节点附加
   - 推断类型 `Type`
   - 效果行 `EffRow`
   - flow tag 信息
   - dimension 信息
   - stage 索引信息
   - refinement 环境快照（用于诊断）
3) **Core IR**：极小核心，方便生成 Haskell
   - `Var`, `Lam`, `App`, `Let`
   - `Case`（带 pattern）
   - `Ctor`/`Record`
   - `PrimOp`
   - `EffSend`, `EffHandle`（或 lowered 到 runtime API 调用）
   - `ResultBind`/`ResultRaise`（也可统一到 monadic bind）
   - 注解节点（类型/位置）

### 4.2 位置与诊断
- 所有节点携带 `Span {start,end,file}`，诊断系统可做：
  - 指向具体 token
  - 输出上下文行
  - 给出修复建议（例如缺少 `inverse`、`mut` 逃逸等）

---

## 5. 模块系统与名字解析

### 5.1 依赖图
- 扫描入口模块（CLI 指定）
- 递归解析 import，构建 DAG
- 拓扑排序；若发现环，报错并输出环路径

### 5.2 符号表
区分命名空间：
- 值（term）
- 类型（type constructors）
- 构造器（constructors）
- 类型类（classes）
- 效果（effects）与 op
- flow tag、unit、dimension、contract、context、error hierarchy 名称

处理规则：
- `pub` 控制导出；未导出即不可见
- import 可选择性导入与别名
- 局部作用域（lambda、let、pattern bind）遮蔽外层

---

## 6. 去糖（Desugaring）规范

### 6.1 管道
- `x |> f` -> `f x`
- `x |> f a` -> `(f a) x`
- 链式左结合

### 6.2 条件管道
- 解析为 `CondPipeExpr base [clauses]`
- 去糖为 `if p1 base then f1 base else if p2 base then f2 base else ...`
- 默认 `|!> _ => g` 必须出现（或编译器补默认报错分支）

### 6.3 分支管道
- `x |>> (f, g)` -> `(f x, g x)`

### 6.4 多子句函数 + guard
- 统一成单个 `case`/`if` 链
- 做模式覆盖检查与不可达分支告警（typed 阶段更好做）

### 6.5 `?` 与 Result
- 将 `e?` 标准化为 `try e` 的 Core 形式，附带“错误提升”信息
- 后续 lowering 时变成对 runtime `bindResult`/`mapErr` 的调用

### 6.6 `transition @Stage do ...`
- 去糖为 runtime 的 `ensureStage`/`bracketStage` 形态（见 9.2）
- 仍需静态 stage 追踪保证“正常路径达成目标 stage”

---

## 7. 类型系统实现（HM + typeclass + 约束求解）

### 7.1 主类型推断（HM）
- `let` 绑定泛化（generalization）
- lambda 参数单态
- 应用生成新类型变量并统一（unify）
- 显式 `::` 做检查（check）而不是推断（infer）
- 支持代数数据类型与模式匹配统一

### 7.2 typeclass
- 采用 “Haskell 风格字典传递” 的实现策略：
  - 类型检查阶段收集 class 约束
  - 代码生成阶段可：
    - 生成显式字典参数（最直接）
    - 或依赖 GHC 自身的 typeclass 解析（前提：生成的 Haskell 用 class/instance 表达并让 GHC 解决）
- v0.1 推荐：**尽量把 class/instance 直接生成 Haskell**，让 GHC 做实例解析；编译器只需做 Cael 侧一致性检查（避免生成明显不合法代码）。

### 7.3 约束框架
类型推断输出：
- `Type`（主类型）
- `ClassConstraints`
- `EffectConstraints`（`Console ∈ eff`）
- `HasConstraints`（context lenses）
- 以及附加域的约束（flow/dim/stage/refine）

约束求解顺序建议：
1) HM unify 得到主类型
2) class/Has/effect 约束正规化（去重、简化）
3) 附加域检查（需要主类型形状）

---

## 8. 附加静态域

### 8.1 效果行（Effect Row）
目标：支持 `(Console ∈ eff) => Eff eff a` 形式，并能在 `handle` 后消除效果。

实现策略（v0.1）：
- 将 `eff` 视为抽象的“集合变量”
- 只支持两类约束：
  - membership：`E ∈ eff`
  - row 等价（在类型注解里出现时）
- 不做复杂 row 多态运算（例如行差/并的类型级计算），而是将其交给 runtime 表示（例如 `Eff eff a` 中 eff 是 phantom）

检查：
- 若表达式里调用 `Console.print`，则环境必须含 `Console ∈ eff`
- `handle h prog`：在类型上把某个效果从 `eff` 中“消除”（v0.1 可表现为改变 phantom 或直接交给 runtime 类型）

### 8.2 Temporal Stage（阶段类型）
实现目标：
- `temporal data` 生成阶段索引的表示（推荐 GADT）
- 函数签名可要求 `T @Stage`
- `transition` 块静态追踪阶段变化

静态阶段追踪（建议做法）：
- 给每个 temporal 类型维护一套“阶段变换函数”：
  - 例如 `open : Conn @Created -> Eff ... (Conn @Opened)`
- 类型检查时，变量在 `do` 块中被重新绑定时（`conn <- conn.open ...`），其类型随之更新
- `transition @Closed do ...`：
  - 正常路径最后表达式必须处于目标 stage（或能证明最终已 close）
  - 若中途有早退（Result Err / Eff abort），则由 runtime ensure 清理（但 v0.1 最好仍要求显式 close，除非你引入结构化资源 API）

### 8.3 Flow Contracts（显式数据流）
表示：
- `T @Tag` 作为类型修饰（compile-time）
- flow rule 图：边带 “requires f >> g” 的证明路径（用于解释错误信息）

检查点：
- 函数应用：实参 tag 必须可流向形参 tag
- 返回值：函数声明的返回 tag 必须满足表达式 tag
- sink（Print/Serialize/Log）：禁止 `@Sensitive` 流入（除非先 `redact`）

v0.1 不做：
- 控制流隐式流（if 分支泄漏）的强安全保证（可留作严格模式扩展）

### 8.4 Dimensional Types（量纲单位）
表示：
- 维度表达式规范化为基维度指数映射（如 `Length^1 * Time^-2`）
- 单位到基单位的比例因子（编译期常量或运行时值）

检查与转换插入：
- `+/-`：要求维度相同；若单位不同，插入 `convertTo`（纯函数）
- `* /`：合成新维度并化简
- `convert`：需要相同维度；若汇率类运行时转换，要求显式 `ExchangeRate` 参数

代码生成：
- 用 runtime `Quantity` newtype 表示，并携带数值与单位 phantom
- 插入的转换以乘除常量因子实现（或调用库函数以保可读性）

### 8.5 Refinement / Evidence（可判定片段）
v0.1 谓词语言：
- 数值线性约束（`x > c`、`x >= y + c`）、合取为主
- 对 `check` 引入的事实做简单蕴含（区间合并、传递性等）
- 可选：对线性算术调用 SMT（如 Z3），但需保证可控（超时/回退策略）

降级策略：
- 复杂谓词（字符串 contains/regex）不进入 refinement；使用 `contract + seal`
- 用户可写 `assume` 强制引入证据（标记为不安全并在诊断中高亮）

---

## 9. 高级特性落地（v0.1 语义）

### 9.1 `inv`（可逆函数）
实现要点：
1) 解析 `inv f :: ...` 与 `inv f x = ...`（以及可选 `inverse ... = ...` 子句）
2) 若无 `inverse`：
   - 对函数体做 **InvCore 语法/结构检查**
   - 在通过时，自动构造逆函数 AST（或直接生成 Haskell 逆实现）
3) 若有 `inverse`：
   - 检查正逆类型一致性
   - 可选 verify：对有限输入域抽样/属性测试；或对可符号片段做等式验证

InvCore 建议实现为一个小 IR：
- 可逆原语：线性变换、记录/元组重排、可逆 concat（需长度前缀/无歧义分隔）
- 禁止一般递归与丢信息分支
- `.` / `|>` 组合在 InvCore 上闭包

`~f` 实现：
- 若 `f` 为 inv，解析为对其逆函数的调用
- 组合函数 `g = f1 . f2` 若两者 inv，则可推出 `~g = ~f2 . ~f1`（结构性）

### 9.2 Crystal（效果结晶）
核心决策：结晶体是“可检查/可编辑的效果程序 AST”。

实现组件：
- runtime `Crystal` 数据结构：顺序、绑定、op 节点、返回节点
- `crystallize do ...`：
  - 编译器不生成 `Eff` 执行，而生成构建 `Crystal` AST 的纯表达式
- `inspect`：遍历 AST 输出步骤
- `crystalMap`：在 AST 上做 rewrite
- `simulate`：解释 AST，但用用户给的替代规则（纯）
- `melt`：解释 AST 到 `Eff`/`IO`

Serializable 子集检查：
- 若用户调用 `serialize/remoteExecute`，编译器检查：
  - Crystal 内节点参数/返回必须是可序列化（存在 Serialize typeclass 或内建规则）
  - 禁止捕获函数值与不可移植引用

### 9.3 `synth`（有限空间合成）
v0.1 实现为“受控搜索”：
- 候选空间：由一组可配置组合子与小常量构成（例如 `map/filter/fold/words/unwords/capitalize` 等）
- 搜索策略：
  - 枚举表达式（按大小递增）
  - 先用 examples 过滤（必须全部匹配）
  - 再用 constraints 生成的属性测试验证（随机/小范围穷举）
- 失败时输出：
  - 当前最接近的候选
  - 最小反例
  - 建议：补充 examples / 放宽 constraints / 增大搜索深度 / 启用更多组合子集

编译产出：
- synth 成功：生成一个普通函数定义（写入生成的 Haskell 或写回中间文件）
- synth 失败：编译失败（除非用户配置允许生成 stub）

---

## 10. 并发模型实现（structured concurrency）

基于 Haskell runtime：
- `parallel (a,b,c)`：
  - runtime 提供 `parallel3` 等组合器，内部用 `async`/`Concurrently`
  - 失败传播：任一子任务异常/Err -> cancel 其余
- `Channel a`：
  - 用 STM `TBQueue`（有界）实现
  - `close` 语义：额外状态位；`forEach` 读取直到关闭
- `select`：
  - 编译为 STM `orElse` 链
  - `after (t)` 用 `registerDelay` 或定时器 channel

与 Eff 集成：
- 提供 `Concurrency` effect handler 到 IO/STM
- 或让并发 API 直接在 `Eff` 中运行（由 runtime 提供 lift）

---

## 11. 代码生成到 Haskell（Codegen）

### 11.1 目标
- 每个 Cael 模块生成一个 Haskell 模块
- 生成可读、可调试的 Haskell（保留源位置信息到注释/调试表）
- 尽量把 typeclass/ADT 直接用 Haskell 表达，让 GHC 做后续类型检查与优化

### 11.2 映射规则（核心）
- `data` -> `data` / `newtype`
- `temporal data`（阶段型）-> `GADT`（推荐）
- `effect` -> 一组 effect 标识与 op 类型（依赖 cael-prelude 的 Eff 实现方式）
- `do` -> Haskell 的 do（若 runtime Eff 是 monad）或显式 bind
- `handler`/`handle` -> runtime API 调用
- `Result`/`?` -> runtime `ResultT` 风格或显式 case
- `Quantity`/dimension -> `newtype Quantity dim unit a`
- `Ctx`/context -> `ReaderT` 风格
- `test/property` -> 生成一个测试入口模块（Hspec/QuickCheck）

### 11.3 Haskell 后端选择（Eff 表示）
v0.1 推荐：
- 使用 freer/free 风格 Eff（在 runtime 实现）
- 代码生成侧只需调用 `send/handle` 等 API，而无需复杂类型级行运算

---

## 12. 构建系统与 CLI

### 12.1 CLI 子命令
- `build`：生成 `.hs` + 调用 GHC 构建
- `run`：build 后运行
- `check`：只跑到类型与静态域检查
- `test`：生成测试入口并运行
- `verify`：跑 inv/synth/crystal 可选验证
- `fmt/doc`：可选（不影响 v0.1 核心）

### 12.2 GHC 调用方式
两种路线：
1) **外部进程调用 `ghc`/`cabal`**（实现简单、隔离好）
2) 使用 **GHC API**（可做增量编译与更细诊断，但复杂）

v0.1 建议从外部调用开始。

---

## 13. 运行时库（cael-prelude）设计

运行时库是 v0.1 成功的关键：编译器应尽量“薄”，复杂语义下沉到 runtime。

### 13.1 必备模块与职责
- `Cael.Prelude`
  - 基本类型别名、常用函数、`Result`
- `Cael.Eff`
  - `Eff eff a`、`send`、`Handler`、`handle`、`resume`
- `Cael.Crystal`
  - `Crystal eff a`、`crystallize` 构建器、`inspect/crystalMap/simulate/melt`
- `Cael.Temporal`
  - `Temporal a`、`current/set/diff/atVersion/previous`
- `Cael.Quantity`
  - `Quantity`、单位转换工具、维度 phantom
- `Cael.Ctx`
  - `Ctx`、`runCtx`、`ask`、`Has` 投影
- `Cael.Concurrency`
  - `parallel`、`spawn`、`Channel`、`select`
- `Cael.Testing`
  - test/property 注册与运行（或直接依赖 Hspec/QuickCheck）

### 13.2 与编译器的接口契约
- 编译器生成的 Haskell 只依赖 runtime 提供的稳定 API
- runtime 中每个 API 都要有清晰的类型，便于 GHC 二次检查
- 对不可序列化 Crystal、`assume`、未验证 inv 等行为，runtime 也应提供标记类型/日志钩子，便于 verify 工具消费

---

## 14. 测试与质量保证

### 14.1 编译器自身测试
- parser：golden tests（输入 Cael -> AST pretty）
- desugar：golden tests（Surface -> Core）
- typecheck：正例/反例（错误信息快照）
- codegen：生成的 Haskell 能通过 `-Wall`（或至少无致命警告）

### 14.2 语言特性回归
- temporal stage：错误用法必须被拒绝
- flow：敏感数据流入 sink 必须报错
- dimension：错误量纲运算必须报错
- crystal：serialize 子集检查必须正确
- inv：InvCore 判定与 inverse 类型一致性检查

---

## 15. 实施顺序（不含日期）

建议按“可运行最小闭环”推进：

1) **最小闭环**
- layout + parser（模块/let/函数/data/match/do）
- 生成最简单 Haskell（不含 Eff）
- 用 GHC 构建并运行

2) **类型系统闭环**
- HM 推断 + ADT + 模式匹配
- type signature 检查
- typeclass（先生成到 Haskell，让 GHC 解决）

3) **效果系统**
- effect/handler/handle/do 的类型与 codegen
- runtime Eff 实现稳定

4) **Result/? 与 error hierarchy**
- Result 语法糖与自动提升

5) **context lenses**
- context 声明、Has 约束、Ctx 运行器

6) **dimension**
- 维度代数 + unit 声明 + 插入转换

7) **flow**
- tag 语法与规则检查 + sink 限制

8) **temporal stage**
- temporal data -> GADT 生成
- transition 静态/运行时保障

9) **Temporal value**
- Temporal 历史结构与 API；语法糖 `@(-1)` 等（或库函数）

10) **crystal**
- crystallize/inspect/simulate/melt
- Serializable 子集与 serialize 限制

11) **inv**
- InvCore 子集判定与自动逆生成
- inverse 显式路径与 verify（可选）

12) **synth**
- 候选空间 DSL + 搜索 + 失败诊断
- 与 property 约束联动

13) **并发**
- parallel/channel/select 的语法与 runtime 封装

14) **测试框架**
- test/property 收集与 runner 生成

---

## 16. 风险与边界（v0.1 明确不做/后续再做）

- 完整 refinement 证明（任意谓词、任意量词）不做
- 隐式信息流（控制流泄漏）不做强保证
- 通用程序合成（大规模 DSL、交互式问答、置信度模型）不做
- Crystal 捕获任意闭包并可跨机器序列化：仅做 Serializable 子集
- 多后端（wasm/js/native）不作为 v0.1 核心承诺；可通过不同 GHC 后端实验性支持

---

## 17. 成功判据（Definition of Done）

- `cael build/run/check/test` 在示例集上可用
- 语言参考（README 语法）与实现行为一致
- 关键错误信息可读（指出位置、原因、修复建议）
- 生成的 Haskell 可用 GHC 成功构建
- runtime API 稳定，编译器不需要“知道太多运行时细节”
