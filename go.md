# Go 编码规则（AI 编程约束）

适用语言版本：Go 1.21+

编写、修改、审查 Go 代码时必须遵守。每条规则带唯一编号，供评审时引用（如 `违反 NAME-02`）。
标注 `[补充]` 的内容为源规范文档未包含、额外添加的部分；标注可位于整条规则，也可位于规则内的某个例外或子句。

冲突时优先级：**格式（gofmt 统一，不争论）> 命名（自描述）> 错误处理（显式）> 并发（谨慎，所有权明确）**。

---

## 0. 规则的适用方式

**APPLY-01** `[补充]` 写新代码前先读同目录既有代码。周边风格与本规则冲突时，按**改动性质**分三档处理：

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

---

## 1. 命名（NAME）

**NAME-01** 一律 MixedCaps 驼峰，禁止下划线。导出大写开头，未导出小写开头。测试函数名（`TestXxx_Yyy`）除外。
```go
// ✅ UserService / GetUserByID / MaxRetryCount / userCache / buildQuery / defaultTimeout
// ❌ user_service / MAX_RETRY_COUNT / get_user_by_id
```

**NAME-02** 缩写词整体大写或整体小写，禁止混合。
```go
// ✅ XMLParser  xmlParser  userID  UserID  parseURL
// ❌ XmlParser  userId     parseUrl
// 适用：URL URI HTTP HTTPS ID DB SQL API JSON XML RPC gRPC
```

**NAME-03** 包名全小写、无分隔符、简短（1–2 个词直接拼接），禁止下划线与驼峰。包名的职责要求见 PKG-01。
```go
// ✅ package user / httputil / sqldb
// ❌ package user_service / userService / httpUtil
```
`[补充]`「1–2 个词」：源文档表述为「单个单词」，但其自身示例 `httputil` 即两词拼接，此处按示例修正。

**NAME-04** 导出符号不得重复包名。
```go
// ❌ user.UserService    ✅ user.Service
```

**NAME-05** 变量名长度随作用域增长（照下例示范）。
```go
for i, v := range items { ... }                      // 循环、if 块：单字母可接受
count := 0; retryDelay := 5 * time.Second            // 函数内：描述性单词
func ProcessUserRegistration(ctx context.Context, req *RegisterRequest) error  // 参数、包级：完整名称
var defaultHTTPClient = &http.Client{Timeout: 10 * time.Second}
```

**NAME-06** 接收器名为 1–2 字母类型缩写，同一类型所有方法必须一致。禁止 `this`/`self`。
```go
// ✅ func (u *User) Name() string   func (u *User) SetName(n string)   func (c *Client) Do(...)
// ❌ func (this *User) / func (self *Client) / 同类型混用 u 与 user
```

**NAME-07** 常量按角色命名而非按值命名；禁止全大写下划线、禁止 `k` 前缀。
```go
// ✅ MaxPacketSize = 512        DefaultTimeout = 30 * time.Second
// ❌ MAX_PACKET_SIZE = 512      kDefaultTimeout = 30
```

**NAME-08** iota 枚举从 `iota + 1` 开始，避免零值歧义（`North Direction = iota + 1`）。**例外** `[补充]`：零值本身有明确语义时（如 `StatusIdle = 0`）可从 0 开始。

**NAME-09** 哨兵错误用 `Err`/`err` 前缀，自定义错误类型用 `Error` 后缀。
```go
var ErrNotFound = errors.New("not found")
var errInvalidToken = errors.New("invalid token")   // 包内
type NotFoundError struct{ Resource, ID string }
```

**NAME-10** `[补充]` 标识符只能用英文单词，禁止中文、拼音及拼音缩写。适用于包名、类型、函数、变量、常量、字段、struct tag、文件名。找不到合适英文词时用完整英文短语，不得退化为拼音。
```go
// ❌ type 用户服务 struct{} / YongHuService / GetYongHu / dingdanZhuangtai / ddzt / package yonghu
// ✅ UserService / GetUser / orderStatus / package user

// 中文只允许出现在：注释、日志文本、错误消息、测试用例 name
// logger.Warn("删除临时文件失败", zap.String("file", tmpFile))
// {name: "空字符串", input: "", wantErr: true}
```

---

## 2. 格式（FMT）

