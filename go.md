# Go 编码规则（AI 编程约束）

适用语言版本：Go 1.21+

**两种用途，同一份规则**：① 编写/修改 Go 代码时遵守；② 审查人工编写的代码时作为判定依据，违规按编号引用（如 `违反 NAME-02`）。

冲突时优先级：**格式（gofmt 统一，不争论）> 命名（自描述）> 错误处理（显式）> 并发（谨慎，所有权明确）**。

全文示例中的 `logger` 泛指项目实际使用的结构化日志器（`log/slog` 或等价库），不特指某个日志库。

规则编号连续、不留空号；后续增删规则时整章重排，速查表与正文同步更新。

---

## 速查表

**读法**：编码前只需读本表 + 标 ★ 的条目正文（★ = 反直觉或本文件特有的收紧口径，不提前读就会写错）；其余规则写代码时自然遵守，命中疑问再查对应编号的正文。审查时读全文。
**本表是索引，不是规范** —— 摘要与正文冲突时，一律以正文为准。
**严重度**：`P0` 违反会致生产故障（数据丢失/泄漏/panic/静默走错分支/竞态/超时失效）· `P1` 应改 · `P2` 风格建议。处置流程见文末附录 APPLY-03。

| 编号 | 级别 | 摘要 |
|---|---|---|
| NAME-01 | `P2` | MixedCaps 驼峰，禁下划线 |
| NAME-02 | `P2` | 缩写词整体大写或整体小写（`userID` 非 `userId`） |
| NAME-03 | `P2` | 包名全小写无分隔符，1–2 词拼接 |
| NAME-04 | `P2` | 导出符号不重复包名（`user.Service`） |
| NAME-05 | `P2` | 变量名长度随作用域增长 |
| NAME-06 | `P2` | 接收器 1–2 字母，同类型必须一致，禁 `this` |
| NAME-07 | `P2` | 常量按角色命名，禁全大写下划线 |
| NAME-08 ★ | `P1` | iota 枚举从 `iota + 1` 开始 |
| NAME-09 | `P2` | 哨兵错误 `Err` 前缀，错误类型 `Error` 后缀 |
| NAME-10 ★ | `P1` | 标识符禁中文、拼音及拼音缩写 |
| FMT-01 | `P2` | import 分三组：标准库 → 第三方 → 内部 |
| DOC-01 | `P2` | 导出标识符必须有以标识符名开头的注释 |
| DOC-02 | `P1` | 注释写「为什么」，禁复述代码，禁失效注释 |
| DOC-03 ★ | `P2` | `TODO(责任人): 说明 #单号` 固定格式 |
| ERR-01 | `P0` | error 必须处理，`_` 丢弃需注释论证 |
| ERR-02 | `P1` | 错误先行提前 return，正常逻辑不进 `else` |
| ERR-03 | `P1` | 传递错误用 `%w` 包装并补上下文 |
| ERR-04 | `P1` | 禁「既记日志又返回错误」，日志只在顶层记一次 |
| ERR-05 | `P1` | 按调用方需求选哨兵错误 / 错误类型 / 包装 |
| ERR-06 | `P1` | 库代码禁 panic，仅三类场景合法 |
| ERR-07 ★ | `P1` | 错误消息小写开头、无句末标点、不用 "failed to" |
| ERR-08 | `P0` | 判定错误用 `errors.Is`/`As`，禁 `==` 与类型断言 |
| FUNC-01 ★ | `P1` | 裸返回仅限函数体 ≤4 行；不裸返回就不命名返回值 |
| FUNC-02 ★ | `P1` | 业务参数 >3 用 Options（ctx 与 variadic 不计入） |
| FUNC-03 ★ | `P0` | 资源释放/解锁紧跟 `defer`；写入型必须检查 `Close`/`Flush` 的 error |
| FUNC-04 | `P0` | 禁循环体内直接 `defer`，需匿名函数包裹 |
| FUNC-05 | `P0` | 同类型禁混用值接收器与指针接收器 |
| FUNC-06 ★ | `P1` | 构造函数 `New`/`NewXxx`，返回 `*Xxx` 且非接口，禁两段式初始化 |
| FUNC-07 ★ | `P1` | `init()` 只做不可失败的赋值与注册，禁 I/O、禁起 goroutine |
| PKG-01 ★ | `P1` | 一包一职责，禁 `util`/`common`/`helper` 容器包 |
| PKG-02 | `P2` | 不对外暴露放 `internal/`，可复用放 `pkg/` |
| TYPE-01 ★ | `P1` | 空 slice 用 `var`；JSON 契约要求 `[]` 时必须 `[]T{}` |
| TYPE-02 | `P2` | 已知长度时预分配 slice/map 容量 |
| TYPE-03 ★ | `P0` | 导出边界存入与返回 slice 都要 `copy` |
| TYPE-04 | `P0` | map 必须初始化后再写入（nil map 写入 panic） |
| TYPE-05 | `P2` | 结构体初始化必须指定字段名 |
| TYPE-06 | `P2` | 序列化结构体必须写 tag，敏感字段 `json:"-"` |
| TYPE-07 | `P1` | 时间用 `time.Time`/`time.Duration`，禁裸整数 |
| TYPE-08 ★ | `P1` | 外部系统不支持时间类型时单位写进字段名 |
| TYPE-09 | `P1` | 用 `any`，禁 `interface{}`；且不滥用 `any` |
| CONC-01 ★ | `P1` | channel 容量只用 0 或 1，其他必须注释论证两点 |
| CONC-02 | `P0` | 禁 fire-and-forget goroutine，须有终止与等待手段 |
| CONC-03 | `P1` | 禁嵌入 `sync.Mutex`，必须作私有字段 |
| CONC-04 | `P1` | 读多写少用 `RWMutex`；纯计数器用 `atomic` |
| CONC-05 | `P0` | 共享变量禁无保护并发读写（含 map） |
| CONC-06 | `P0` | 循环变量捕获须传参（Go ≥1.22 不报此项） |
| CONC-07 ★ | `P0` | 并发扇出必须汇总各分支 error |
| CTX-01 | `P1` | I/O 类函数 `ctx` 必须是首参且命名 `ctx` |
| CTX-02 | `P0` | ctx 必须逐层透传到底层 I/O，禁中途丢弃 |
| CTX-03 ★ | `P0` | 禁把 ctx 存进结构体（后台组件 `runCtx` 例外） |
| CTX-04 | `P0` | 禁 nil ctx，生产路径禁 `context.TODO()` |
| CTX-05 | `P0` | `WithTimeout`/`WithCancel` 必须 `defer cancel()` |
| CTX-06 | `P1` | `WithValue` 只传请求元数据，key 用私有类型 |
| IFACE-01 | `P1` | 接口定义在消费方（架构规范另有规定时从架构） |
| IFACE-02 ★ | `P1` | 接口小而单一，用组合扩展；>4 方法即审视拆分 |
| TEST-01 | `P2` | 表驱动 + `t.Run`，用例带 `name`，报错带输入与期望 |
| TEST-02 | `P1` | 测私有方法用同包，只测公共 API 用 `_test` 包 |
| TEST-03 | `P2` | 测试辅助函数首行 `t.Helper()` |
| TEST-04 | `P2` | 用 `t.Fatalf` 禁 `panic`，清理用 `t.Cleanup` |
| PERF-01 | `P2` | 避免不必要的 `string` ↔ `[]byte` 转换（仅热路径且有 profile 证据时报） |
| PERF-02 | `P2` | 单占位符数字转字符串用 `strconv` 不用 `fmt`（仅热路径且有 profile 证据时报） |
| LAYOUT-01 | `P1` | `main` 只启动，`os.Exit`/`log.Fatal` 只在 `main` |

