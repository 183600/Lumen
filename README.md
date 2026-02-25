# **Lumen** 语言设计规范

> *Lumen* — 让计算透明、数据有历史、变换可逆转的函数式语言。

---

## 0 设计哲学

```
每个值知道自己从哪来。
每个变换说得清自己在做什么。
每个类型能随时间生长。
```

三条原则：**溯源透明**、**渐进精确**、**变换可逆**。

---

## 1 基础语法

```lumen
-- 类型定义
type Option a = None | Some a

type User = {
  name  : String,
  age   : Int,
  email : String,
}

-- 函数定义
let greet : User -> String
let greet user = "Hello, " ++ user.name

-- 模式匹配
let describe : Option Int -> String
let describe =
  | None   -> "nothing"
  | Some n -> "got " ++ show n

-- 管道
let result = data |> parse |> validate |> transform

-- Lambda
let add = \x y -> x + y

-- 列表推导
let evens = [x | x <- 1..100, x % 2 == 0]
```

类型系统：Hindley-Milner 推断 + 行多态 + 代数效果。

---

## 2 `why` — 计算解释链

> 任何表达式都能被追问"为什么是这个值"。

```lumen
let score = why (calculateScore user)
-- score : Explained Float
-- score.value == 87.5
-- score.explanation ==
--   CalculateScore
--   ├─ baseScore = 60.0       ← user.level = Gold
--   ├─ activityBonus = 20.0   ← loginDays(45) * 0.44
--   ├─ referralBonus = 7.5    ← referralCount(3) * 2.5
--   └─ result = 87.5

-- 函数内部用 explain 标注人类可读的说明
let calculateScore : User -> Float
let calculateScore user =
  let baseScore = explain "基础分 (等级: {user.level})" <|
    case user.level of
      Gold   -> 60.0
      Silver -> 40.0
      Bronze -> 20.0
  let activityBonus = explain "活跃奖励" <|
    float user.loginDays * 0.44
  baseScore + activityBonus

-- 没有 explain 标注的函数也能工作，编译器自动生成基于代码结构的解释
-- why 是惰性的：只访问 .value 时不构建解释树
```

### 类型

```lumen
type Explained a = {
  value       : a,
  explanation : Trace,
}

type Trace =
  | Literal  String                    -- 字面值
  | Binding  String Trace              -- let 绑定
  | Apply    String (List Trace) Trace -- 函数调用
  | Branch   String Trace Trace        -- 条件分支
```

### 实现

编译器为 `why` 包裹的表达式生成追踪版本。不使用 `why` 时零开销。访问 `.explanation` 时才实际构建追踪树。

---

## 3 `origin` — 值溯源

> 追踪数据从哪来、经过了什么变换。

```lumen
let result = origin {
  let raw = readCSV "data.csv"
  let filtered = filter (age > 18) raw
  let scores = map calculateScore filtered
  let avg = average scores
  avg
}
-- result : Traced Float
-- result.value == 72.5
-- result.origin ==
--   Average
--   └─ Map(calculateScore)
--      └─ Filter(age > 18)  -- 100 → 67 条
--         └─ ReadCSV("data.csv")  -- 100 条

-- 溯源查询
result |> Origin.sources        -- ["data.csv"]
result |> Origin.transforms     -- [ReadCSV, Filter, Map, Average]
result |> Origin.dependsOn "data.csv"  -- true
```

`why` 解释**计算逻辑**（为什么得到这个值），`origin` 追踪**数据血缘**（数据从哪来）。两者正交，可组合：

```lumen
let r = origin { why (pipeline data) }
-- r.explanation : Trace    -- 逻辑解释
-- r.origin      : Origin   -- 数据血缘
```

### 实现

`origin` 块内，编译器为中间值附加惰性 `Origin` 标签（引用计数 DAG）。块外部零开销。

---

## 4 `assume` / `guarantee` — 假设与保证

> 让前提条件成为代码的显式部分，编译器尝试静态验证。

