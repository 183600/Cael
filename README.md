# Cael Language v0.1 (Implementable Spec)

Cael 是一门以 **纯函数 + 强类型推断** 为核心、以 **代数效果** 为主轴、并在类型层面扩展 **阶段（Temporal Stage）/ 流量标签（Flow Tags）/ 量纲单位（Dimensions）/ 契约类型（Contracts）/ 错误层叠（Error Cascade）/ 上下文透镜（Context Lenses）** 的语言。

本仓库包含：
- `cael`：编译器（用 Haskell 编写），**把 Cael 编译到 Haskell**
- `cael-prelude`：运行时与标准库（Haskell 包）
- 文档：本 README（语言与语法）、PLAN（编译器与运行时实现方案）

> v0.1 的原则：**所有静态特性必须可判定、可实现、可工程化落地**。因此对“可逆函数自动求逆 / 精炼证明 / 意图合成 / 结晶体序列化”等特性做了明确的子集限制与降级路径（详见本文）。

---

## 1. 快速开始

### 1.1 安装（开发者模式）
依赖：GHC、cabal（或 stack）。

```bash
cabal build all
cabal install cael
```

### 1.2 第一个程序

`App/Main.cael`

```cael
module App.Main where

effect Console where
  print    :: String -> ()
  readLine :: () -> String

main :: (Console ∈ eff) => Eff eff ()
main = do
  print "What's your name?"
  name <- readLine ()
  print ("Hello, " <> name <> "!")
```

运行：

```bash
cael run App.Main
```

---

## 2. 语言概览（What’s in v0.1）

### 已落地（v0.1 目标）
- 纯函数默认、不可变绑定
- ADT（data/record/variant）、模式匹配、guard、`match`
- Hindley–Milner 风格推断 + typeclass（Haskell 风格）
- 代数效果：`effect`、`Eff`、`handler`、`handle`、`do`
- 管道与组合：`|>`、`.`、分支管道 `|>>`、条件管道 `|?>` / `|!>`
- Temporal Stage（阶段类型）：`T @Stage`（GADT 生成）
- Temporal Value（带历史）：`Temporal a`（版本索引；时间点查询为库函数）
- Flow Contracts（显式数据流）：`T @Tag` + 规则检查（不含隐式流）
- Dimensional Types：`Float <km/hr>`，编译期量纲代数检查 + 同维度单位自动换算
- Contract Types：`contract` + `seal`（运行时验证一次）
- Error Cascade：`error hierarchy` + `?` 自动提升
- Context Lenses：`context` + `Has` + `Ctx`（Reader 风格）
- 结构化并发（基于运行时库封装 async/STM）：`parallel`、`Channel`、`select`
- 测试：`test` / `property`（生成到 Haskell 测试入口）

### 受限落地（有清晰子集）
- 可逆函数 `inv`：
  - **仅当函数体属于 InvCore 子集**时允许编译器自动生成逆 `~f`
  - 否则必须显式提供 `inverse ... = ...`
  - 等式保证以“结构性可逆性检查”为主；可选 `cael verify` 做更强验证
- 证据传播 / 精炼类型：
  - 谓词语言限制为可判定片段（数值线性约束为主）
  - 超出片段请用 `contract + seal` 或显式 `assume`（不安全）
- 意图合成 `synth`：
  - v0.1 为“有限候选空间搜索”（可配置 DSL），失败会给诊断并要求用户补充例子或改写
- 效果结晶 `Crystal`：
  - 支持 `inspect/crystalMap/simulate/melt`
  - **仅对 Serializable 子集**支持 `serialize/remoteExecute`（不允许捕获不可序列化值/函数值）

---

## 3. 完整语法（v0.1）

本节给出 **可实现的完整表面语法**：词法、布局规则、运算符优先级、声明与表达式、类型语法、模式语法、效果/结晶/阶段/量纲/流量/契约/错误层叠/上下文/并发/测试/合成。