元规则 APPLY-01（改动性质分档）与 APPLY-02（严重度定义）见下一章，两种用途都适用。

---

## 0. 规则的适用方式

**APPLY-01** 写新代码前先读同目录既有代码。周边风格与本规则冲突时，按**改动性质**分三档处理：

| 改动性质 | 做法 |
|---|---|
| 新建文件、新建包 | 直接遵循本规则全文，无豁免 |
| 重构 | 按本规则改，把范围内不合规处一并改正 |
| 局部修改（加方法、修 Bug、改逻辑） | **局部一致优先** —— 跟随周边既有风格，不在同一文件里制造两种风格 |

判定「是否重构」只看一点：**用户是否明确要求重构、改名或统一风格**。没有明确要求时一律按局部修改处理，不得自行认定「顺手重构」。局部修改中为保持一致而偏离本规则的地方，单独提出，不夹带进本次改动。
```go
// 场景：某类型既有方法的接收器统一写作 this（NAME-06 明令禁止 this）
// ✅ 局部修改：新方法继续用 this —— 守住 NAME-06「同一类型接收器名必须一致」这一半
// ❌ 局部修改：新方法改用 a —— 同一类型出现两种接收器名，制造了新的 NAME-06 违规
// ❌ 局部修改：顺手把全文件 this 批量改名，把无关 diff 混进本次改动
// ✅ 重构（用户要求「统一接收器命名」）：全文 this 一律改为 a，两项要求同时满足
```

**APPLY-02** 严重度分级：每条规则标注 `P0`/`P1`/`P2`，含义与审查处置：

| 级别 | 含义 | 审查处置 |
|---|---|---|
| `P0` | 违反会导致**生产环境的正确性或稳定性问题** —— 数据丢失、资源/goroutine 泄漏、panic、静默走错分支、竞态、超时失效 | 逐条报告，视为阻塞项 |
| `P1` | 不直接致故障，但显著影响正确性风险、可维护性或契约清晰度 | 逐条报告，建议本次改 |
| `P2` | 风格与局部优化，影响一致性但无功能后果 | **不逐条展开**，汇总为「规则编号 × 出现次数 + 位置列表」 |

`P0` 共 15 条：ERR-01、ERR-08、FUNC-03、FUNC-04、FUNC-05、CONC-02、CONC-05、CONC-06、CONC-07、CTX-02、CTX-03、CTX-04、CTX-05、TYPE-03、TYPE-04（其中 CONC-06 仅在 `go.mod` 声明 <1.22 时生效）。

审查如何组织输出、哪些规则交给 lint，见文末**附录：审查执行细则**（APPLY-03/04）。编码时可跳过该附录。

---

## 1. 命名（NAME）

**NAME-01** `P2` 一律 MixedCaps 驼峰，禁止下划线。导出大写开头，未导出小写开头。测试函数名（`TestXxx_Yyy`）除外。
```go
// ❌ user_service / MAX_RETRY_COUNT / get_user_by_id      ✅ UserService / MaxRetryCount / buildQuery
```