```lumen
let safeDivide : Int -> Int -> Int
let safeDivide a b =
  assume { b != 0 }
  a / b

-- 调用处
safeDivide 10 5     -- ✓ 编译器证明 5 != 0
safeDivide 10 x     -- ⚠ 无法证明，插入运行时检查
safeDivide 10 0     -- ✗ 编译错误

-- 多条假设
let processAge : Int -> String
let processAge age =
  assume {
    age >= 0,
    age <= 150,
  }
  if age < 18 then "未成年" else "成年"

-- guarantee 声明函数输出满足的条件
let clamp : Int -> Int
let clamp x =
  guarantee { result >= 0, result <= 100 }
  if x < 0 then 0
  else if x > 100 then 100
  else x

-- 假设沿调用链传播
let foo : Int -> Int
let foo x =
  assume { x > 0 }
  bar (x + 1)    -- 编译器推导 x+1 > 1 > 0

let bar : Int -> Int
let bar y =
  assume { y > 0 }   -- foo 的调用自动满足
  y * 2
```

### 编译器行为

```
能静态证明满足    → 零运行时开销
不能证明也不能反证 → 插入运行时检查 + 编译警告
能静态证明违反    → 编译错误
```

内置轻量级 SMT 求解器处理线性算术和简单逻辑。

---

## 5 `reversible` / `invert` — 自动函数反转

> 可逆变换自动获得镜像函数。

```lumen
reversible let celsiusToFahr : Float -> Float
reversible let celsiusToFahr c = c * 9.0 / 5.0 + 32.0

let fahrToCelsius = invert celsiusToFahr
-- fahrToCelsius f = (f - 32.0) * 5.0 / 9.0

-- 构造器天然可逆
type Celsius = Celsius Float
reversible let wrap : Float -> Celsius
reversible let wrap x = Celsius x
invert wrap  -- Celsius -> Float

-- 组合可逆函数
reversible let pipeline = f >> g >> h
invert pipeline  -- invert h >> invert g >> invert f

-- 不可逆函数用 lossy 标记
lossy let roundToInt : Float -> Int
lossy let roundToInt x = round x
-- invert roundToInt  -- 编译错误

-- 记录变换
type UserDTO   = { userName : String, userAge : Int }
type UserModel = { name : String, age : Int }

reversible let toModel : UserDTO -> UserModel
reversible let toModel dto = {
  name = dto.userName,
  age  = dto.userAge,
}

let toDTO = invert toModel
```

### 实现

编译器将函数体转为符号等式，对输入变量求解。求解成功则生成反函数代码；失败则报编译错误并指出不可逆的操作。

---

## 6 `draft` / `refine` / `final` — 渐进类型

> 类型可以从草稿逐步完善，不必一开始就完整。

```lumen
-- 草稿类型：省略号表示还有未定义的字段
draft type Config = {
  host : String,
  port : Int,
  ...
}

-- 已知字段正常使用
let url config = config.host ++ ":" ++ show config.port  -- ✓

-- 访问未定义字段：编译警告
let x = config.timeout  -- ⚠ timeout 未在 Config 中定义 (draft)

-- 逐步补充
refine type Config = {
  host    : String,
  port    : Int,
  timeout : Duration,
  ...
}

-- 定稿
final type Config = {
  host    : String,
  port    : Int,
  timeout : Duration,
  tls     : Bool,
}
-- 此后访问不存在的字段为编译错误
```

### 草稿函数

```lumen
draft let processOrder : Order -> Result
draft let processOrder order =
  let validated = validate order
  todo "应用折扣"
  todo "生成发票"
  Ok validated

-- todo 在运行时抛 TodoError，编译器记录所有 todo
-- $ lumen build
--   ⚠ processOrder has 2 todos
--   ⚠ Config has unknown fields
```

### 草稿传播

```lumen
final let main : IO ()   -- ✗ 编译错误：依赖草稿函数
draft let main : IO ()   -- ✓
```

### 实现

`draft` 类型 = 开放行类型（带行变量 `| r`）。`refine` 只允许添加字段或收窄类型。`final` 关闭行变量。

---

## 7 `adapt` — 自动类型适配

> 结构相似的类型之间不必手写转换。

```lumen
type UserAPI = { user_name : String, user_age : Int, user_email : String }
type UserDB  = { name : String, age : Int, email : String }

let dbUser = adapt apiUser : UserAPI -> UserDB
-- 编译器推导：
--   user_name  → name   (去前缀 user_)
--   user_age   → age
--   user_email → email
```

### 匹配规则（按优先级）

1. 精确名称匹配
2. 大小写变体（camelCase ↔ snake_case）
3. 常见前缀/后缀
4. 类型兼容 + 位置
5. 嵌套递归匹配

### 手动消歧