> 约定：Cael 使用**缩进布局（layout）**，语法上不使用 `{}`/`;`。编译器在词法阶段把缩进转换为 `INDENT/DEDENT/NEWLINE`。

---

### 3.1 词法（Lexical）

#### 注释
- 行注释：`-- ...`
- 块注释：`{- ... -}`（可嵌套：v0.1 可选；建议实现）

#### 标识符
- 变量/函数名（小写开头）：`lowerIdent`，可包含字母数字下划线与 `'`
- 类型构造器/数据构造器（大写开头）：`UpperIdent`
- 运算符：由符号字符构成，如 `+ - * / == /= <= >= <> |> |>> |?> |!> . :: -> =>`

#### 字面量
- Int：`123`
- Float：`3.14`
- String：`"text"`（支持常见转义）
- Char（可选）：`'a'`
- Bool：`true` / `false`
- List：`[1,2,3]`
- Tuple：`(a,b)`、`(a,b,c)`（v0.1 建议支持到一定长度或用嵌套）

---

### 3.2 运算符与结合性（Operator Precedence）

> v0.1 建议的解析优先级（从高到低）：

1. 原子：字面量、变量、括号、记录/构造器字面量
2. 函数应用：`f x y`（左结合）
3. 前缀：`~f`（逆调用前缀）、一元 `-`（负号）
4. 字段访问：`x.field`（若采用点访问；也可去糖为 `getField`）
5. 乘除：`* /`
6. 加减：`+ -`
7. 比较：`< <= > >= == /=`
8. 逻辑：`&& ||`
9. 连接：`<>`
10. 组合：`.`（右结合）
11. 管道：`|>`（左结合）
12. 条件管道：`|?>` / `|!>`（左结合，语法层面是链式结构）
13. 分支管道：`|>>`（左结合）
14. 类型标注：`::`

---

### 3.3 模块与导入

```ebnf
Module      ::= "module" ModName "where" NEWLINE INDENT { TopDecl } DEDENT
ModName     ::= UpperIdent {"." UpperIdent}

ImportDecl  ::= "import" ModName [ImportSpec]
ImportSpec  ::= "(" ImportItem {"," ImportItem} ")"
ImportItem  ::= Ident | Ident "as" Ident
```

示例：

```cael
module Math.Geometry where

import Std.Collections (List, Map)
import Std.IO (Console)
```

---

### 3.4 顶层声明（Top-level Declarations）

```ebnf
TopDecl ::=
    ValueDecl
  | TypeSigDecl
  | DataDecl
  | TemporalDataDecl
  | TypeAliasDecl
  | ClassDecl
  | InstanceDecl
  | EffectDecl
  | FlowDecl
  | FlowRuleDecl
  | ContractDecl
  | ErrorHierarchyDecl
  | ContextDecl
  | SignatureDecl
  | SynthDecl
  | TestDecl
  | PropertyDecl
  | ComptimeDecl
  | ExportModifierDecl
```

> 说明：`pub` 是修饰符，作用于紧随其后的一个顶层声明。

#### 3.4.1 值声明与类型签名

```ebnf
TypeSigDecl ::= Ident "::" Type
ValueDecl   ::= ["pub"] FunDecl | ["pub"] LetDecl

FunDecl     ::= Ident {Pattern} "=" Expr
             | Ident NEWLINE INDENT {FunClause} DEDENT

FunClause   ::= Ident {Pattern} [Guards] "=" Expr
Guards      ::= NEWLINE INDENT {"|" Expr "=" Expr NEWLINE} DEDENT

LetDecl     ::= "let" ["mut"] Pattern ["::" Type] "=" Expr
```

**重要（v0.1 约束）**  
- `mut` 仅允许出现在 `do` 块内部或专用的 `state` 块（若实现）；顶层 `mut` 禁止。
- `mut` 变量赋值使用 `:=`（避免与 `do` 绑定 `<-` 冲突）。

示例：