**NAME-02** `P2` 缩写词整体大写或整体小写，禁止混合。适用：URL URI HTTP HTTPS ID DB SQL API JSON XML RPC gRPC。
```go
// ❌ XmlParser / userId / parseUrl      ✅ XMLParser / xmlParser / userID / parseURL
```

**NAME-03** `P2` 包名全小写、无分隔符、简短（1–2 个词直接拼接），禁止下划线与驼峰。职责要求见 PKG-01。
```go
// ❌ package user_service / userService / httpUtil      ✅ package user / httputil / sqldb
```

**NAME-04** `P2` 导出符号不得重复包名。
```go
// ❌ user.UserService      ✅ user.Service
```

**NAME-05** `P2` 变量名长度随作用域增长：循环/if 块内单字母可接受，函数内用描述性单词，参数与包级用完整名称。
```go
// ✅ for i, v := range items{}  ·  retryDelay := 5*time.Second  ·  var defaultHTTPClient = &http.Client{...}
```

**NAME-06** `P2` 接收器名为 1–2 字母类型缩写，同一类型所有方法必须一致。禁止 `this`/`self`。
```go
// ❌ func (this *User) / 同类型混用 u 与 user      ✅ func (u *User) / func (c *Client)
```

**NAME-07** `P2` 常量按角色命名而非按值命名；禁止全大写下划线、禁止 `k` 前缀。
```go
// ❌ MAX_PACKET_SIZE / kDefaultTimeout      ✅ MaxPacketSize / DefaultTimeout
```

**NAME-08** `P1` iota 枚举从 `iota + 1` 开始，避免零值歧义（`North Direction = iota + 1`）。
【判定】仅当存在一个语义明确表示「未设置/空闲/无」的零值成员（如 `StatusIdle = 0`）时，允许从 0 开始；若零值成员是一个正常业务状态，视为违反 —— 未赋值的字段会被误读成该状态。

**NAME-09** `P2` 哨兵错误用 `Err`/`err` 前缀，自定义错误类型用 `Error` 后缀。
```go
// ✅ var ErrNotFound = errors.New("not found")  ·  var errInvalidToken = ...  ·  type NotFoundError struct{}
```

**NAME-10** `P1` 标识符只能用英文单词，禁止中文、拼音及拼音缩写。适用于包名、类型、函数、变量、常量、字段、struct tag、文件名。找不到合适英文词时用完整英文短语，不得退化为拼音。
```go
// ❌ type 用户服务 / YongHuService / GetYongHu / dingdanZhuangtai / ddzt / package yonghu
// ✅ UserService / GetUser / orderStatus / package user
// 中文只允许出现在：注释、日志文本、错误消息、测试用例 name
```

---

## 2. 格式（FMT）

> 缩进、对齐、空行、声明分组的括号位置以 gofmt 输出为准，不由本文件单列规则，也不自行调整空白。

**FMT-01** `P2` import 分三组、组间空行、组内字母序：标准库 → 第三方 → 项目内部包。

---

## 3. 注释（DOC）

**DOC-01** `P2` 所有导出标识符必须有文档注释：以标识符名开头，完整句子。包注释以 `Package <name>` 开头。
【判定】同一 const 组已有组注释、且常量为字面量映射时，单条豁免。
```go
// ✅ // GetByID 根据用户 ID 获取用户信息。用户不存在时返回 ErrNotFound。
```

**DOC-02** `P1` 注释必须说明「为什么这么做」（业务原因、算法选型、取舍依据），禁止复述代码字面含义。禁止留下失效注释 —— 改代码时必须同步更新或删除已不成立的注释。
【判定】把注释删掉，读代码是否仍能推知同样的信息。**能推知 → 复述，属违反**；不能推知（隐藏约束、外部系统行为、取舍理由）→ 合格。失效注释的判定：注释描述的代码路径、参数、外部行为在当前文件中已不存在。
```go
// ❌ count++ // 将 count 加 1                —— 删掉不丢信息
// ❌ // 此处处理旧版 API 兼容逻辑             —— 所述逻辑已删除，失效注释
// ✅ // 用 SET NX 而非先 GET 再 SET，避免并发下的 TOCTOU 竞态
```

**DOC-03** `P2` 待办标记固定格式：`TODO` 带责任人与单号，`FIXME` 说明缺陷，`HACK` 说明绕过原因与移除条件。
```go
// ✅ // TODO(zhangsan): 待 v2.0 后移除此兼容代码 #ISSUE-1234
// ✅ // HACK: 绕过第三方库 Bug，等待官方 v1.5.0 修复后删除
```

---

## 4. 错误处理（ERR）

**ERR-01** `P0` error 必须处理。禁止调用返回 error 的函数而不接收；禁止用 `_` 丢弃，除非紧跟注释说明为何安全。处置二选一：向上返回（优先），或就地记一次日志。
【判定】`_` 丢弃且无注释 → 违反。有注释但理由不是「该函数契约保证不返回 error」→ 违反。资源关闭的 error 见 FUNC-03。
```go
// ❌ os.Remove(tmpFile)   ·   json.Unmarshal(data, &result)   ·   _ = tx.Commit()
// ✅ if err := os.Remove(p); err != nil { return fmt.Errorf("remove temp file %q: %w", p, err) }
// ✅ n, _ := buf.Write(p) // bytes.Buffer.Write 文档保证永不返回非 nil error
```