```lumen
let converted = adapt source : TypeA -> TypeB with {
  fieldX = source.fieldY,
  fieldZ = defaultValue,
}
```

### 嵌套适配

```lumen
type OrderAPI = { order_id : String, customer : CustomerAPI, items : List ItemAPI }
type OrderDB  = { id : String, customer : CustomerDB, items : List ItemDB }

let dbOrder = adapt apiOrder : OrderAPI -> OrderDB
-- 自动递归适配嵌套类型和列表元素
```

无损适配自动标记为 `reversible`，可以 `invert`。

---

## 8 `patch` / `diff` / `apply` — 一等补丁

> 数据的差异是可以操作的值。

```lumen
type User = { name : String, age : Int, email : String }

-- 创建补丁
let birthday : Patch User = patch { age = age + 1 }
let rename : String -> Patch User = \n -> patch { name = n }

-- 应用补丁
let alice = { name = "Alice", age = 30, email = "a@b.com" }
let older = alice |> apply birthday
-- { name = "Alice", age = 31, email = "a@b.com" }

-- 补丁组合
let update = birthday <> rename "Alice Smith"

-- 补丁求逆
let undo = invert birthday  -- age = age - 1

-- 从两个值生成补丁
let p = diff alice older  -- patch { age = 31 }

-- 嵌套补丁
type Company = { name : String, ceo : User }
let promote = patch Company { ceo.age = ceo.age + 1 }

-- 条件补丁
let cap = patch User {
  age = if age < 18 then 18 else age
}
```

`Patch a` 内部是字段级修改指令列表。编译器自动为记录类型派生 `Patchable`。

---

## 9 `example` — 示例即文档即测试

> 示例是代码的一等组成部分。

```lumen
let formatName : String -> String -> String
let formatName first last = first ++ " " ++ last

example formatName {
  formatName "Alice" "Smith" == "Alice Smith"
  formatName "Bob" "Jones"   == "Bob Jones"
  formatName "" "X"          == " X"
}
```

### 四合一

1. **文档**：自动出现在生成的文档中
2. **测试**：`lumen test` 自动运行
3. **类型提示**：帮助推断/验证类型
4. **合成**：函数体缺失时，编译器尝试从示例合成实现

### 从示例合成

```lumen
let classify : Int -> String

example classify {
  classify 0  == "zero"
  classify 1  == "positive"
  classify -1 == "negative"
  classify 42 == "positive"
}

-- 编译器合成：
-- let classify n =
--   if n == 0 then "zero"
--   else if n > 0 then "positive"
--   else "negative"
--
-- ℹ Synthesized from 4 examples. Please verify.
```

### 属性

```lumen
example reverse {
  reverse [1,2,3] == [3,2,1]
  reverse [] == []

  forall xs : List Int =>
    reverse (reverse xs) == xs
}
```

---

## 10 `source` — 来源类型

> 值在类型层面携带来源标记，编译器区分不同来源的数据。

```lumen
source FileSystem, Network, UserInput, Computed, Validated

-- 函数声明接受哪些来源
let saveToDB : String @Validated -> IO ()

-- 验证改变来源标记
let validate : String @UserInput -> Option (String @Validated)
let validate raw =
  if isSafe raw then Some (raw as @Validated) else None

-- 来源自动传播
let readConfig : IO (Config @FileSystem)
let readConfig = parse (readFile "config.toml")

-- 编译错误
let bad input = saveToDB input  -- ✗ 期望 @Validated，得到 @UserInput
```

与 `assume` 互补：`source` 追踪数据**从哪来**，`assume` 约束数据**满足什么条件**。

---

## 11 `pathway` — 类型路径约束

> 值从类型 A 到类型 B 必须经过指定的步骤。

```lumen
pathway RawInput -> SafeOutput requires
  step sanitize  : RawInput -> SanitizedInput
  step validate  : SanitizedInput -> ValidatedInput
  step transform : ValidatedInput -> SafeOutput

-- 合法
let process input =
  input |> sanitize |> validate |> transform  -- ✓

-- 非法：跳过 sanitize
let shortcut input =
  input |> validate  -- ✗ 编译错误：RawInput 不能直接到 validate

-- 路径可以有分支
pathway Payment -> Receipt requires
  step authorize : Payment -> AuthorizedPayment
  branch
    | step chargeCard : AuthorizedPayment -> Receipt
    | step chargeBank : AuthorizedPayment -> Receipt
```

---