```cael
add :: Int -> Int -> Int
add x y = x + y

let name :: String = "Cael"
```

#### 3.4.2 data / record / variant

```ebnf
DataDecl ::= ["pub"] "data" TypeCon {TypeVar} "=" DataRhs [Deriving]
DataRhs  ::= Ctor {"|" Ctor}
Ctor     ::= UpperIdent [CtorArgs]
CtorArgs ::= "{" FieldDecl {"," FieldDecl} "}"
          | TypeAtom {TypeAtom}

FieldDecl ::= Ident "::" Type
Deriving  ::= "deriving" "(" ClassName {"," ClassName} ")"
```

#### 3.4.3 Temporal Stage（阶段型）数据声明

```ebnf
TemporalDataDecl ::=
  ["pub"] "temporal" "data" TypeCon "where" NEWLINE INDENT {StageCtor} DEDENT

StageCtor ::=
  "@" UpperIdent "->" UpperIdent [ "{" FieldDecl {"," FieldDecl} "}" ]
```

语义：每个 `@Stage` 对应一个“阶段索引”。编译器生成阶段区分的数据表示（推荐 GADT）。

#### 3.4.4 type 别名（可选）

```ebnf
TypeAliasDecl ::= ["pub"] "type" TypeCon {TypeVar} "=" Type
```

#### 3.4.5 class / instance

```ebnf
ClassDecl ::=
  ["pub"] "class" [ClassConstraints "=>"] ClassName TypeVar "where"
  NEWLINE INDENT {MethodSig} DEDENT

MethodSig ::= Ident "::" Type

InstanceDecl ::=
  "instance" [ClassConstraints "=>"] ClassName TypeAtom "where"
  NEWLINE INDENT {ValueDecl} DEDENT
```

#### 3.4.6 effect / handler / handle（效果系统）

```ebnf
EffectDecl ::=
  ["pub"] "effect" EffectName "where" NEWLINE INDENT {EffectOpSig} DEDENT

EffectOpSig ::= Ident "::" Type

-- handler 是表达式层构造（见 3.5），但允许顶层 let 绑定它
```

效果约束语法（类型层面）：

```ebnf
EffectConstraint ::= EffectName "∈" TypeVar
```

#### 3.4.7 flow 与规则

```ebnf
FlowDecl     ::= ["pub"] "flow" UpperIdent
FlowRuleDecl ::= "flow" "rule:" FlowSpec "->" FlowSpec "requires" FlowPath
FlowSpec     ::= UpperIdent | "@" UpperIdent
FlowPath     ::= Ident { ">>" Ident }
```

> v0.1 仅检查显式值流：函数参数/返回、赋值/绑定、传入 sink（如 Print/Serialize）。

#### 3.4.8 contract 与 seal

```ebnf
ContractDecl ::=
  ["pub"] "contract" UpperIdent "=" Type "where"
  NEWLINE INDENT {PredicateLine} DEDENT

PredicateLine ::= Expr
```

`seal` 是表达式（见 3.5）。

#### 3.4.9 error hierarchy 与 `?`

```ebnf
ErrorHierarchyDecl ::=
  ["pub"] "error" "hierarchy" UpperIdent NEWLINE INDENT ErrorAlt { "|" ErrorAlt } DEDENT

ErrorAlt ::= UpperIdent [ "{" ErrorCtor { "|" ErrorCtor } "}" ]
ErrorCtor ::= UpperIdent [ "{" FieldDecl {"," FieldDecl} "}" ]
```

`?` 是表达式后缀操作符（见 3.5.14）。

#### 3.4.10 context（上下文透镜）

```ebnf
ContextDecl ::=
  ["pub"] "context" UpperIdent "where"
  NEWLINE INDENT {FieldDecl} DEDENT
```

#### 3.4.11 signature（接口签名）

```ebnf
SignatureDecl ::=
  ["pub"] "signature" UpperIdent TypeVar "where"
  NEWLINE INDENT {MethodSig} DEDENT
```