> 缩进、对齐、空行、声明分组的括号位置以 gofmt 输出为准，不由本文件单列规则，也不自行调整空白。

**FMT-01** import 分三组、组间空行、组内字母序：标准库 → 第三方 → 项目内部包。
```go
import (
    "context"
    "net/http"

    "go.uber.org/zap"

    "github.com/myorg/myapp/internal/config"
)
```

**FMT-02** 禁止折断参数列表跨行，行过长时提取变量来缩短。参数控制在 3 个以内（见 FUNC-02），行自然不会过长。
```go
// ❌ result, err := someVeryLongFunctionName(firstParam,
//        secondParam, thirdParam)
// ✅ params := buildParams(firstParam, secondParam)
//    result, err := someVeryLongFunctionName(params, thirdParam)
```

---

## 3. 注释（DOC）

**DOC-01** 所有导出标识符必须有文档注释：以标识符名开头，完整句子。同一 const 组已有组注释、且常量为字面量映射时，单条豁免。包注释以 `Package <name>` 开头。
```go
// Package user 提供用户管理相关功能，包括注册、认证和权限控制。
package user

// GetByID 根据用户 ID 获取用户信息。
// 如果用户不存在，返回 ErrNotFound。
// ctx 用于控制超时和取消。
func (s *UserService) GetByID(ctx context.Context, id string) (*User, error)
```

**DOC-02** 注释必须说明「为什么这么做」（业务原因、算法选型、取舍依据），禁止复述代码字面含义。禁止留下失效注释 —— 修改代码时必须同步更新或删除已不成立的注释。
```go
// ❌ count++ // 将 count 加 1                    —— 复述代码
// ❌ // 此处处理旧版 API 兼容逻辑                 —— 所述逻辑已被删除，属失效注释
// ✅ // 用 SET NX 而非先 GET 再 SET，避免并发下的竞态（TOCTOU）
// ✅ // 布隆过滤器预检，避免缓存穿透打穿数据库；误判率约 0.1%，业务可接受
```

**DOC-03** 待办标记固定格式：`TODO` 带责任人与单号，`FIXME` 说明缺陷，`HACK` 说明绕过原因与移除条件。
```go
// TODO(zhangsan): 待 v2.0 后移除此兼容代码 #ISSUE-1234
// FIXME: 高并发下存在竞态条件，需加分布式锁
// HACK: 绕过第三方库 Bug，等待官方 v1.5.0 修复后删除
```

---

## 4. 错误处理（ERR）

**ERR-01** error 必须处理。禁止调用返回 error 的函数而不接收；禁止用 `_` 丢弃，除非紧跟注释说明为何安全。
```go
// ❌ os.Remove(tmpFile)        json.Unmarshal(data, &result)
// ✅
if err := os.Remove(tmpFile); err != nil {
    logger.Warn("删除临时文件失败", zap.String("file", tmpFile), zap.Error(err))
}
n, _ := buf.Write(p) // bytes.Buffer.Write 文档保证永不返回非 nil error
```
资源关闭的 error 是否可忽略，见 FUNC-03。

**ERR-02** 错误先行、提前 return，正常路径保持最外层缩进。禁止把正常逻辑写进 `else`。
```go
// ✅
data, err := parse(input)
if err != nil {
    return "", fmt.Errorf("parse input: %w", err)
}
return transform(data)
```

**ERR-03** 向上传递错误时用 `%w` 包装并补充上下文，禁止丢弃原始错误另造新 error。
```go
// ❌ return errors.New("operation failed")
// ✅ return fmt.Errorf("get user %q: %w", userID, err)
```

**ERR-04** 禁止「既记日志又返回错误」。中间层只包装返回，日志只在顶层（`main`、HTTP handler 等边界）记一次。当某层持有顶层无法重建的上下文时，允许记一次结构化日志，但 message 不得复述 error 文本。
```go
// ❌ log.Printf("get user failed: %v", err); return err
// ✅ 顶层：logger.Error("处理请求失败", zap.Error(err)); http.Error(w, "Internal Server Error", http.StatusInternalServerError)
```

**ERR-05** 按调用方需求选择错误形态：

| 场景 | 做法 | 调用方 |
|---|---|---|
| 静态消息，需判定类别 | 哨兵错误 `var ErrXxx = errors.New(...)` | `errors.Is` |
| 动态消息，需读取字段 | 自定义错误类型（`Error()` 方法） | `errors.As` |
| 无需匹配 | `fmt.Errorf("...: %w", err)` | — |