## 12 `metamorphic` — 类型蜕变

> 值在生命周期中改变类型，每个阶段暴露不同的操作。

```lumen
metamorphic type Connection
  stage Disconnected where
    let connect : Self -> Address -> Connecting

  stage Connecting where
    let awaitHandshake : Self -> Connected | Failed
    let cancel : Self -> Disconnected

  stage Connected where
    let send : Self -> Bytes -> Connected
    let receive : Self -> (Bytes, Connected)
    let disconnect : Self -> Disconnected

  stage Failed where
    let retry : Self -> Connecting
    let abort : Self -> Disconnected

-- 使用
let communicate addr =
  let conn = Connection.new              -- : Connection @Disconnected
  let conn = conn.connect addr           -- : Connection @Connecting
  case conn.awaitHandshake of
    | Connected conn ->
        let (data, conn) = conn.receive
        conn.disconnect
        Ok data
    | Failed conn ->
        conn.abort
        Err ConnectionFailed

-- 编译错误
let bad (conn : Connection @Disconnected) =
  conn.send data  -- ✗ send 只在 Connected 阶段可用
```

编译器提供状态可达性分析。

---

## 13 `evolving` — 类型版本演化

> 类型可以声明版本演化，编译器自动生成迁移链。

```lumen
evolving type UserProfile
  v1 = { name : String, email : String }

  v2 = { name : String, email : String, avatar : Url }
    migrate from v1 = \old -> { ...old, avatar = defaultAvatar }

  v3 = { firstName : String, lastName : String, email : String, avatar : Url }
    migrate from v2 = \old -> {
      firstName = old.name |> split " " |> head,
      lastName  = old.name |> split " " |> last,
      email     = old.email,
      avatar    = old.avatar,
    }

-- 自动链式迁移
let current : UserProfile @v3 = oldData.migrateTo @v3

-- 序列化自带版本号
let json = serialize profile  -- { "_version": 3, ... }
let loaded = deserialize json  -- 自动迁移到最新版本
```

---

## 14 `layered` — 分层不可变数据

> 每次修改产生新层，旧层自动保留，支持回溯、分叉、合并。

```lumen
layered type Document = {
  title   : String,
  content : String,
}

let doc0 = Document { title = "Hello", content = "" }
let doc1 = doc0 with { content = "First draft" }
let doc2 = doc1 with { content = "Second draft" }

-- 回溯
doc2.atLayer 0 |> .content  -- ""

-- 查看差异
doc2.diff doc0  -- [Layer 1: content changed, Layer 2: content changed]

-- 分叉
let branchA = doc1 with { title = "A" }
let branchB = doc1 with { title = "B" }

-- 合并
let merged = merge branchA branchB with
  | conflict field a b -> a
```

底层使用持久化数据结构（HAMT）+ 层元数据。

---

## 15 `progressive` — 渐进式计算

> 函数声明多个精度层级，消费者决定等待的精度。

```lumen
progressive let renderScene : Scene -> Image =
  level 1 : wireframe scene
  level 2 : flatShading scene
  level 3 : rayCasting scene
  level 4 : pathTracing scene 1000

-- 选择精度
let preview = renderScene scene |> atLevel 1
let final   = renderScene scene |> atLevel 4

-- 逐步获取
for image in renderScene scene |> progressively do
  display image
  if userSatisfied then break

-- 限时
let best = renderScene scene |> bestWithin 100.ms
```

---

## 16 `recipe` — 声明式配方

> 声明依赖而非顺序，编译器自动推导执行顺序和并行机会。

```lumen
recipe let buildReport : UserId -> IO Report
recipe let buildReport userId =
  let user     = fetchUser userId        -- 独立
  let orders   = fetchOrders userId      -- 独立
  let reviews  = fetchReviews userId     -- 独立
  let profile  = computeProfile user     -- 依赖 user
  let stats    = computeStats orders     -- 依赖 orders
  let summary  = summarize profile stats reviews
  Report { user, profile, stats, summary }

-- 编译器推导的执行图：
--   fetchUser ──→ computeProfile ──┐
--   fetchOrders ─→ computeStats ───┼→ summarize
--   fetchReviews ──────────────────┘
-- 前三个自动并行
```

---

## 17 `graft` — 函数嫁接

> 替换函数中的某个命名步骤，而不重写整个函数。