**ERR-02** `P1` 错误先行、提前 return，正常路径保持最外层缩进。禁止把正常逻辑写进 `else`。
```go
// ❌ if err != nil { return err } else { ...正常逻辑... }
// ✅ if err != nil { return fmt.Errorf("parse input: %w", err) }  然后正常逻辑不缩进
```

**ERR-03** `P1` 向上传递错误时用 `%w` 包装并补充上下文，禁止丢弃原始错误另造新 error。
```go
// ❌ return errors.New("operation failed")      ✅ return fmt.Errorf("get user %q: %w", userID, err)
```

**ERR-04** `P1` 禁止「既记日志又返回错误」。中间层只包装返回，日志只在顶层（`main`、HTTP handler 等边界）记一次。
【判定】同一函数内既有日志调用、又 return 非 nil error → 违反。**例外**：该层日志字段包含上游不持有的值（重试次数、内部任务 ID、供应商原始响应码）时，允许记一次结构化日志，但 message 不得复述 error 文本。
```go
// ❌ log.Printf("get user failed: %v", err); return err
// ✅ 顶层：logger.Error("处理请求失败", "err", err); http.Error(w, "Internal Server Error", 500)
```

**ERR-05** `P1` 按调用方需求选择错误形态：

| 场景 | 做法 | 调用方 |
|---|---|---|
| 静态消息，需判定类别 | 哨兵错误 `var ErrXxx = errors.New(...)` | `errors.Is` |
| 动态消息，需读取字段 | 自定义错误类型（`Error()` 方法） | `errors.As` |
| 无需匹配 | `fmt.Errorf("...: %w", err)` | — |

【判定】看调用方代码：有 `errors.Is` 需求却只有 `fmt.Errorf` → 违反；定义了自定义错误类型但无人读其字段 → 过度设计，也算违反。

**ERR-06** `P1` 库代码禁止 panic，一律返回 error。panic 仅三类场景合法：① `main` 中初始化配置明显错误；② 不应到达的代码路径（逻辑 Bug 断言）；③ 必须成功的操作用 `Must` 包装（`template.Must(...)`）。
```go
// ❌ panic(fmt.Sprintf("invalid user ID: %s", s))      ✅ return 0, fmt.Errorf("invalid user ID %q: %w", s, err)
```

**ERR-07** `P1` 错误消息小写开头、句末不加标点、不用 "failed to"/"error" 前缀 —— 错误会被 `%w` 层层拼接，大写与句号会在链条中间产生断句。
```go
// ❌ errors.New("Failed to get user.")   → "handle request: Failed to get user.: sql: no rows"
// ✅ fmt.Errorf("get user %q: %w", id, err)   → "handle request: get user \"42\": sql: no rows"
```

**ERR-08** `P0` 判定错误一律用 `errors.Is`/`errors.As`，禁止 `==` 比较和直接类型断言 —— ERR-03 要求 `%w` 包装，包装后 `==` 与断言必然失效，且不报错、只是静默走错分支。
```go
// ❌ if err == ErrNotFound {   ·   if e, ok := err.(*ValidationError); ok {
// ✅ if errors.Is(err, ErrNotFound) {   ·   var ve *ValidationError; if errors.As(err, &ve) {
```

---

## 5. 函数与方法（FUNC）

> `error` 永远是最后一个返回值 —— 属 Go 默认写法，不单列规则。

**FUNC-01** `P1` 裸返回仅允许在函数体 ≤4 行时使用。既不裸返回、也不在 `defer` 中回写时，就不要命名返回值（纯属噪音）。**唯一例外**：需在 `defer` 中回写 error 时，命名返回值是必需的（见 FUNC-03）。
```go
// ✅ 函数体 4 行内：func split(p string) (dir, file string) { ...; return }
// ❌ 函数体 ≥5 行出现裸返回
// ❌ func parseSize(s string) (size int64, err error) { size, err = ...; return size, err }  —— 命名毫无作用
```

**FUNC-02** `P1` 参数超过 3 个时改用 Options 结构体。
【判定】`context.Context` 与 variadic 选项参数**不计入**（前者是 Go 固定首参约定，后者本身就是 Options 的等价写法），只数真正的业务参数。
```go
// ❌ func NewServer(host string, port int, timeout time.Duration, maxConn int, tls bool) *Server
// ✅ func NewServer(opts ServerOptions) *Server
// ✅ func Submit(ctx context.Context, kind, payload string, id int64, opts ...Option) error  —— 业务参数 3 个
```

**FUNC-03** `P0` 资源释放与解锁一律用 `defer`，紧跟在获取成功之后。**只读资源**可直接 `defer f.Close()`；**写入型资源**（文件写、`bufio.Writer`、`gzip.Writer`、DB 事务）必须检查关闭/刷新的 error —— 忽略它意味着数据可能未落盘却上报成功。
【判定】看资源是否存在缓冲：`Close`/`Flush` 可能返回「数据未真正写出」的 error 即为写入型。
```go
// ✅ 只读：defer f.Close()        ✅ 锁：s.mu.Lock(); defer s.mu.Unlock()
// ❌ 写入：defer w.Close()  —— w 是 *gzip.Writer，未 flush 的数据丢失且无人知晓
// ✅ 写入：用命名返回值回写
func writeFile(path string, data []byte) (err error) {
    f, err := os.Create(path)
    if err != nil {
        return fmt.Errorf("create file: %w", err)
    }
    defer func() {
        if cerr := f.Close(); cerr != nil && err == nil {
            err = fmt.Errorf("close file: %w", cerr)
        }
    }()
    _, err = f.Write(data)
    return err
}
```