**ERR-06** 库代码禁止 panic，一律返回 error。
```go
// ❌ panic(fmt.Sprintf("invalid user ID: %s", s))
// ✅ return 0, fmt.Errorf("invalid user ID %q: %w", s, err)
```
panic 仅三类场景合法：① `main` 中初始化配置明显错误；② 不应到达的代码路径（逻辑 Bug 断言）；③ 必须成功的操作用 `Must` 包装 —— `var tmpl = template.Must(template.New("").Parse(s))`。

**ERR-07** `[补充]` 错误消息小写开头、句末不加标点、不用 "failed to"/"error" 前缀。错误会被 `%w` 层层拼接，首字母大写与句号会在链条中间产生断句。
```go
// ❌ errors.New("Failed to get user.")   → "handle request: Failed to get user.: sql: no rows"
// ✅ fmt.Errorf("get user %q: %w", id, err)   → "handle request: get user \"42\": sql: no rows"
```

**ERR-08** `[补充]` 判定错误一律用 `errors.Is`/`errors.As`，禁止 `==` 比较和直接类型断言。ERR-03 要求用 `%w` 包装，包装后 `==` 与断言必然失效，且失效时不报错、只是静默走错分支。
```go
// ❌ if err == ErrNotFound {                    —— 被 %w 包装后永远为 false
// ❌ if e, ok := err.(*ValidationError); ok {
// ✅ if errors.Is(err, ErrNotFound) {
// ✅ var ve *ValidationError
//    if errors.As(err, &ve) { /* 可访问 ve.Field */ }
```

---

## 5. 函数与方法（FUNC）

> `error` 永远是最后一个返回值 —— 属 Go 默认写法，不单列规则。

**FUNC-01** 函数体 5 行及以上即为长函数，禁止裸返回（naked return）。仅函数体 4 行以内可用命名返回值配合裸返回。既不裸返回、也不在 `defer` 中回写时，就不要命名返回值（纯属噪音）。
```go
// ✅ 函数体 4 行，命名返回值 + 裸返回
func split(path string) (dir, file string) {
    i := strings.LastIndex(path, "/")
    dir, file = path[:i+1], path[i+1:]
    return
}
// ✅ 不用裸返回就不命名：   func divide(a, b float64) (float64, error)
// ❌ 命名了，却既不裸返回也不在 defer 中回写 —— 命名毫无作用
func parseSize(s string) (size int64, err error) {
    size, err = strconv.ParseInt(s, 10, 64)
    return size, err
}
// ❌ 函数体 5 行及以上出现裸返回
```
**唯一例外**：需要在 `defer` 中回写 error 时，命名返回值是必需的，不算噪音（见 FUNC-03）。

**FUNC-02** 参数超过 3 个时改用 Options 结构体。
```go
// ❌ func NewServer(host string, port int, timeout time.Duration, maxConn int, tls bool) *Server
// ✅ func NewServer(opts ServerOptions) *Server
```

**FUNC-03** 资源释放与解锁一律用 `defer`，紧跟在获取成功之后。`[补充]` **只读资源**可直接 `defer f.Close()`；**写入型资源**（文件写、`bufio.Writer`、`gzip.Writer`、事务）必须检查关闭/刷新的 error —— 忽略它意味着数据可能未落盘却上报成功（源文档只示范了 `defer f.Close()`，未区分读写）。
```go
// ✅ 只读：f, err := os.Open(path); ...; defer f.Close()  —— Close 失败不影响已读到的数据
// ✅ 锁：  s.mu.Lock(); defer s.mu.Unlock()
// ❌ 写入：defer w.Close()  —— w 是 *gzip.Writer：未 flush 的数据丢失且无人知晓

// ✅ 写入型资源：用命名返回值回写 Close 的 error
func writeFile(path string, data []byte) (err error) {
    f, err := os.Create(path)
    if err != nil {
        return fmt.Errorf("create file: %w", err)
    }
    defer func() {
        if cerr := f.Close(); cerr != nil && err == nil {
            err = fmt.Errorf("close file: %w", cerr) // 否则写入失败会被静默吞掉
        }
    }()
    _, err = f.Write(data)
    return err
}
```