```lumen
let processImage : Image -> Image
let processImage img =
  img
  |> step normalize : adjustBrightness
  |> step denoise   : gaussianBlur 1.5
  |> step sharpen   : unsharpMask 0.8
  |> step resize    : bilinearResize 800 600

-- 只替换一个步骤
let v2 = processImage |> graft denoise with medianFilter 3

-- 替换多个步骤
let v3 = processImage
  |> graft denoise with bilateralFilter 9
  |> graft resize  with lanczosResize 1024 768

-- 类型安全：替换步骤的输入输出类型必须兼容
let bad = processImage
  |> graft denoise with toGrayscale  -- ✗ Image -> GrayImage 不兼容
```

---

## 18 `context` / `chameleon` — 上下文多态

> 同一函数在不同上下文中自动选择不同实现。

```lumen
context Testing where
  config = { verbose = true, mockExternal = true }

context Production where
  config = { verbose = false, mockExternal = false }

chameleon let log : String -> IO ()
chameleon let log =
  in Testing    -> \msg -> print ("[TEST] " ++ msg)
  in Production -> \msg -> asyncLogToFile msg

chameleon let fetchData : Id -> IO Data
chameleon let fetchData =
  in Testing    -> mockData
  in Production -> database.query

-- 使用
within Testing {
  process 42  -- 使用 mock 数据、控制台日志
}

-- 编译器检查所有上下文都覆盖了
chameleon let sendEmail =
  in Testing    -> recordEmail
  in Production -> smtp.send
  -- ⚠ 编译警告：Benchmarking 上下文未定义
```

---

## 19 `isolate` — 恐慌隔离域

> 任何表达式可以在隔离域中求值，异常不传播。

```lumen
let result = isolate {
  let x = mightPanic ()
  let y = alsoRisky x
  x + y
}  -- : Result Int Panic

-- 带超时
let result = isolate (timeout = 5.seconds) {
  potentiallyInfinite ()
}  -- : Result Value (Panic | Timeout)

-- 带资源限制
let result = isolate (memory = 100.mb) {
  memoryHungry ()
}  -- : Result Value (Panic | ResourceExhausted)
```

---

## 20 `laws` — 代数法则声明

> 为自定义类型声明代数法则，编译器据此自动简化。

```lumen
laws for MyMonoid a where
  law associative : combine (combine a b) c == combine a (combine b c)
  law identityLeft  : combine empty a == a
  law identityRight : combine a empty == a

-- 编译器自动简化
combine (combine (combine a empty) b) (combine empty c)
-- → combine (combine a b) c

-- 内置简化
list |> map f |> map g       -- → map (f >> g) list
list |> filter p |> filter q -- → filter (\x -> p x && q x) list
list |> reverse |> reverse   -- → list

-- 幂等性
laws for Cache where
  law idempotent : cache.put k v |> .put k v == cache.put k v

-- 编译器报告
-- $ lumen build --show-simplifications
-- src/main.lumen:15  map f >> map g → map (f >> g)  [map-fusion]
-- src/main.lumen:22  combine x empty → x            [identity-right]
```

---

## 21 `observe` — 非侵入式观察

> 观察中间值，不改变语义、不影响求值顺序。

```lumen
let result =
  data
  |> map transform
  |> observe (\x -> print ("中间: " ++ show x))
  |> filter predicate
  |> reduce combine

-- 编译器保证：移除所有 observe 后程序行为完全相同
-- observe 不阻止优化（如 map fusion）

-- 条件观察
result |> observeWhen isDebug (\x -> dump x)

-- 采样观察
largeStream |> observeSample 0.01 (\x -> log x)
```

---

## 22 `chord` — 和弦调用

> 多个异步事件全部到达时才触发联合回调。

```lumen
chord let completeCheckout
  (payment   : Event PaymentConfirmed)
  (inventory : Event InventoryReserved)
  (address   : Event AddressValidated)
  : CheckoutResult =
  finalizeOrder payment.data inventory.data address.data

-- 发射事件（顺序不重要）
emit PaymentConfirmed { amount = 99.99 }
emit AddressValidated { addr = "..." }
emit InventoryReserved { itemId = 42 }
-- → 三个都到达后自动触发 completeCheckout

-- 带超时
chord let syncData
  (a : Event DataA)
  (b : Event DataB)
  : SyncResult
  timeout 5.seconds -> SyncResult.partial arrivedEvents
```

---

## 23 `degrade` — 优雅降级