**FUNC-04** `P0` 禁止在循环体内直接 `defer`（`defer` 在函数返回时才执行，不是代码块结束时）—— 长循环会累积到句柄耗尽。需即时释放必须用匿名函数包裹。
```go
// ❌ for _, f := range files { fd, _ := os.Open(f); defer fd.Close() }
// ✅ for _, f := range files { func() { fd, err := os.Open(f); if err != nil { return }; defer fd.Close(); process(fd) }() }
```

**FUNC-05** `P0` 同一类型的方法不得混用值接收器与指针接收器。需修改接收者状态、类型含 `sync.Mutex` 等不可复制字段、或结构体较大时，一律用指针接收器。
```go
// ❌ func (c Cache) Get(k string) any { c.mu.RLock(); ... }  —— 值接收器复制了 Mutex，锁保护的是副本
// ✅ func (c *Cache) Get(k string) any { c.mu.RLock(); ... }
// ✅ 小的不可变值类型可用值接收器，但同一类型必须全部一致：func (p Point) String() string
```

**FUNC-06** `P1` 构造函数统一 `New` 前缀：包内只有一个主类型时用 `New`（调用处读作 `user.New()`，见 NAME-04）；一包多个类型时用 `NewXxx`。四项要求：

- **默认返回 `*Xxx` 而非 `Xxx`** —— 仅当该类型按 FUNC-05 可用值接收器时才返回值类型。返回值类型会让每次赋值与传参都复制整个结构，且一旦有指针接收器方法就同时踩中 FUNC-05。
- **返回具体类型而非接口** —— 调用方需要接口时由消费方自行声明（见 IFACE-01）。
- **不变量一次性建立完毕** —— 禁止「先 `New` 再调一串 setter 才能用」的两段式初始化；内部 map、channel、锁保护的字段都在此初始化好（见 TYPE-04）。
- **参数非法返回 error，不 panic**（见 ERR-06）；业务参数超过 3 个时按 FUNC-02 改用 Options 结构体。

【判定】四项任一命中即违反：① `New` 返回值类型，而该类型按 FUNC-05 应当用指针接收器；② `New` 的返回类型是接口；③ `New` 的返回值不再调用额外的初始化/赋值方法就不可用（panic 或走错分支）；④ 参数非法时 panic。
```go
// ❌ func NewCache(n int) Cache                    —— Cache 含 mu 与 map，返回值类型：复制即破锁（FUNC-05）
// ❌ func NewClient(addr string) Clienter          —— 返回接口，调用方拿不到具体类型的其余方法
// ❌ c := NewCache(); c.SetTTL(d); c.Init()        —— 两段式：Init 之前 c 不可用，且无机制强制调用
// ✅ func New(cfg Config) (*Client, error) {
//        if cfg.Addr == "" { return nil, errors.New("addr is required") }
//        return &Client{addr: cfg.Addr, conns: make(map[string]*conn)}, nil   // 不变量在此建立完毕
//    }
// ✅ 值类型例外：func NewPoint(x, y int) Point —— 按 FUNC-05 可用值接收器
```

**FUNC-07** `P1` `init()` 只允许做**不可失败、无外部副作用**的赋值：预编译正则/模板、构建查找表、向注册表自注册。禁止在 `init()` 中建立数据库/网络连接、读文件、读环境变量决定行为、启动 goroutine、做重试或 `time.Sleep`。这些工作放显式的 `New`/`Setup`/`run()`，由 `main` 按确定顺序调用（见 LAYOUT-01）—— `init()` 的执行时机由 import 图决定，不可控、无法传 ctx、失败无处返回，且只要包被导入就会执行（哪怕本次只用到该包的一个常量）。
【判定】`init()` 内出现 I/O、`os.Getenv`、`go` 语句、重试或 sleep → 违反。仅含 `regexp.MustCompile`、`register(...)`、map 字面量填充 → 合规；`MustXxx` 形式的 panic 属允许的例外（模式写错是程序错误，非运行时故障，见 ERR-06）。
```go
// ❌ func init() { db, _ = sql.Open(os.Getenv("DSN")); go startReaper() }   —— 连接失败无人知，goroutine 无人管
// ✅ func init() { providers.Register("openai", build) }                    —— 纯注册，不可失败
// ✅ var reTag = regexp.MustCompile(`^[a-z]+$`)                             —— 包级变量即可，无需 init
```

---

## 6. 包与模块（PKG）

**PKG-01** `P1` 一个包只承担一个清晰职责。禁止 `util`/`common`/`helper` 容器包，包名本身应描述功能。
【判定】能否用**一个名词短语**概括该包全部导出符号。需要用「和」连接两个不相关的名词才能说清 → 违反。
```go
// ❌ package util —— 塞进 FormatDate / ValidateEmail / ParseUserID / SendHTTPRequest
// ✅ package timeutil / validate / userid / httpclient
```

**PKG-02** `P2` 不对外暴露的实现放 `internal/`，供外部复用的放 `pkg/`。

---

## 7. 数据类型（TYPE）

> 以下属 Go 默认写法，不单列规则：判空用 `len(x) == 0` 而非 `x == nil`；取 map 值用双返回值 `v, ok := m[k]`。