**FUNC-04** 禁止在循环体内直接 `defer`（`defer` 在函数返回时才执行，不是代码块结束时）。需即时释放必须用匿名函数包裹。
```go
// ❌ for _, file := range files { f, _ := os.Open(file); defer f.Close() }
// ✅
for _, file := range files {
    func() {
        f, err := os.Open(file)
        if err != nil {
            return
        }
        defer f.Close()
        processFile(f)
    }()
}
```

**FUNC-05** `[补充]` `ctx context.Context` 必须是第一个参数；禁止存进 struct 字段；禁止传 `nil`（无 ctx 可用时传 `context.TODO()`）；`context.Value` 只用于传递请求域元数据（trace ID、认证信息），禁止传业务参数。
```go
// ✅ func (s *Service) GetUser(ctx context.Context, id string) (*User, error)
// ❌ func (s *Service) GetUser(id string, ctx context.Context)   —— ctx 不在首位
// ❌ type Service struct { ctx context.Context }                 —— ctx 随请求走，不属于对象状态
// ❌ GetUser(nil, id)                                            —— 应传 context.TODO()
// ❌ ctx = context.WithValue(ctx, "userID", id)                   —— 业务参数应走函数签名
```

**FUNC-06** `[补充]` 同一类型的方法不得混用值接收器与指针接收器。需修改接收者状态、类型含 `sync.Mutex` 等不可复制字段、或结构体较大时，一律用指针接收器。
```go
// ❌ func (c Cache) Get(k string) any { c.mu.RLock(); ... }  —— 值接收器复制了 Mutex，锁保护的是副本
// ✅ func (c *Cache) Get(k string) any { c.mu.RLock(); ... }
// ✅ 小的不可变值类型可用值接收器，但同一类型必须全部一致：func (p Point) String() string
```

---

## 6. 包与模块（PKG）

**PKG-01** 一个包只承担一个清晰职责。禁止 `util`/`common`/`helper` 容器包，包名本身应描述功能。
```go
// ❌ package util —— 塞进 FormatDate / ValidateEmail / ParseUserID / SendHTTPRequest
// ✅ package timeutil / validate / userid / httpclient
```

**PKG-02** 不对外暴露的实现放 `internal/`，供外部复用的放 `pkg/`。

---

## 7. 数据类型（TYPE）

> 以下属 Go 默认写法，不单列规则：判空用 `len(x) == 0` 而非 `x == nil`；取 map 值用双返回值 `v, ok := m[k]`。

**TYPE-01** 空 slice 用 `var items []string` 声明（nil slice 合法，可直接 append/len/range）；禁止无必要的 `[]string{}`。
**例外** `[补充]`：参与 JSON 序列化且契约要求空数组的字段，必须用 `[]T{}` —— nil slice 会被序列化成 `null` 而非 `[]`，破坏调用方契约（源文档未涉及序列化差异）。
```go
// ✅ 内部使用：var items []string
// ✅ API 响应必须是 [] 而非 null：resp := Response{Items: []Item{}}
```

**TYPE-02** 已知目标长度时预分配 slice 与 map 容量，避免过程中反复扩容与再哈希：`make([]string, 0, expectedLen)`、`make(map[string]int, len(keys))`。全文其他位置提到「预分配」均指本条。

**TYPE-03** 在函数边界复制 slice，避免外部持有内部状态引用 —— 存入与返回两个方向都要复制。
```go
// ❌ 存入：s.items = items                                  —— 调用方后续改动会穿透进结构体
// ✅ 存入：s.items = make([]Item, len(items)); copy(s.items, items)

// ❌ 返回：func (s *S) Items() []Item { return s.items }     —— 调用方改元素会写穿内部数组
// ✅ 返回：
func (s *S) Items() []Item {
    out := make([]Item, len(s.items))
    copy(out, s.items)
    return out
}
```
`[补充]` 返回方向：源文档只示范了存入方向的 `copy`，未涉及 getter 泄漏内部 slice。仅当返回的是**接收者长期持有**的 slice 时需复制；函数内新建的返回值直接返回，不必复制。

**TYPE-04** map 必须初始化后再写入：nil map 读取返回零值，写入 panic。容量预分配见 TYPE-02。