> 函数声明降级策略链，主实现失败时自动尝试降级方案。

```lumen
let fetchProfile : UserId -> IO UserProfile
let fetchProfile id =
  primary =
    db.query "SELECT * FROM users WHERE id = ?" id

  degrade to CachedProfile when DbError =
    cache.get ("user:" ++ show id) ?? UserProfile.skeleton id

  degrade to MinimalProfile when CacheError =
    UserProfile { id = id, name = "Unknown", avatar = defaultAvatar }

-- 降级层级可查询
let result = fetchProfile 42
result.degradationLevel  -- 0 = primary, 1 = first fallback, ...
```

---

## 24 `faceted` — 多面值

> 一个绑定持有同一数据的多种表示，按上下文自动选择。

```lumen
faceted type Color =
  facet RGB  { r : UInt8, g : UInt8, b : UInt8 }
  facet HSL  { h : Float, s : Float, l : Float }
  facet Hex  { value : String }
  facet CMYK { c : Float, m : Float, y : Float, k : Float }

  biject RGB <-> HSL  = (rgbToHsl, hslToRgb)
  biject RGB <-> Hex  = (rgbToHex, hexToRgb)
  biject RGB <-> CMYK = (rgbToCmyk, cmykToRgb)

let color = Color.rgb 255 128 0

let renderCSS : Color @Hex -> String
let renderCSS c = "color: #" ++ c.value

let mixPaint : Color @CMYK -> PaintMix

renderCSS color   -- 自动转为 Hex
mixPaint color    -- 自动转为 CMYK
-- 转换惰性缓存
```

---

## 25 类型系统总览

```lumen
-- 基础
type Bool = True | False
type Option a = None | Some a
type Result e a = Err e | Ok a

-- 行多态
type HasName r = { name : String | r }

-- 效果
let readFile : String -> IO String
let pureCalc : Int -> Int               -- 无效果 = 纯函数

-- 来源标记
type a @s  -- a 带来源 s

-- 特殊类型
type Explained a     -- why
type Traced a        -- origin
type Patch a         -- patch
type Inverse f       -- invert

-- 核心 trait
trait Adaptable a b where
  adapt : a -> b

trait Patchable a where
  apply : Patch a -> a -> a
  diff  : a -> a -> Patch a

trait Invertible f where
  invert : f -> InverseOf f
```

---

## 26 模块系统

```lumen
module MyApp.Users exposing (User, createUser, findUser)

import Data.List (map, filter)
import Network.HTTP (get, post)

-- 草稿模块
draft module MyApp.Billing exposing (..)

-- $ lumen status
-- ✓ MyApp.Users    (final)
-- ◐ MyApp.Billing  (draft: 3 todos, 1 draft type)
```

---

## 27 工具链

```bash
$ lumen build
  ✓ 12 modules compiled
  ⚠ 2 draft modules, 5 todos
  ✓ 47 assumptions statically verified
  ⚠ 3 assumptions need runtime checks
  ✓ 2 functions auto-inverted
  ✓ 4 type adaptations generated

$ lumen test
  ✓ 156 examples passed
  ✓ 23 properties passed (1000 samples each)

$ lumen why "MyApp.calculateScore testUser"
  calculateScore(testUser)
  ├─ baseScore = 60.0 (Gold)
  ├─ activityBonus = 20.0 (45 days × 0.44)
  └─ result = 80.0

$ lumen origin "MyApp.pipeline testData"
  pipeline result = 72.5
  └─ Average of 67 values
     └─ Filtered from 100 (age > 18)
        └─ Source: data.csv

$ lumen draft-report
  Types:   Config (2 unknown fields)
  Funcs:   processOrder (1 todo)
  Modules: 1 of 5 draft
```

---

## 28 完整示例