**TYPE-01** `P1` 空 slice 用 `var items []string` 声明（nil slice 可直接 append/len/range），禁止无必要的 `[]string{}`。
【判定】**例外**：字段带 `json` tag 且契约要求空数组时必须用 `[]T{}` —— nil slice 序列化成 `null` 而非 `[]`，破坏调用方契约。
```go
// ✅ 内部：var items []string        ✅ API 响应必须是 [] 而非 null：Response{Items: []Item{}}
```

**TYPE-02** `P2` 已知目标长度时预分配 slice 与 map 容量：`make([]string, 0, expectedLen)`、`make(map[string]int, len(keys))`。全文其他位置提到「预分配」均指本条。

**TYPE-03** `P0` 在函数边界复制 slice，避免外部持有内部状态引用 —— 存入与返回两个方向都要复制。
【判定】只约束**导出的**构造函数、方法与跨包传递，且仅当该 slice 被**接收者长期持有**时。包内私有函数间传递、函数内新建的返回值，均不必复制。
```go
// ❌ 存入：s.items = items                              ✅ s.items = make([]Item, len(items)); copy(s.items, items)
// ❌ 返回：func (s *S) Items() []Item { return s.items } ✅ 复制到新 slice 再返回
```

**TYPE-04** `P0` map 必须初始化后再写入 —— nil map 读取返回零值，写入直接 panic。容量预分配见 TYPE-02。
```go
// ❌ var m map[string]int; m["k"] = 1      ✅ m := make(map[string]int)
```

**TYPE-05** `P2` 结构体初始化必须指定字段名，零值字段省略。
```go
// ❌ csv.Reader{',', '#', 4, false}      ✅ csv.Reader{Comma: ',', Comment: '#', FieldsPerRecord: 4}
```

**TYPE-06** `P2` 参与序列化的结构体必须写 struct tag，敏感字段用 `json:"-"`。
```go
// ✅ DatabaseHost string `json:"database_host" yaml:"database_host"`   ·   Password string `json:"-"`
```

**TYPE-07** `P1` 时间与时长一律用 `time.Time`/`time.Duration`，禁止裸整数 —— 裸整数丢失单位信息，调用方必然会传错。
```go
// ❌ func poll(delay int) / CreatedAt int64      ✅ func poll(delay time.Duration) / CreatedAt time.Time
```

**TYPE-08** `P1` 外部系统不支持时间类型时，单位必须写进字段名：`TimeoutMillis int`。

**TYPE-09** `P1` 任意类型一律写 `any`，禁止 `interface{}`（Go 1.18 起 `any` 是其官方别名，完全等价，混用只是风格分裂）。`any` 是类型系统的逃逸口，只在确实无法确定类型时使用（反序列化目标、透传元数据）；能用具体类型或泛型表达的不要退化为 `any`。
```go
// ❌ map[string]interface{} / func Log(args ...interface{})      ✅ map[string]any / func Log(args ...any)
```

---

## 8. 并发（CONC）

**CONC-01** `P1` channel 容量只用 0（同步通信）或 1（允许一次非阻塞发送）。非 0/1 容量必须在注释中论证两点：**容量如何得出**、**填满后的行为**。
【判定】容量为字面量魔数（`100`、`1024`）且无注释 → 违反。容量由 `len(tasks)` 等有界表达式推导并注明「每项只发一次故永不阻塞」→ 合规。
```go
// ✅ done := make(chan struct{})   ·   result := make(chan error, 1)
// ✅ results := make(chan Result, len(tasks)) // 容量 = 任务数，有界；每任务只发一次，故永不阻塞
// ❌ c := make(chan int, 100)  —— 容量无依据，填满时行为未定义
```

**CONC-02** `P0` 禁止 fire-and-forget goroutine。每个 goroutine 必须有终止机制（`context` 取消或 channel 关闭）**和**等待退出的手段（`sync.WaitGroup`）。
【判定】`go func(){...}()` 内含无 `select { case <-ctx.Done(): }` 的 `for` 循环，或调用方没有 `wg.Wait()` 能确认其退出 → 违反。进程退出时该 goroutine 的工作会被静默截断。
```go
// ❌ go func() { for { process(queue) } }()
// ✅ wg.Add(1); go func() { defer wg.Done(); for { select { case <-ctx.Done(): return; case t, ok := <-queue:
//        if !ok { return }; processTask(t) } } }()   —— 调用方 cancel() 停止、wg.Wait() 确认退出
```

**CONC-03** `P1` 禁止嵌入 `sync.Mutex`（会把 `Lock`/`Unlock` 提升为公共 API，外部可越过封装加锁），必须作为私有字段。
```go
// ❌ type SMap struct { sync.Mutex; data map[string]string }      ✅ type SMap struct { mu sync.Mutex; data ... }
```

**CONC-04** `P1` 读多写少场景用 `sync.RWMutex`（读 `RLock`、写 `Lock`）。纯计数器用 `atomic.Int64`，不要用 mutex 保护一个 int。

**CONC-05** `P0` 共享变量禁止无保护并发读写（含并发访问的 map）。用锁、`sync/atomic` 原子类型，或通过 channel 传递数据避免共享内存。
```go
// ❌ var counter int; go func() { counter++ }()      ✅ var counter atomic.Int64; go func() { counter.Add(1) }()
// ❌ 多 goroutine 直接读写同一 map                    ✅ sync.Map 或 RWMutex 保护（锁选择见 CONC-04）
```