**TYPE-05** 并发访问的 map 必须用 `sync.Map` 或读写锁保护（锁类型选择见 CONC-04，通用要求见 CONC-05）。

**TYPE-06** 结构体初始化必须指定字段名，零值字段省略。
```go
// ❌ csv.Reader{',', '#', 4, false, false, false, false}
// ✅ csv.Reader{Comma: ',', Comment: '#', FieldsPerRecord: 4}
```

**TYPE-07** 参与序列化的结构体必须写 struct tag，敏感字段用 `json:"-"`。
```go
type Config struct {
    DatabaseHost string `json:"database_host" yaml:"database_host"`
    Password     string `json:"-"`
}
```

**TYPE-08** 时间与时长一律用 `time.Time`/`time.Duration`，禁止裸整数。
```go
// ❌ func poll(delay int) { time.Sleep(time.Duration(delay) * time.Millisecond) }  poll(10)
// ✅ func poll(delay time.Duration) { time.Sleep(delay) }                          poll(10 * time.Second)
// ❌ CreatedAt int64      ✅ CreatedAt time.Time `json:"created_at"`  // RFC3339
```

**TYPE-09** 外部系统不支持时间类型时，单位必须写进字段名：`TimeoutMillis int`。

---

## 8. 并发（CONC）

**CONC-01** channel 容量只用 0（同步通信）或 1（允许一次非阻塞发送）。其他容量必须在注释中论证两点：容量如何得出、填满后的行为。**审查动作**：遇到无论证的魔数容量，必须提示该设计不合理，并指出缺的正是这两点。
```go
// ✅ done := make(chan struct{})    result := make(chan error, 1)
// ✅ results := make(chan Result, len(tasks)) // 容量 = 任务数，有界；每个任务只发一次，故永不阻塞
// ⚠️ c := make(chan int, 100)  —— 需提示：容量 100 缺乏依据，填满时的行为未定义
```

**CONC-02** 禁止 fire-and-forget goroutine。每个 goroutine 必须有终止机制（`context` 取消或 channel 关闭）和等待退出的手段（`sync.WaitGroup`）。
```go
// ❌ go func() { for { process(queue) } }()
// ✅
wg.Add(1)
go func() {
    defer wg.Done()
    for {
        select {
        case <-ctx.Done():
            return
        case task, ok := <-queue:
            if !ok {
                return
            }
            processTask(task)
        }
    }
}()
// 调用方通过 cancel() 停止，wg.Wait() 确认退出
```

**CONC-03** 禁止嵌入 `sync.Mutex`（会把 Lock/Unlock 提升为公共 API），必须作为私有字段。
```go
// ❌ type SMap struct { sync.Mutex; data map[string]string }
// ✅ type SMap struct { mu sync.Mutex; data map[string]string }
```

**CONC-04** 读多写少场景用 `sync.RWMutex`，读路径 `RLock`/`RUnlock`，写路径 `Lock`/`Unlock`。纯计数器用 `atomic.Int64`，不要用 mutex 保护一个 int。

**CONC-05** 共享变量禁止无保护并发读写。用 `sync/atomic` 原子类型，或通过 channel 传递数据避免共享内存。
```go
// ❌ var counter int;  go func() { counter++ }()
// ✅ var counter atomic.Int64;  go func() { counter.Add(1) }()
// ✅ results := make(chan int, len(items)) // 容量 = 任务数，有界；每项只发一次，故永不阻塞（CONC-01 的论证）
//    for _, it := range items { go func(x Item) { results <- compute(x) }(it) }  // 传参，见 CONC-06
```

**CONC-06** `[补充]` goroutine 或闭包内使用循环变量，必须显式传参或在循环体内重新声明。**Go 1.21 及更早版本，循环变量在整个循环中复用同一份存储**，直接捕获会让所有 goroutine 读到最后一次迭代的值。
```go
// ❌ for _, item := range items { go func() { process(item) }() }
// ✅ for _, item := range items { go func(it Item) { process(it) }(item) }      // 显式传参
// ✅ for _, item := range items { item := item; go func() { process(item) }() } // 循环体内重新声明
```
注：Go 1.22+ 改为每次迭代新建变量，此坑消失；但本规则适用 Go 1.21+，低版本仍须显式处理。

---

## 9. 接口（IFACE）