v0.1 语义：可编译为 typeclass 或 record-of-functions（由编译器选项决定）。

#### 3.4.12 synth（有限候选空间合成）

```ebnf
SynthDecl ::=
  ["pub"] "synth" Ident "::" Type NEWLINE
  INDENT
    "examples" NEWLINE INDENT {ExampleLine} DEDENT
    "constraints" NEWLINE INDENT {ConstraintLine} DEDENT
  DEDENT

ExampleLine    ::= Expr "~>" Expr
ConstraintLine ::= Expr
```

v0.1 约束：constraints 必须是可执行属性（可转 QuickCheck/SmallCheck）或 `pure`。

#### 3.4.13 test / property

```ebnf
TestDecl     ::= "test" StringLiteral "=" Expr
PropertyDecl ::= "property" StringLiteral "=" Expr
```

#### 3.4.14 comptime（可选，编译期常量求值）

```ebnf
ComptimeDecl ::= "comptime" Ident "=" Expr
```

v0.1：仅允许在可终止的纯子集内求值（无效果、无无限递归）。

---

### 3.5 表达式（Expressions）

```ebnf
Expr ::=
    Atom
  | Lambda
  | App
  | LetExpr
  | MatchExpr
  | IfExpr
  | CheckExpr
  | DoExpr
  | HandleExpr
  | HandlerExpr
  | CrystallizeExpr
  | MeltExpr
  | ParallelExpr
  | SelectExpr
  | TransitionExpr
  | PipeExpr
  | ComposeExpr
  | InverseCall
  | TryOp
  | AnnotExpr
```

#### 3.5.1 原子（Atom）

```ebnf
Atom ::=
    Ident
  | Literal
  | "(" Expr ")"
  | TupleLit
  | ListLit
  | RecordLit
  | CtorLit
```

#### 3.5.2 Lambda

```ebnf
Lambda ::= "\" Pattern "->" Expr
```

#### 3.5.3 函数应用

```ebnf
App ::= Expr Atom   -- 左结合
```

#### 3.5.4 局部 let（表达式内）

```ebnf
LetExpr ::=
  "let" Pattern ["::" Type] "=" Expr "in" Expr
```

> 顶层 `let` 是声明；局部 let 使用 `let ... in ...`，避免缩进歧义。

#### 3.5.5 if

```ebnf
IfExpr ::= "if" Expr "then" Expr "else" Expr
```

#### 3.5.6 match（模式匹配）

```ebnf
MatchExpr ::=
  "match" Expr NEWLINE INDENT {MatchAlt} DEDENT

MatchAlt ::=
  Pattern [GuardInline] "->" Expr

GuardInline ::= "|" Expr
```

#### 3.5.7 check（证据传播/精炼）

```ebnf
CheckExpr ::=
  "check" Expr "then" Expr "else" Expr
```

v0.1：`check` 条件若属于可判定精炼片段，则在 then 分支引入证据。

#### 3.5.8 do（效果/monadic）

```ebnf
DoExpr ::=
  "do" NEWLINE INDENT {Stmt} DEDENT

Stmt ::=
    Pattern "<-" Expr               -- bind
  | "let" Pattern ["::" Type] "=" Expr
  | "let" "mut" Ident ["::" Type] "=" Expr
  | Ident ":=" Expr                 -- mut assignment
  | Expr                            -- last expression allowed
```

#### 3.5.9 handler / handle

```ebnf
HandlerExpr ::=
  "handler" NEWLINE INDENT {HandlerClause} DEDENT

HandlerClause ::=
  Ident Pattern "->" Expr           -- operation clause
  | "return" Pattern "->" Expr      -- optional return clause (v0.1 可选)

HandleExpr ::=
  "handle" Expr Expr                -- handle handler program
```

> 运行时库需要 `resume` 语义；表面语法里可以直接调用 `resume`（由编译器注入到 handler 子句作用域）。