**CONC-06** `P0` goroutine 或闭包内使用循环变量，必须显式传参或在循环体内重新声明 —— **Go 1.21 及更早版本循环变量复用同一份存储**，直接捕获会让所有 goroutine 读到最后一次迭代的值。
【判定】Go 1.22+ 已改为每次迭代新建变量，此坑消失；审查 `go.mod` 声明 ≥1.22 的项目时**不报此项**。
```go
// ❌ for _, item := range items { go func() { process(item) }() }
// ✅ go func(it Item) { process(it) }(item)   ·   ✅ item := item 后再捕获
```

**CONC-07** `P0` 并发扇出后必须汇总每个分支的 error，禁止只 `wg.Wait()` 就当成功。首个错误应能取消其余分支。
【判定】`wg.Wait()` 之后没有任何对分支 error 的读取 → 违反：全部分支失败也会报成功。
```go
// ❌ for _, it := range items { wg.Add(1); go func(x Item){ defer wg.Done(); process(x) }(it) }; wg.Wait()
// ✅ 标准库：errCh := make(chan error, len(items)) // 容量=分支数，有界；每分支只发一次，故永不阻塞（满足 CONC-01）
//    wg.Wait() → close(errCh) → range 收集 → errors.Join(errs...)
// ✅ 已引入 golang.org/x/sync：errgroup.WithContext + g.Go + g.Wait()，自带首错取消，优先用它
```
「返回全部错误还是首个错误」由调用方需求决定：需逐项报告用 `errors.Join`，只需快速失败用 `errgroup`。

---

## 9. 上下文（CTX）

**CTX-01** `P1` 涉及 I/O、跨进程调用、可能阻塞或需要取消的函数，`context.Context` 必须是第一个参数，命名固定为 `ctx`。纯计算函数不要为了「统一」硬加 ctx。
```go
// ❌ func (s *Service) Get(id string, ctx context.Context)   ·   ❌ func Sum(ctx context.Context, xs []int) int
// ✅ func (s *Service) Get(ctx context.Context, id string) (*User, error)
```

**CTX-02** `P0` ctx 必须逐层透传到最底层的 I/O 调用，禁止在中间层丢弃后另起一个 —— 丢弃会让上游超时与取消在这一层静默失效：请求已断，下游仍在跑。
【判定】函数签名收了 `ctx` 但函数体内的 I/O 调用未使用它，或体内出现 `context.Background()`/`TODO()` → 违反。
```go
// ❌ func (r *Repo) Get(ctx context.Context, id string) { return r.db.Query("...", id) }   —— ctx 收了不用
// ❌ r.db.QueryContext(context.Background(), ...)   —— 中途换新 ctx，取消链断开
// ✅ return r.db.QueryContext(ctx, "...", id)
```

**CTX-03** `P0` 禁止把 `context.Context` 存进结构体字段 —— ctx 的生命周期属于单次调用，存起来会导致第二次调用复用第一次的取消状态。
【判定】**例外**：长期运行的后台组件（worker、连接池）可持有一个代表**自身生命周期**的 ctx，但字段名须写明意图（如 `runCtx`），且不得用它服务外部请求。
```go
// ❌ type Service struct { ctx context.Context; repo Repo }      ✅ type Service struct { repo Repo }  // ctx 走参数
```

**CTX-04** `P0` 禁止传 nil ctx；生产代码路径禁止 `context.TODO()`。没有上游 ctx 的起点（`main`、后台任务、测试）用 `context.Background()`。
```go
// ❌ svc.Get(nil, id)  —— 下游 ctx.Done() 直接 panic      ❌ svc.Get(context.TODO(), id)  —— TODO 是待补标记
// ✅ ctx := context.Background()   // main / 后台循环的根
```

**CTX-05** `P0` `context.WithTimeout`/`WithCancel` 返回的 `cancel` 必须 `defer cancel()`，紧跟在创建之后 —— 不调用会泄漏定时器与父 ctx 上的关联，直到超时才释放。
```go
// ✅ ctx, cancel := context.WithTimeout(ctx, 3*time.Second); defer cancel()
```

**CTX-06** `P1` `context.WithValue` 只用于传递请求作用域的元数据（trace ID、认证主体），禁止用来传函数的必要参数或依赖。key 必须是包内私有的自定义类型，不能用 string。
```go
// ❌ context.WithValue(ctx, "userID", id)  —— string key 跨包碰撞
// ❌ 用 ctx 传 *sql.DB、配置、必填业务参数   —— 依赖应显式注入，参数应显式声明
// ✅ type ctxKey struct{}; ctx = context.WithValue(ctx, ctxKey{}, traceID)
```

---

## 10. 接口（IFACE）

**IFACE-01** `P1` 接口定义在使用方（消费方），只声明使用者真正需要的方法。禁止在实现包预先导出大接口。
【判定】接口与其**唯一实现**位于同一个包 → 违反。**例外**：项目架构规范（分层架构、DDD 端口与适配器等）明确规定了某类接口的归属位置时以架构规范为准 —— 那是被依赖侧刻意声明的契约，本条只约束**没有架构规定时**的默认做法。
```go
// ❌ userservice/interface.go 里导出含全部方法的 UserServiceInterface
// ✅ handler 包内：type userFetcher interface { GetUser(ctx context.Context, id string) (*User, error) }
```