**IFACE-01** 接口定义在使用方（消费方），只声明使用者真正需要的方法。禁止在实现包预先导出大接口。
```go
// ❌ userservice/interface.go 里导出 UserServiceInterface（含全部方法）
// ✅ handler/user.go
type userFetcher interface {
    GetUser(ctx context.Context, id string) (*User, error)
}
type UserHandler struct{ users userFetcher }
```

**IFACE-02** 接口小而单一，需要更多能力时用组合（`ReadWriter { Reader; Writer }`）。`[补充]` 单个接口超过 4 个方法即需重新审视是否该拆分（源规范仅定性表述「大而全」，此阈值为便于判定而设）。

**IFACE-03** 用空白赋值做编译期实现校验，位置紧随类型声明：`var _ http.Handler = (*MyHandler)(nil)`，无运行时开销。

---

## 10. 测试（TEST）

**TEST-01** 多输入场景用表驱动测试 + `t.Run` 子测试；用例含 `name` 字段；错误信息带上输入与期望值。
```go
tests := []struct {
    name    string
    input   string
    want    int64
    wantErr bool
}{
    {name: "有效的数字 ID", input: "12345", want: 12345},
    {name: "非数字字符串", input: "abc", wantErr: true},
}
for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
        got, err := ParseUserID(tt.input)
        if (err != nil) != tt.wantErr {
            t.Errorf("ParseUserID(%q) error = %v, wantErr %v", tt.input, err, tt.wantErr)
            return
        }
        if got != tt.want {
            t.Errorf("ParseUserID(%q) = %v, want %v", tt.input, got, tt.want)
        }
    })
}
```

**TEST-02** 需测私有方法 → 同包 `package user`；只测公共 API → 独立包 `package user_test`。

**TEST-03** 测试辅助函数首行必须 `t.Helper()`，使失败信息指向调用位置。

**TEST-04** 测试中初始化失败用 `t.Fatalf`，禁止 `panic`；清理用 `t.Cleanup` 而非 `defer`。
```go
db, err := sql.Open("sqlite3", ":memory:")
if err != nil {
    t.Fatalf("打开测试数据库失败: %v", err)
}
t.Cleanup(func() { db.Close() })
```

---

## 11. 性能（PERF）

**PERF-01** 避免不必要的 `string` ↔ `[]byte` 转换（每次转换都复制一份内存）。首选让函数直接接受 `string`；确实需要 `[]byte` 且循环内用的是同一份数据时，才把转换提到循环外。
```go
// ❌ for _, s := range list { process([]byte(s)) }                  —— 循环内逐个转换
// ✅ for _, s := range list { process(s) }                          —— 改 process 接受 string，从根上不转换
// ✅ b := []byte(payload); for i := 0; i < retries; i++ { send(b) } —— 同一份数据，转换提到循环外
```

**PERF-02** 单个数字转字符串用 `strconv`，不用 `fmt.Sprintf`（fmt 有反射开销）。`[补充]` 判定只看格式串里的占位符个数：**1 个 → 用 `strconv`；2 个及以上 → 用 `fmt` 合理**。带字面量前后缀不改变判定，字面量用 `+` 拼接（源文档只给了 `fmt.Sprintf("%d", n)` 这一个反例，未给判定标准）。
```go
// ❌ fmt.Sprintf("%d", n)
// ✅ strconv.Itoa(n) / strconv.FormatInt(n, 10) / strconv.FormatFloat(f, 'f', 2, 64)
// ❌ fmt.Sprintf("user_%d", id)      —— 仍是单个占位符，前缀不改变判定
// ✅ "user_" + strconv.Itoa(id)
// ✅ fmt.Sprintf("(%d,%d)", x, y)    —— 两个占位符，不属本条约束
```

---

## 12. 项目结构（LAYOUT）

**LAYOUT-01** 新增文件按职责归位：`cmd/<name>/` 放可执行入口，每个目录一个二进制；`internal/` 与 `pkg/` 的划分见 PKG-02。

**LAYOUT-02** `main` 只负责启动，业务流程放 `run() error`；`os.Exit`、`log.Fatal` 只允许出现在 `main` 中。
```go
func main() {
    if err := run(context.Background()); err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
}
// run(ctx) 内部：加载配置 → 连接依赖（defer 关闭）→ 启动 server，每步错误都以 %w 包装后 return
```