#### 3.5.10 crystallize / melt（效果结晶）

```ebnf
CrystallizeExpr ::= "crystallize" DoExpr
MeltExpr        ::= "melt" Expr
```

额外库函数（非语法）：`inspect`, `crystalMap`, `simulate`, `serialize`（Serializable 子集）。

#### 3.5.11 transition（阶段资源管理）

```ebnf
TransitionExpr ::= "transition" "@" UpperIdent DoExpr
```

#### 3.5.12 parallel / spawn / channel / select

```ebnf
ParallelExpr ::= "parallel" TupleLit

SelectExpr ::=
  "select" NEWLINE INDENT {SelectAlt} DEDENT

SelectAlt ::=
    "value" "from" Expr "->" Expr
  | "after" "(" Expr ")" "->" Expr
```

`spawn`, `newChannel`, `send`, `close`, `forEach` 等作为标准库函数提供。

#### 3.5.13 管道与组合

```ebnf
PipeExpr    ::= Expr "|>" Expr
ComposeExpr ::= Expr "." Expr       -- 右结合
```

分支管道与条件管道（语法级链）：

```ebnf
BranchPipeExpr ::= Expr "|>>" "(" Expr "," Expr ")"

CondPipeExpr ::=
  Expr NEWLINE INDENT {CondClause} DEDENT

CondClause ::=
    "|?>" Expr "=>" Expr
  | "|!>" "_" "=>" Expr
```

> v0.1：条件管道会去糖成 if/else 链；分支管道去糖成元组应用。

#### 3.5.14 逆调用 `~f`

```ebnf
InverseCall ::= "~" Expr Atom       -- 语义：对 inv 函数取逆后调用
```

#### 3.5.15 `?`（错误层叠）

```ebnf
TryOp ::= Expr "?"
```

语义：若 `Expr : Result a e1`，在期望 `Result a e2` 上下文中自动提升 `e1 -> e2` 并短路返回。

#### 3.5.16 类型标注（表达式）

```ebnf
AnnotExpr ::= Expr "::" Type
```

---

### 3.6 模式（Patterns）

```ebnf
Pattern ::=
    "_"                       -- wildcard
  | Ident                     -- variable
  | Literal
  | "(" Pattern ")"
  | TuplePat
  | ListPat
  | ConsPat
  | RecordPat
  | CtorPat

TuplePat  ::= "(" Pattern {"," Pattern} ")"
ListPat   ::= "[" [Pattern {"," Pattern}] "]"
ConsPat   ::= "(" Pattern ":" Pattern ")"        -- 或 Pattern ":" Pattern（需括号避免歧义）
RecordPat ::= "{" Ident {"," Ident} "}"          -- 字段名绑定到同名变量（v0.1）
CtorPat   ::= UpperIdent {Pattern}
           | UpperIdent "{" FieldBind {"," FieldBind} "}"
FieldBind ::= Ident "=" Pattern | Ident          -- v0.1 可选支持 `x = pat`
```

---

### 3.7 类型语法（Types）

```ebnf
Type ::=
    ForallType
  | FuncType

ForallType ::= "forall" TypeVar "." Type         -- v0.1 可选（可由推断替代）

FuncType ::= TypeTerm {"->" TypeTerm}

TypeTerm ::=
    TypeAtom
  | "(" Type ")"

TypeAtom ::=
    TypeCon
  | TypeVar
  | TypeAtom TypeAtom             -- 类型应用（左结合）
  | TypeAtom DimAnnot
  | TypeAtom FlowOrStageAnnot
  | "(" Type "," Type {"," Type} ")"
  | "[" Type "]"

DimAnnot        ::= "<" DimExpr ">"
FlowOrStageAnnot::= "@" UpperIdent

DimExpr ::=
    UnitName
  | DimExpr "*" DimExpr
  | DimExpr "/" DimExpr
  | DimExpr "^" IntLiteral
  | "(" DimExpr ")"

-- 约束（用于类型签名前置）
ClassConstraints ::=
  Constraint {"," Constraint}

Constraint ::=
    ClassName TypeAtom
  | EffectName "∈" TypeVar
  | "Has" TypeAtom TypeVar        -- (Has Database ctx)
```