**IFACE-02** `P1` 接口小而单一，需要更多能力时用组合（`ReadWriter { Reader; Writer }`）。
【判定】单个接口超过 4 个方法即需重新审视是否该拆分。

---

## 11. 测试（TEST）

**TEST-01** `P2` 多输入场景用表驱动测试 + `t.Run` 子测试。
【判定】三项缺一即违反：用例结构体含 `name` 字段；`t.Run(tt.name, ...)` 而非裸循环；错误信息带上**输入值与期望值**（`t.Errorf("ParseUserID(%q) = %v, want %v", tt.input, got, tt.want)`）。

**TEST-02** `P1` 需测私有方法 → 同包 `package user`；只测公共 API → 独立包 `package user_test`。

**TEST-03** `P2` 测试辅助函数首行必须 `t.Helper()`，使失败信息指向调用位置。

**TEST-04** `P2` 测试中初始化失败用 `t.Fatalf`，禁止 `panic`；清理用 `t.Cleanup` 而非 `defer`。
```go
// ✅ if err != nil { t.Fatalf("打开测试数据库失败: %v", err) }; t.Cleanup(func() { db.Close() })
```

---

## 12. 性能（PERF）

> 本章**只在热路径**（profile 或 benchmark 证明的瓶颈、高频循环、单请求内大量重复调用）上强制。冷路径（启动、配置加载、错误分支、日志组装）以可读性为先，不得为省一次分配牺牲清晰度。
> 【判定】本章仅在有 profile/benchmark 证据时按 `P2` 处置（汇总计数，见 APPLY-02）；无证据时**完全不报**，也不计入 `P2` 汇总。

**PERF-01** `P2` 避免不必要的 `string` ↔ `[]byte` 转换（每次转换复制一份内存）。首选让函数直接接受 `string`；确需 `[]byte` 且循环内用同一份数据时，把转换提到循环外。
```go
// ❌ for _, s := range list { process([]byte(s)) }      ✅ 改 process 接受 string，从根上不转换
```

**PERF-02** `P2` 单个数字转字符串用 `strconv`，不用 `fmt.Sprintf`（fmt 有反射开销）。
【判定】只看格式串占位符个数：1 个 → 用 `strconv`；2 个及以上 → 用 `fmt` 合理。字面量前后缀不改变判定，用 `+` 拼接。
```go
// ❌ fmt.Sprintf("%d", n)   ·   ❌ fmt.Sprintf("user_%d", id)      ✅ strconv.Itoa(n)   ·   ✅ "user_" + strconv.Itoa(id)
// ✅ fmt.Sprintf("(%d,%d)", x, y)  —— 两个占位符，不属本条
```

---

## 13. 项目结构（LAYOUT）

**LAYOUT-01** `P1` `main` 只负责启动，业务流程放 `run() error`；`os.Exit`、`log.Fatal` 只允许出现在 `main` 中 —— 其他位置调用会跳过所有 `defer`，导致资源不释放。
```go
func main() {
    if err := run(context.Background()); err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
}
// run(ctx) 内部：加载配置 → 连接依赖（defer 关闭）→ 启动 server，每步错误以 %w 包装后 return
```

---

## 附录：审查执行细则

**本附录仅在审查代码时适用，编写代码时不必读。**

**APPLY-03** 审查输出契约

- 每条违规一行：`<文件>:<行> · <规则编号> · <级别> · 一句话问题 · 违反后果`。后果必须写具体现象（「取消信号丢失，请求已断下游仍在跑」），不写「不符合规范」。
- **不报**：APPLY-01 判定为「局部一致优先」的偏离；本文件未覆盖的个人风格偏好；已在 CI lint 中拦截的规则（见 APPLY-04）—— 除非该项目未配置 lint。
- 判定依赖运行时信息或缺失上下文（如无法确定某函数是否在热路径、某 map 是否真被并发访问）时，标注**待确认**并说明缺什么，不得断言违反。
- 禁止为凑数报 `P2`。一次审查若只有 `P2`，结论就写「无 P0/P1 问题」。

**APPLY-04** 工具覆盖：下表中的规则可由 lint 全自动检查，**审查时不逐条人工比对**，跑一次 lint 即可。**未列入下表的规则一律靠人工判定**，其中最吃判断力的是 DOC-02、ERR-04、ERR-05、FUNC-03、FUNC-06、FUNC-07、PKG-01、CONC-01/02/07、CTX-02/03、IFACE-01。

| 规则 | 工具 |
|---|---|
| NAME-01/02/03/04/06/07、DOC-01 | `revive` |
| FMT-01 | `gci` / `goimports` |
| ERR-01 | `errcheck` |
| ERR-08 | `staticcheck` |
| FUNC-01（裸返回）、FUNC-04（循环内 defer） | `nakedret`、`gocritic` |
| FUNC-05（接收器混用） | `staticcheck` ST1016 |
| CONC-03（嵌入 Mutex）、CONC-05（数据竞争） | `go vet`、`go test -race` |
| CONC-06（循环变量捕获） | `copyloopvar` / `loopclosure` |
| CTX-01（ctx 首参）、CTX-03（ctx 入结构体）、CTX-05（漏 cancel） | `revive:context-as-argument`、`containedctx`、`govet:lostcancel` |
| TYPE-05、TYPE-09 | `govet:composites`、`revive` |
| PERF-02 | `perfsprint` |

规则文本仍是唯一权威（工具可能未启用或误报），但**默认信任工具结论**，不重复劳动。