```lumen
module Shop.Orders

import Shop.Types (Order, Customer, Product, Money)
import Shop.DB (findCustomer, findProduct, saveOrder)

-- 渐进类型
draft type OrderRequest = {
  customerId : String,
  items      : List { productId : String, quantity : Int },
  ...
}

type OrderResponse = {
  orderId : String,
  total   : Money,
  status  : OrderStatus,
}

type OrderStatus = Pending | Confirmed | Shipped | Delivered

-- 可逆变换
reversible let requestToInternal : OrderRequest -> InternalOrder
reversible let requestToInternal req = {
  customer = CustomerId req.customerId,
  items = req.items |> map (\i -> {
    product  = ProductId i.productId,
    quantity = Quantity i.quantity,
  }),
}

let internalToRequest = invert requestToInternal

-- 业务逻辑
let processOrder : OrderRequest -> IO (Result OrderError OrderResponse)
let processOrder request = do
  assume {
    length request.items > 0,
    all (\i -> i.quantity > 0) request.items,
  }

  let customer <- findCustomer request.customerId
  assume { customer.isActive }

  let total = origin {
    request.items
    |> map (\item -> do
        let product <- findProduct item.productId
        let price = explain "单价 × 数量" <|
          product.price * fromInt item.quantity
        let discounted = explain "会员折扣 {customer.tier}" <|
          applyDiscount customer.tier price
        discounted)
    |> sum
  }

  let orderId <- saveOrder (requestToInternal request) total
  Ok { orderId = orderId, total = total.value, status = Pending }

-- 示例
example processOrder {
  let req = {
    customerId = "C001",
    items = [
      { productId = "P001", quantity = 2 },
      { productId = "P002", quantity = 1 },
    ],
  }

  with mockDB {
    customers = [{ id = "C001", isActive = true, tier = Gold }],
    products = [
      { id = "P001", price = Money 10.00 },
      { id = "P002", price = Money 25.00 },
    ],
  }

  let result = processOrder req
  result == Ok {
    orderId = _,
    total = Money 40.50,
    status = Pending,
  }
}

-- 补丁
let updateQuantity : String -> Int -> Patch OrderRequest
let updateQuantity productId newQty = patch {
  items = items |> map (\item ->
    if item.productId == productId
    then { ...item, quantity = newQty }
    else item)
}

let addItem : String -> Int -> Patch OrderRequest
let addItem productId qty = patch {
  items = items ++ [{ productId = productId, quantity = qty }]
}

let modification = updateQuantity "P001" 3 <> addItem "P003" 1
let modified = apply modification original

-- 适配外部系统
type PaymentAPI.Order = {
  order_id     : String,
  amount_cents : Int,
  currency     : String,
}

let toPaymentOrder : OrderResponse -> PaymentAPI.Order
let toPaymentOrder response = adapt response with {
  order_id     = response.orderId,
  amount_cents = toCents response.total,
  currency     = "CNY",
}
```

---

## 29 特性协同矩阵

|            | why | origin | assume | reversible | draft | adapt | patch | example | source | pathway | metamorphic | evolving | progressive | recipe | graft | context | isolate | laws | observe | chord | degrade | faceted |
|------------|:---:|:------:|:------:|:----------:|:-----:|:-----:|:-----:|:-------:|:------:|:-------:|:-----------:|:--------:|:-----------:|:------:|:-----:|:-------:|:-------:|:----:|:-------:|:-----:|:-------:|:-------:|
| **why**        | — | 互补 | 显示假设检查 | 解释反转推导 | 可用于草稿 | 解释适配 | 解释补丁 | 示例中可用 | 解释来源验证 | 解释路径 | 解释阶段转换 | 解释迁移 | 每级可解释 | 解释步骤 | 解释替换 | 按上下文解释 | 域内可用 | 解释简化 | 解释观察点 | 解释触发 | 解释降级 | 解释转面 |
| **reversible** | | | 反函数继承假设 | — | 草稿不可逆 | 适配可逆 | 补丁天然可逆 | 验证 f∘f⁻¹=id | | | | | | | | | | | | | | 面转换可逆 |
| **assume**     | | | — | | 草稿可带假设 | 适配满足假设 | 补丁满足假设 | 验证假设 | 来源满足假设 | 路径满足假设 | 阶段转换前验证 | | | | | | | | | | | |
| **draft**      | | | | | — | 草稿可适配 | 草稿可打补丁 | 帮助细化 | | | | 草稿可演化 | | | | | | | | | | |
| **example**    | | | | | | | | — | | | | | | | | | | | | | | |

---

## 30 实现路线图

```
Phase 1 — 基础：ML 核心 + example + draft + assume + isolate
Phase 2 — 透明：why + origin + observe
Phase 3 — 变换：reversible/invert + adapt + patch + faceted
Phase 4 — 生命周期：metamorphic + evolving + layered + source + pathway
Phase 5 — 并发与调度：recipe + chord + progressive + context + degrade + graft
Phase 6 — 优化：laws + 零开销抽象 + 增量编译 + LSP
```