**类型注解的推荐顺序（v0.1 约定）**  
- `Float <km/hr> @Sensitive`：先量纲 `<...>`，再 flow/stage `@...`。

---

## 4. 语义要点（v0.1 限制说明）

### 4.1 `mut` 的限制
- `mut` 只在 `do`（或未来的 `state`）块内允许。
- 赋值用 `:=`。
- 编译器禁止将可变引用泄漏出作用域（保持纯语义）。

### 4.2 `inv`（可逆函数）的限制：InvCore
`inv f` 若不写 `inverse`，则函数体必须处于 InvCore 子集：
- 允许：构造/解构、字段重排、可逆算术（如 `x * c` 需 `c ≠ 0`）、可逆编码组合子、可逆组合与管道
- 不允许：信息丢失分支、一般递归、hash/排序等不可逆操作
- 超出 InvCore：必须显式写 `inverse`

### 4.3 精炼/证据传播
- `check` 只对可判定片段做自动蕴含与传播（主要是数值线性约束）
- 复杂谓词请用 `contract` + `seal`

### 4.4 Crystal 的可序列化子集
- `serialize/remoteExecute` 仅对 “无函数值/无不可序列化捕获”的 Crystal 生效
- 否则仅支持本地 `inspect/simulate/melt`

---

## 5. 标准库（概览）

运行时库（`cael-prelude`）至少提供：
- `Result a e = Ok a | Err e`
- `Eff eff a`、`Handler`、`handle`、`send`、`resume`
- `Crystal eff a` 与 `inspect/crystalMap/simulate/melt`
- `Temporal a`（历史版本）
- `Quantity dim unit a`（量纲单位）
- `Ctx ctx a` 与 `runCtx`、`ask`
- 并发：`parallel/spawn/Channel/select`
- 常用容器与函数：`List/Map/Set`、`foldl/map/filter` 等

---

## 6. CLI 工具

```text
cael build <Module>         编译到 Haskell 并用 GHC 构建
cael run   <Module>         编译并运行
cael test  <Module>         运行测试/属性测试
cael check <Module>         仅做静态检查（类型/效果/流量/量纲/阶段/精炼片段）
cael verify <Module>        额外验证（inv 结构性验证、可选符号验证、synth 约束回归）
cael fmt   <Path>           格式化（可选）
cael doc   <Module>         生成文档（可选）
cael crystal inspect <...>  结晶体调试（可选）
```

---

## 7. 示例（精选）

### 7.1 管道

```cael
let result =
  [1,2,3,4,5]
  |> filter (> 2)
  |> map (* 10)
  |> foldl (+) 0
```

### 7.2 效果 + handler

```cael
effect Console where
  print    :: String -> ()
  readLine :: () -> String

mockConsole :: Handler Console a
mockConsole = handler
  print msg  -> resume ()
  readLine _ -> resume "TestUser"

greet :: (Console ∈ eff) => Eff eff ()
greet = do
  print "Name?"
  name <- readLine ()
  print ("Hello " <> name)

testGreet :: ()
testGreet = handle mockConsole greet
```

### 7.3 契约类型

```cael
contract Email = String where
  contains "@" self
  length self > 3
  length self < 255

createEmail :: String -> Result Email ValidationError
createEmail raw = seal Email raw
```

---

## 8. 版本与兼容性
- v0.1 是可实现子集规范；后续版本可能扩展精炼证明、inv 验证强度、synth 搜索空间与交互能力、Crystal 的分布式执行等。

---

## 9. 贡献
欢迎提交 issue/PR：
- 语法一致性与去糖规则
- 运行时库接口设计
- 编译器诊断与错误信息质量
- 示例与标准库补全
