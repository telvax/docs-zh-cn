> 译自 [telvax/telvax@7f400a6](https://github.com/telvax/telvax/commit/7f400a605113ccec612c6418625d8607023adeef)（`docs/guide/tour.md`）。

# Telvax 导览

懂 Python 的人读 Telvax 会很轻松：代码块靠缩进划分，块以 `:` 开头，没有花括号，也没有分号。它有自己的一套关键字（`fn`、`let` / `var`、
`type`、`x => x + 1`、`Int?`），并且在程序运行之前就检查出 Python 要到运行中途才会发现的问题：
每个名字都有类型，可以显式写出，也可以由编译器推断。

本指南中的代码都经过测试：每个没有标记 `ignore` 的代码块都会被检查、格式化，
如果含有 `fn main`，还会被运行。请在仓库根目录下运行它们。

## 函数、类型与类型推断

```telvax
type Point:
    x: Float
    y: Float

fn norm(p: Point) -> Float:
    p.x * p.x + p.y * p.y

fn main() -> i32:
    let p = Point(x: 3.0, y: 4.0)
    var total = 0  # an untyped literal: its type comes from the uses
    for i in range(10):
        total += i
    print(f"norm = {norm(p)}, total = {total}")
    total
```

* 函数外部的 `let` 可以用大写字母书写（`MAX_TEXTS`），它的方法和字段与其他名字的用法相同。
* `let x = 1` 创建一个不能改变的名字；`var x = 1` 创建一个可以改变的名字（`x += 1`）。
  `let x: Int = 1` 把类型写出来。函数的最后一个表达式就是它的值，所以
  只有提前退出时才需要 `return`。
* 类型以大写字母开头，与你自己定义的类型一样：`Int`（64 位）、`Float`
  （64 位）、`Bool`、`Text`、`List[T]`、`Map[K, V]`、`T?`（一个值或 `None`）、`Result[T, E]`、
  `Tensor[T, dims]` 和 `Table[Row]`（表的列也可以是 `Date` 和 `Timestamp`）。需要指定位宽时：`i8`、`i16`、`i32`、`u8` 到 `u64`、
  `f32`、`f64`。
* 没有 `->` 的函数返回其最后一个表达式的类型：`fn double(x: Int): x * 2` 返回
  一个 `Int`。方法也是如此（`fn area(self): self.r * self.r`），实现某个
  trait 的方法返回该 trait 为它声明的类型。`main`、测试、endpoint 和泛型
  函数要写 `->`，函数带有错误时也要写。
* 以 `_` 开头的名字是文件私有的；其余的都可以被导入。
* 控制流：`if` / `elif` / `else`、`while`、`for x in items`、`for i in range(n)`、
  `for i in range(a, b)`、`break`、`continue`、`pass` 和 `return`。`a if cond else b` 是一个
  表达式。`and`、`or`、`not`、`in` 和 `not in` 是运算符。
* 注释以 `#` 开头。字符串用 `"` 或 `'`，`f"..."` 用来插值 `{expr}`。

## 命名参数与默认值

调用时可以给参数命名，参数或字段可以有默认值，这样配置
读起来像一句话，只需要常见情形的调用也保持简短：

```telvax
type Query:
    text: Text
    top_k: Int = 5

fn search(q: Query, limit: Int = 10, rerank: Bool = False) -> List[Text]:
    []

let q = Query(text: "tensor shapes")  # top_k is 5
let hits = search(q, rerank: True)  # limit is 10
```

名字会对照函数进行检查：拼错或重复的名字是编译错误。

## 管道

`x |> f(a)` 等价于 `f(x, a)`，所以各个阶段从左到右阅读。以 `|>` 开头的行延续
上一行：

```telvax ignore
let answer = prompt |> llm.complete() |> std.text.trim()

let docs = query
    |> embed()
    |> store.search(3)
```

规则只有一条：`x |> stage` 就是以 `x` 作为第一个参数调用 `stage`，此后它就是一次
普通的调用。

* 阶段可以是函数（`x |> f`、`x |> std.text.trim`）、一次调用（`x |> f(a)`）或一个闭包
  （`x |> (n => n + 1)`）。
* 名字和默认值把管道提供的值也算在内：对于 `fn scale(x: Int, k: Int = 2)`，
  `1 |> scale` 和 `1 |> scale(k: 3)` 都可以。
* 阶段后面的后缀作用于调用的结果：`s |> parse()?` 就是 `parse(s)?`，
  `n |> make().field` 就是 `make(n).field`，`n |> items()[0]` 就是 `items(n)[0]`。
* `|>` 的优先级低于所有运算符：`a + b |> f` 是把和送入管道，而 `x |> f == 1` 是送入
  `f == 1`，这是一个错误。请写成 `(x |> f()) == 1`。

## 张量与表

```telvax
fn forward(x: Tensor[f32, 2, 3], w: Tensor[f32, 3, 2]) -> Tensor[f32, 2, 2]:
    x |> std.tensor.matmul(w) |> std.tensor.relu()  # [2,3] x [3,2] -> [2,2], checked

type Row:
    id: Int
    score: f32

fn best(t: Table[Row]) -> Table[Row]:
    t |> std.table.sql("SELECT * FROM t WHERE score > 0.5 ORDER BY score DESC")

fn mean_score(t: Table[Row]) -> f32:
    std.tensor.mean(t.score)  # a zero-copy column
```

`std.table.read_csv(path)`、`read_parquet` 和 `read_ipc` 需要写出结果的类型
（`let t: Table[Row] = ...`），`std.tensor.zeros()`、`from_vec` 和 `reshape` 也一样。

表有一些保留行类型的方法。它们不需要 SQL，所以在 `telvax-lite` 中也能使用：

```telvax
type Row:
    id: Int
    score: f32

fn top_two(t: Table[Row]) -> Table[Row]:
    t.sort_by_desc("score").head(2)

fn passed(t: Table[Row]) -> Table[Row]:
    t.filter(r => r.score > 0.5)
```

`head(n)` 和 `tail(n)` 取前 `n` 行或后 `n` 行，`sort_by(column)` 和 `sort_by_desc(column)` 按某一
列排序，`filter(keep)` 保留 `keep` 为真的行，`a.concat(b)` 追加 `b` 的行
（两张表需要相同的行类型）。不存在的列名是一个运行时错误，错误信息会列出所有的列。

`t.select(["name", "id"])` 只保留这些列，并按这个顺序排列。列名会在程序被
检查时验证，结果的行类型由它们推算出来，所以不需要任何标注：

```telvax
type Person:
    id: Int
    name: Text
    age: Int

fn names(t: Table[Person]) -> List[Text]:
    var out: List[Text] = []
    for r in t.select(["name", "id"]):
        out.push(f"{r.id}: {r.name}")
    out
```

列名必须直接写成列表里的文本字面量（不能是变量），不存在的列，或者
写了两次的列，都是编译错误。

`t.group_by(keys, aggregations)` 为键列的每个不同取值生成一行，顺序为首次出现的顺序。
每个聚合写成 `column:function`，并成为名为 `<column>_<function>` 的列：

```telvax
type Sale:
    region: Text
    units: Int
    price: f64

fn report(t: Table[Sale]) -> List[Text]:
    var out: List[Text] = []
    for r in t.group_by(["region"], ["units:sum", "price:mean"]).sort_by_desc("units_sum"):
        out.push(f"{r.region}: {r.units_sum} units, {r.price_mean} on average")
    out
```

可用的函数有 `sum`（整数列得到 `Int`，小数列得到 `Float`）、`mean`（`Float`）、`min` 和
`max`（与列相同的类型）以及 `count`（`Int`，适用于任何列）。结果的行类型由
名字推算，与 `select` 相同。未知的列或函数、类型为 `T?` 的列、对文本求和或求平均，以及结果中
列名重复，都是编译错误。同样的查询用 SQL 仍然可行；`group_by` 在 `telvax-lite` 中也能运行。

类型为 `T?` 的列可以包含空值：`temp: f64?` 会把 CSV 中的空单元格、Parquet 的空值以及
`from_rows` 中的 `None` 都读作 `None`，此时 `t.temp` 是一个 `List[f64?]`（不是张量）。没有 `?` 的列
如果遇到空值，会报错，指出列名并提示写上 `?`。

```telvax
type Reading:
    id: Int
    temp: f64?

fn warm(t: Table[Reading]) -> Table[Reading]:
    t.filter(r => r.temp.unwrap_or(0.0) > 20.0)
```

`Date` 和 `Timestamp` 是表示日期和时间的列类型。在表中它们是 Arrow 的 `Date32` 和
`Timestamp(µs, UTC)`，所以 Parquet、SQL 和 PyArrow 看到的是真正的日期。在程序中，值是 ISO-8601 格式的 `Text`：
`"2026-10-17"` 和 `"2026-10-17T16:53:03Z"`。`from_rows` 和 CSV 接受任何偏移量（`+02:00`）并按 UTC 存储；不是日期的
文本会报错，提示写成 `YYYY-MM-DD`。目前还没有日期运算：可以比较
文本、对表排序，或者使用 SQL。

```telvax
type Event:
    id: Int
    day: Date
    at: Timestamp?

fn latest(t: Table[Event]) -> Table[Event]:
    t.sort_by_desc("day").head(1)
```

`std.table.read_ipc(path)` 和 `write_ipc(table, path)` 读写 Arrow IPC 文件（Feather v2），PyArrow、
Polars 和 pandas 也能读写这种文件。`read_ipc` 把文件映射到内存，列直接指向其中，所以只有一个批次的
文件不会被复制。在使用从某个文件读出的表期间，不要重写或截断该文件。用 LZ4 或 Zstd
压缩的文件在完整版 `telvax` 中会被解码（需要复制）；`telvax-lite` 只读取未压缩的文件。

表可以切成块。`t.split(n)` 给出一个 `List[Table[Row]]`，其中每张表最多 `n` 行；
它们共享 `t` 的缓冲区，所以不会复制任何数据。`std.table.read_csv_batches(path, n)`、
`read_parquet_batches` 和 `read_ipc_batches` 直接把文件读成这样的块（不会构建整个
文件的合并副本），并且需要写出结果的类型：`let parts: List[Table[Row]] = ...`。这些块同时都在
内存中；目前还没有一次只交出一块的读取器。

```telvax
type Row:
    id: Int
    score: f32

fn total_score(path: Text) -> f32:
    let parts: List[Table[Row]] = std.table.read_parquet_batches(path, 100000)
    var sum = 0.0
    for part in parts:
        sum += part.score.sum()
    sum
```

`std.table.read_flight(url, ticket)` 从 Arrow Flight 服务器获取一张表（`DoGet`），
`std.table.write_flight(table, url, path)` 发送一张表（`DoPut`）。地址是 `grpc://host:port`，或者
使用 TLS 时是 `grpc+tls://host:port`。两者都需要 `net` 权限，并且需要启用 `flight` 特性的构建
（`cargo install telvax --features flight`）；已发布的二进制文件不包含该特性，没有它时会
给出提示。这里只有客户端，没有 Flight 服务器。

## HTTP endpoint

```telvax ignore
@endpoint(POST, "/v1/score")
fn score(input: In) -> Result[Out, Status]:
    ...
```

参数的类型就是经过验证的 JSON 请求体（GET/DELETE 从查询字符串中读取）。
返回 `Err(Status.BadRequest("..."))` 会变成 HTTP 400。

带有以字面量写出的默认值（`language: Text = "en"`）或类型为 `T?` 的字段，可以在请求体中
省略；它会取默认值，或者 `None`。其他字段都是必填的。

路径中的 `{name}` 片段会从 URL 读入同名的参数：

```telvax ignore
@endpoint(GET, "/v1/results/{job_id}")
fn result(job_id: Text) -> Result[Job, Status]:
    ...

@endpoint(PUT, "/v1/jobs/{id}/steps/{step}")
fn update(id: Int, step: Text, input: Patch) -> Result[Job, Status]:
    ...
```

路径参数可以是 `Text`、数值类型或 `Bool`。无法转换的值会得到 HTTP 400：
`invalid path parameter id: expected i64, found `x``。不在路径中的参数就是
payload，最多只能有一个。`/a/{x}` 和 `/a/{y}` 是同一条路由。

类型为 `Request` 的参数接收请求。它不属于 payload，最多只能有一个：

```telvax ignore
import std.http.Request

@endpoint(DELETE, "/v1/cache")
fn clear(req: Request) -> Result[Cleared, Status]:
    if req.header("x-admin-token") != Some(token):
        return Err(Status.Unauthorized("invalid token"))
    ...
```

`req.header(name)` 是该请求头的第一个值，或者 `None`；名字不区分大小写。
`req.method()` 和 `req.path()` 是 `Text`。

存在于函数之外的值由所有请求共享。保存 `List` 或 `Map` 的顶层 `let`
可以在 endpoint 中被修改，每次 `push`、`insert` 或 `remove` 都是原子的，所以缓存或
任务表不需要其他东西：

```telvax ignore
let cache: Map[Text, Scores] = {}
let stored_at: Map[Text, Float] = {}

fn lookup(text: Text) -> Scores?:
    match stored_at.get(text):
        case Some(t):
            if std.time.now() - t < 300.0:
                cache.get(text)
            else:
                None
        case None:
            None
```

不允许使用顶层 `var`，所以计数器是只有一个键的 `Map`。对同一个值连续两次调用
合起来并不是原子的：另一个请求可能在 `get` 和 `insert` 之间插进来。没有大小限制，也没有过期；请自己
删除旧的键。

带上限和过期时间的缓存是 `std.cache.new(max, ttl_seconds)`。缓存满时它会丢弃最近最少使用的条目，
在条目写入 `ttl_seconds` 之后遗忘它，并且像上面的 `Map` 一样由所有请求共享，所以
`stored_at` 表和手动删除都不再需要：

```telvax ignore
let cache = std.cache.new(10000, 300.0)

fn lookup(text: Text) -> Scores?:
    cache.get(text)

fn remember(text: Text, scores: Scores):
    cache.set(text, scores)
```

方法有 `get`、`set`、`remove(key)`、`clear()`（返回清除前的条目数）和 `len()`。键是 `Text`。值目前不做类型检查：
`get` 给出的是所存内容的 option。上限按分片（shard）维护（容量为 10,000 条的缓存有 16 个分片），所以
被淘汰的是它所在分片中最近最少使用的条目。

endpoint 默认以 `200` 和 JSON 形式的值作答。若要选择状态码或添加请求头，请返回 `Response`：

```telvax ignore
import std.http.Response

@endpoint(POST, "/v1/analyze_batch")
fn submit(input: Batch) -> Result[Response, Status]:
    let job = start(input)
    Ok(Response.new(Accepted(job_id: job.id))
        .status(202)
        .header("location", "/v1/results/" + job.id))
```

`Response.new(body)` 是 `200`；`.status(code)` 接受 100 到 599 之间的数字（其他值会得到 HTTP 500），
`.header(name, value)` 设置一个请求头，并替换同名的头，包括 `content-type`。
`Err(Status...)` 仍然用来给出错误状态码。

### 响应之后的工作

`std.task.spawn(task)` 启动一个 `async fn` 并立即返回；任务在 endpoint 作答之后继续运行，所以
服务可以先回复 `202`，稍后再完成，而客户端则轮询：

```telvax ignore
let jobs: Map[Text, Text] = {}

async fn work(id: Text, texts: List[Text]):
    # ... the slow part ...
    jobs.insert(id, "done")

@endpoint(POST, "/v1/jobs")
fn start(input: Batch) -> Result[Response, Status]:
    let id = new_id()
    jobs.insert(id, "running")
    std.task.spawn(work(id, input.texts))
    Ok(Response.new(Accepted(job_id: id)).status(202))
```

* 任务的值会被丢弃。其中的错误会作为 `background task failed: ...` 写到标准错误，
  除非第二个参数指明如何处理它：`std.task.spawn(work(id), failed)` 会调用 `failed(message)`，这是一个
  接收一个 `Text` 的函数，这样任务就可以被标记为失败，而不是一直停在 `running`。需要报告
  结果的任务把结果写到程序读取的地方，就像上面的 `jobs`。
* 同时最多运行 256 个任务；再多一个，启动它的 endpoint 就会报错。
* 任务共享程序的顶层 `List` 和 `Map`（见上文关于共享状态的说明）。它随
  进程结束：停止服务器就会停止任务。
* 只有服务器会让任务继续运行。在 `telvax run` 和 `telvax test` 中，`spawn` 会等待任务完成，与 `std.task.run` 相同。

### Server-sent events

`Response.events(producer)` 以 `text/event-stream` 作答。客户端读取时，服务器用一个 `Sink` 调用 `producer`，
每个 `sink.send(...)` 就是一个事件，立即发出：

```telvax ignore
import std.http.Response
import std.http.Sink

@endpoint(POST, "/v1/analyze_stream")
fn stream(input: Batch) -> Result[Response, Status]:
    fn produce(sink: Sink):
        for text in input.texts:
            sink.send(score(text))
        sink.event("done", "{}")
    Ok(Response.events(produce))
```

* `send(data)` 接受一个 `Text`（原样发送），或任何其他值（以 JSON 发送）。`event(name, data)` 添加一行
  `event:`，`comment(text)` 写入一行客户端会忽略的内容，用来保持连接。
* 生产者是写在 endpoint 内部的函数（闭包只有一行），所以它可以使用 endpoint 的参数。
* 状态码和请求头会先发出，所以生产者中的错误无法改变它们：流会以
  `event: error` 和错误消息结束。客户端断开连接会让下一次 `send` 失败，从而结束生产者。
* `telvax serve` 和 `build --serve` 支持流式输出；原生后端不提供 HTTP 服务。

## PostgreSQL 上的 SQL

`std.db.postgres.PostgresClient` 在 PostgreSQL 服务器上运行 SQL。`execute` 给出语句
所改变的行数，`query` 把行作为你所写的某个 `type` 的列表返回：

```telvax ignore
import std.db.postgres.PostgresClient

type Job:
    id: Text
    status: Text
    attempts: Int
    result: Text?

let db = PostgresClient.connect(std.env.get("DATABASE_URL").unwrap())

fn finish(id: Text, result: Text) -> Result[Int, Text]:
    db.execute("UPDATE jobs SET status = $1, result = $2 WHERE id = $3", ("done", result, id))

fn queued() -> Result[List[Job], Text]:
    let rows: List[Job] = db.query("SELECT id, status, attempts, result FROM jobs WHERE status = $1", ["queued"])?
    Ok(rows)
```

* `$1`、`$2` 的值与 SQL 分开传入：类型相同时用列表，类型不同时用
  元组。它们由服务器绑定，所以值无法改变语句本身。`None` 就是 `NULL`。
* 字段从同名的列读取，`Text?` 允许 `NULL`。不是字段的列会被
  忽略；没有对应列的字段是错误。`json` 和 `jsonb` 列是 `Text`。`timestamptz`、`numeric` 或
  `uuid` 列会报错，提示在 SQL 中进行转换（`created_at::text`）。
* 服务器的错误是带有其消息的 `Err(Text)`，例如失败的约束的名字。
* 连接由第一条语句打开并保持。每个客户端一个连接，一次执行一条语句，
  目前还没有事务。
* 需要 `--live` 和 `net` 权限：SQL 没有离线替代品。

### 指标与日志

`telvax serve app.tvx --metrics --log-format json` 提供 `GET /metrics`（按路由统计请求和延迟），并为每个请求写一行
带有请求 id 的 JSON。程序用 `std.metrics.count` 统计只有它自己知道的事情，服务器
会以它自己的名字列出，比如 `cache_hit`：

```telvax ignore
fn lookup(key: Text) -> Text?:
    match cache.get(key):
        case Some(v):
            std.metrics.count("cache_hit")
            Some(v)
        case None:
            std.metrics.count("cache_miss")
            None
```

`--cors "*"`、`--rate-limit 100/min` 和 `--security-headers` 在每个响应外围加上常用的保护，
无需反向代理。详情见[命令行参考](https://github.com/telvax/telvax/blob/main/docs/reference/cli.md)。

计数器可以带标签，标签是一个 `Map[Text, Text]`：`std.metrics.count("jobs_total", {"status": "done"})` 会被列为
`jobs_total{status="done"}`，`by` 是第三个参数。名字由字母、数字、`_` 和 `:` 组成，一个计数器最多有
1,000 组标签（所以标签不应该由用户发送的内容生成），`--process-time-header` 会给每个响应添加
`x-process-time-ms`。
`--error-format detail` 把服务器产生的错误写成 `{"detail": ...}`，把不符合其类型的请求体写成 `422`，
与 FastAPI 一致；默认是 `{"error": ...}` 和 `400`。程序用 `Response` 构造的响应永远不会被改动。
运行时失败的函数会以 `500` 和 `Internal server error` 作答；原因连同请求 id 写入日志。

程序可以用 `std.log.debug`、`info`、`warn` 和 `error` 写自己的日志行，格式由 `--log-format` 决定，
级别由 `--log-level` 决定，还可以带可选字段：`std.log.warn("slow query", {"ms": "840"})`。请求日志行也带有 `level`
（400 以下为 `info`，4xx 为 `warn`，5xx 为 `error`），所以 `--log-level warn` 在两者中都只保留失败的内容。

服务器的选项可以写在程序里，而不是命令行上，这样一个文件就说明了服务的
行为：

```
serve:
    metrics: True
    cors: "*"
    rate_limit: "100/min"
    security_headers: True
    error_format: "detail"
```

值是固定的，或者在服务器启动时从环境中读取：`cors: env("CORS_ORIGINS", "*")` 是变量
`CORS_ORIGINS`，未设置或为空时则是 `*`（默认值是可选的；没有默认值时，这一行等同于不存在）。
变量的文本按命令行标志的方式读取，所以 `log_format: env("LOG_FORMAT", "json")` 接受 `json` 或 `text`，
`metrics: env("METRICS")` 在值为 `1` 或 `true` 时开启。`TELVAX_*` 变量或命令行标志优先于该块，而 `--port`、`--host` 以及其他
关于程序在哪里运行的选项仍然只是命令行标志。完整列表见[命令行参考](https://github.com/telvax/telvax/blob/main/docs/reference/cli.md)。

你自己的中间件包裹每一个请求。标有 `@middleware` 的函数接收请求和 `next`（链中的其余部分），
并返回一个 `Response`；块中的 `middleware:` 一行列出它们，最外层的在最前面：

```telvax ignore
import std.http.Next
import std.http.Request
import std.http.Response

@middleware
fn version_header(req: Request, next: Next) -> Response:
    next(req).header("x-api-version", "1")

@middleware
fn admin_only(req: Request, next: Next) -> Response:
    if req.path() == "/admin" and req.header("x-admin-token") != Some(admin_token):
        return Response.new(Denied(detail: "Invalid admin token")).status(401)
    next(req)

serve:
    middleware: [admin_only, version_header]
```

`next(req)` 先运行下一个中间件，再运行路由，并返回它们的 `Response`，你可以修改其状态码和请求头。
不调用 `next` 的中间件自己作答，路由不会运行。内置的中间件（`metrics`、`cors`、
`rate_limit` 等）位于外层，并保留各自的选项。中间件在解释器中对每个请求运行，包括 `/health` 和
`/metrics`（用 `req.path()` 判断即可把某条路由排除在外），并且它不能读取或修改请求体。

块中唯一不是值的一行是 `on_start`，它是一个函数列表，在服务器开始
监听之前按顺序运行一次：建表、预热缓存、检查某项设置。函数不接收参数，不返回任何东西，或者返回
`Result[(), Text]`；`Err` 会终止启动，其消息写到标准错误，退出状态为 1。

```telvax ignore
fn init_db():
    db.execute("CREATE TABLE IF NOT EXISTS jobs (id TEXT PRIMARY KEY)", [])

serve:
    on_start: [init_db]
```

### API 文档

endpoint 的类型已经描述了 API，所以服务器会把它发布出来。`GET /openapi.json` 是一份 OpenAPI 3.0
文档：包含每条路由及其路径参数、查询参数或 JSON 请求体、响应的类型，以及
错误响应（`{"error": "..."}`）。`GET /docs` 是列出这些内容的页面。每个服务器都有这两个路径，如果
程序在其中某个路径上有自己的 endpoint，则提供程序自己的那个。`GET /healthz` 返回 `ok`，服务器正在停止时返回 `503`。

服务器收到 `SIGTERM` 或 `Ctrl-C` 时，会停止接受连接，完成手头的请求，并等待用
`std.task.spawn` 启动的任务，最多等待 `--shutdown-wait N` 秒（默认 30，或 `TELVAX_SHUTDOWN_WAIT`）。第二次
信号会立即退出。如果有工作没有完成，退出状态为 1。

返回 `Response` 的函数没有类型化的响应，所以它的 `200` 没有 schema。类型为 `T?` 或
`Date` 的字段按其被检查的方式描述，不会超出它的 JSON 类型。

## 模块与包

```telvax ignore
from util import clamp  # ./util.tvx
from mathx.vec import dot  # dependency `mathx`, file vec.tvx
```

顶层 `let` 的导入方式相同（`from settings import LIMIT`）；它在整个程序中是同一个值，所以在一个模块中
改变的 `List` 或 `Map` 在所有模块中都会改变。`import util.clamp` 引入同一个名字。名字可以用 `as` 重命名：
`from util import clamp as limit`、`import util.clamp as limit`、`from std.text import trim as strip`。
`std` 中的名字不需要导入：`std.text.trim(s)`。模块不是名字：`import util` 是一个错误，
会提示你写 `from util import name`。

```sh
tvpm new app && cd app
tvpm add mathx --registry ~/registry     # or --path ../mathx
tvpm install --registry ~/registry       # resolves, writes telvax.lock, downloads
tvpm run                                  # runs src/main.tvx with the dependencies
tvpm update                               # newest versions that fit; `tvpm update mathx` moves only mathx
```

## 分布式

```telvax ignore
fn work(item: Job) -> Done:
    ...

fn run_all(jobs: List[Job]) -> List[Done]:
    std.dist.map(jobs, work)
```

`std.dist.map` 接收顶层函数的*名字*。默认在本地线程上运行；
`telvax run app.tvx run_all --workers host1:7077,host2:7077` 把调用发送给运行着
`telvax worker` 的机器，`telvax.ray.RayExecutor`（Python）则把它们发送给 Ray。

## 量化

`--quantize int8`（用于 `run`、`serve`、`jit`、`build`）让 `f32` 矩阵乘法使用对称的
逐张量 int8 运算，并以 `i32` 累加。解释器和原生后端给出
相同的结果。

`--quantize int8-channel` 和 `--quantize int4-channel`（`run` 和 `serve`；原生后端
不支持）为激活值的每一行和权重的每一列提供各自的缩放比例，这样
远大于或远小于其余部分的列就不会再损失精度。int4 模式
以 4 位存储权重。缩放比例在每次调用时计算；没有校准步骤。

## 加速器

* `--device gpu`（用于 `run` 和 `serve`，需要 `cargo build --features gpu`）通过 `wgpu` 在 GPU 上运行 `f32` 张量运算：macOS 上是 Metal，Linux 上是 Vulkan，Windows 上是 DX12。
* `telvax build file.tvx --emit stablehlo` 为文件中的张量函数输出 StableHLO，用于 XLA / PJRT。
* `telvax build file.tvx --emit ptx [--sm sm_80]` 为文件使用的张量运算输出 PTX 内核（需要 `--features llvm`）。

PTX 和 StableHLO 只是生成并检查，不会被执行：见 `docs/design/status.md`。

## 枚举、`match`、`break` 和 `continue`

```telvax
enum Shape:
    Circle(Float)
    Rect(Float, Float)
    Empty

fn area(s: Shape) -> Float:
    match s:
        case Shape.Circle(r):
            r * r * 3.14159
        case Shape.Rect(w, h):
            w * h
        case Shape.Empty:
            0.0

fn first_over(limit: i32) -> i32:
    var found = -1
    for i in range(1, 1000):
        if i * i > limit:
            found = i
            break
    found
```

`match` 必须覆盖所有情况（编译器会指出缺失的变体），它也适用于
`Result`（`Ok(v)` / `Err(e)`）、整数、字符串、布尔值和 `_`。枚举以 JSON 传输：没有值的
变体就是它的名字（`"Empty"`），带值的变体是 `{"Circle": 2.0}` 或 `{"Rect": [2.0, 3.0]}`。
枚举在解释器中运行；原生后端目前还不能编译它们。

## 限制程序可以做什么

程序可以读写文件、读取环境变量并访问网络。`run`、`test` 和 `serve` 上的 `--allow`
列出它可以做的事；不带该标志时，一切都被允许。

```sh
telvax run job.tvx main --allow fs.read          # may read files, nothing else
telvax test app.tvx --allow none                 # may touch nothing outside itself
telvax serve app.tvx --live --allow net,env
```

| 词 | 允许 |
|---|---|
| `fs.read` | `std.fs.read`、`std.fs.exists`、`read_csv`、`read_parquet`、`read_ipc` 及其 `_batches` 形式 |
| `fs.write` | `std.fs.write`、`write_csv`、`write_parquet`、`write_ipc` |
| `fs` | `fs.read` 和 `fs.write` |
| `env` | `std.env.get` |
| `net` | 使用 `--live` 时的 Ollama、OpenAI 和 PostgreSQL 客户端，以及 `read_flight` / `write_flight` |
| `all`、`none` | 全部、无 |

不被允许的调用会终止程序，并指出要添加哪个词：
`` `std.fs.read` needs the `fs.read` capability; run with `--allow fs.read` ``。这一限制是在
程序运行时检查的，而不是在编译时，并且 `std.io.print` 和 `std.time` 始终被允许。

### 二进制文件内部的文件

`std.fs.embed("path")` 是一个 `Text`，内容为某个文件的内容，在构建程序时读取并保存在程序内部。
路径是文本字面量，相对于主文件所在的目录，所以 `telvax build --serve` 生成的单个二进制文件
不需要旁边再放数据文件：

```telvax
let lexicon = std.fs.embed("lexicons/vader.txt")
```

它不需要任何权限，因为文件是在构建时读取的，而不是由运行中的程序读取。不存在的文件，
或者不是字面量的路径（`"data/" + name`），在检查程序时就会报错。

## 测试

名为 `test_...` 的函数是一个测试。`assert(cond)` 和 `assert_eq(a, b)` 在失败时
带着消息终止它：

```telvax
fn double(x: Int) -> Int:
    x * 2

fn test_double():
    assert_eq(double(2), 4)
```

```sh
telvax test file.tvx           # run them all, exit 1 if one fails
telvax test file.tvx double    # only the tests whose name contains `double`
telvax test tests/             # every `test_*.tvx` and `*_test.tvx` file under a directory
telvax test                    # the same, in this directory
```

### Fixture

带参数的测试从 fixture 获得这些参数。`fn test_total(order: Order)` 会调用
`fixture_order()` 并传入结果；fixture 可以用同样的方式接收其他 fixture。每个测试
都得到新的值。如果存在 `teardown_order(order: Order)` 函数，它会在测试之后运行，
无论测试通过与否。

```telvax
fn fixture_base() -> Int:
    10

fn fixture_total(base: Int) -> Int:
    base * 2

fn teardown_total(total: Int):
    print(f"done with {total}")

fn test_total(base: Int, total: Int):
    assert_eq(total, base * 2)
```

没有对应 `fixture_...` 函数的参数，或者返回了其他类型的 fixture，会让测试失败，
并说明该怎么写。

### 用例

若要用多组值运行同一个测试，请写一个 `cases_...` 函数，名字是测试名去掉
`test_`。它返回一个列表；测试对每一项各运行一次。元组给出测试的前几个参数，
剩下的参数仍然是 fixture。

```telvax ignore
fn cases_double() -> List[(Int, Int)]:
    [(1, 2), (2, 4), (3, 6)]

fn test_double(x: Int, want: Int):
    assert_eq(double(x), want)
```

每次运行会列为 `test_double[(2, 4)]`。为空的 `cases_...` 函数，或者其项与参数
不匹配的函数，会让测试失败并说明原因。

### Mock

`std.test.mock(target, replacement)` 让测试调用另一个函数来代替某个函数。目标是
你自己的某个函数（不加引号书写），或者以文本形式给出路径的 std 函数。函数目标
必须与其替代者类型相同；std 目标在测试运行时检查。
`std.test.calls(target)` 给出替代者被调用的次数。每个 mock 都会在
测试结束时移除。

```telvax
fn price(item: Text) -> Int:
    100

fn total(item: Text) -> Int:
    price(item) * 2

fn fake_read(path: Text) -> Result[Text, Text]:
    Ok("from the test")

fn test_total():
    std.test.mock(price, item => 7)
    assert_eq(total("a"), 14)
    assert_eq(std.test.calls(price), 1)

fn test_read():
    std.test.mock("std.fs.read", fake_read)
    assert_eq(std.fs.read("any.txt"), Ok("from the test"))
```

替代者不能调用它所替代的函数。通过方法进行的调用（`x.len()`）不能被
mock，`std.test.*` 本身也不能。

### 覆盖率

`--coverage` 为项目的每个文件列出含代码的行数、测试运行了其中多少行，
以及未运行的行号。`--fail-under 90` 还会在运行的行数不足 90% 时让本次运行失败，退出
码为 1，这样流水线就可以守住这条底线。

```sh
telvax test tests/ --coverage
telvax test tests/ --fail-under 90
```

```text
coverage (lines that ran / lines with code):
  src/price.tvx  18/20  90.0%   missing 14-15
  total  18/20  90.0%
```

含有语句、块的最后一个表达式或 `case` 的主体的行才计入。`test_`、`fixture_`、`teardown_` 和 `cases_`
函数不计入，项目之外的文件（含有 `telvax.toml` 的目录，或你所测试的目录）
也不计入。统计的是行，而不是
分支：一行中写了 `if` 及其两个结果，只算一次。

## 宏

宏写作 `name!(...)`。编译器有五个宏，你不能再定义更多。每个宏在程序被检查之前
都会被改写为对某个 `std` 函数的调用。

| 宏 | 作用 |
|---|---|
| `assert!(cond)` | 与 `assert` 相同，消息会引用该条件：`assertion failed: x > 3` |
| `assert!(cond, "message")` | `cond` 为假时，带着你的消息终止 |
| `dbg!(value)` | 把 `value_as_written = result` 打印到 stderr，并返回 `value` |
| `todo!()` | 以 `not yet implemented` 终止；可以出现在期望任何类型的位置 |
| `unreachable!()` | 以 `unreachable code was reached` 终止 |
| `panic!("message")` | 带着该消息终止 |

```telvax
fn pick(n: Int) -> Int:
    if n > 0:
        n
    else:
        todo!()

fn test_pick():
    let x = dbg!(pick(4))  # stderr: pick(4) = 4
    assert!(x == 4)
```

宏与 `assert` 一样在解释器中运行；原生后端不编译它们。

## REPL

```sh
telvax repl                    # an interactive session
telvax repl app.tvx            # start with the definitions and variables of a file
```

```text
>>> let xs = [3, 1, 2]
>>> xs.len()
3
>>> fn total(v: List[Int]) -> Int:
...     v.sum()
...
>>> total(xs)
6
>>> var n = 0
>>> for x in xs:
...     n = n + x
...
>>> n
6
>>> :type xs
List[Int]
```

你可以输入定义（`fn`、`type`、`enum`、`trait`、`extend`、`import`）、变量（`let`、`var`）、
语句或表达式；表达式的值会显示出来。块以空行结束。名字一直有效，
直到 `:reset`：名字可以重新定义（`let x = x + 1`，或者同名的新 `fn`），而 `let`
不能被赋值，与在文件中一样。每次输入都会连同之前定义的所有内容一起检查，所以类型错误
会显示出那一行，并让会话保持原样。单独写的数字（`let n = 4`）是 `Int`，这样
接收 `Int` 的函数就能接受它。闭包需要保留其参数的类型
（`let inc = (n: Int) => n + 1`）。

命令以冒号开头：`:type <expression>`、`:vars`、`:defs`、`:load <file>`、`:reset`、`:help` 和
`:quit`。Tab 补全名字、字段和方法，方向键可以浏览之前的输入（保存在
`~/.telvax_history`；`TELVAX_HISTORY` 可指定另一个文件）。`--allow` 限制会话可以做什么，与
`telvax run` 相同。每次输入只运行一次：它打印或写入的内容只发生一次，并且输入下一条时不会重新计算
之前的值。

## 调试

```sh
telvax dap       # a debug adapter on standard input and output
```

在 VS Code 中，对 Telvax 文件按 F5 即可调试（见扩展的 README）。`telvax dap` 使用[调试适配器协议](https://microsoft.github.io/debug-adapter-protocol/)，所以任何
带有该协议客户端的编辑器都可以在解释器中运行程序，在断点处停下，并逐步执行。
它的 launch 请求接受 `program`（`.tvx` 文件）、`function`（默认 `main`）、`args`（你会交给 `telvax run` 的
那些词）、`stopOnEntry`、`noDebug` 和 `allow`（与 `--allow` 相同的列表）。它在每条语句
之前停下，所以断点应该设在语句开头的那一行；设在其他行时，它会移到
下一条语句开头的行。停下时，它会列出正在进行的调用及其变量，并且可以对其中某个名字或
字段（`p.x`、`xs.0`）求值；其他表达式则不行。它调试的是解释器，而不是
`telvax build` 生成的程序，并且没有条件断点。

## 性能分析

```sh
telvax profile app.tvx                       # runs `main` and reports where the time and memory went
telvax profile app.tvx handler 3 "text"      # another function, with its arguments
telvax profile app.tvx --top 30 --json r.json --collapsed stacks.txt --speedscope p.json
```

报告分四个部分：总耗时和内存（峰值、分配的字节数以及
分配次数）；函数，包含调用次数、函数自身的耗时（`self`）和包含它所
发起调用的耗时（`total`），以及它分配的字节数；语句，包含每条语句运行的次数，以及它花费的时间，
直到同一次调用中的下一条语句开始为止；以及 `std` 函数（`std.tensor.matmul`、
`std.table.sql` 等），包含调用次数、耗时、输入和输出的行数或元素个数，以及结果的
字节数。`--json` 把所有这些写入文件，`--collapsed` 写出供 `flamegraph.pl` 或
`inferno` 使用的调用栈，`--speedscope` 写出供 <https://www.speedscope.app> 使用的文件。

耗时是解释器围绕程序所做的事情测量的，所以它们显示的是*程序*把时间
花在了哪里，其中也包含解释器自身的时间。内存统计的是整个进程，所以在
多线程时，一个函数的统计也会包含其他线程在这期间分配的内存。用
`telvax build` 构建的程序不会被分析。

## 文档

`telvax schema data.csv` 为 CSV、Parquet 或 Arrow 文件的各列输出 `type`，这样你就不用
手写了：含有空值的列会写成 `T?`，`--name Row` 用来指定类型的名字。

`telvax doc` 根据代码写出 Markdown 参考文档：每个函数及其签名，每个 `type` 及
其字段表（和默认值），每个 `enum` 及其变体，每个 `trait`，以及 `extend` 下的方法。
紧挨在某一项上方的 `#` 注释行（中间没有空行）会成为它的描述。

```sh
telvax doc file.tvx                # print the reference of one file
telvax doc src/ -o REFERENCE.md    # every `.tvx` file under the directory, written to a file
telvax doc --public src/           # leave out names that start with `_`
```

```tvx
# Sum of the first `n` squares.
fn squares(n: Int) -> Int:
    n * (n + 1) * (2 * n + 1) / 6
```

## 格式化与 lint

```sh
telvax fmt file.tvx            # rewrite in place
telvax fmt src/ tests/         # every `.tvx` file under the directories (no path: this directory)
telvax fmt --check             # list files that would change (exit 1), for CI
telvax fmt --diff src/         # show the lines that would change, write nothing (exit 1)
telvax lint file.tvx           # unused variables, unused private functions, unreachable cases
telvax lint --list-rules       # the rules and what they find
```

每条 lint 规则都有名字（`unused-variable`、`unused-function`、`unreachable-case`）和级别：
`allow`（什么都不说）、`warn`（默认）或 `deny`（错误，所以 `telvax lint` 会失败）。在 `telvax.toml` 中为
一个包设置级别，或者在一次运行中用 `--allow rule`、`--warn rule` 和
`--deny rule` 设置；命令行标志优先于文件。`--deny-warnings` 会让任何警告都使本次运行失败。

```toml
[lint]
unused-variable = "deny"
unused-function = "allow"
```

缩进是语法的一部分，所以 `telvax fmt` 绝不会在块之间移动某一行：它把每一级设为四个
空格，并整理行内的空格、续行和空行。遍历目录时会跳过隐藏
文件夹、`target`、`node_modules` 和 `.tvpm`。

语言服务器会在你输入时补全名字：`:` 和 `->` 之后的类型、点号之后的字段、列和
方法（也包括 `s.trim().` 这样的调用之后）、`Shape.` 之后的变体、你正在编写的闭包的
参数（`xs.map(p => p.`），以及 `t.sql("..` 里面的表的列。符合代码所期望
类型的内容排在最前面。

语言服务器也会格式化文档，所以 VS Code 可以在保存时格式化。它还会查找对
变量、函数、全局变量、类型、枚举变体、字段或方法的所有引用并重命名它们（当新名字会与作用域内的另一个名字
冲突、某个文件有语法错误，
或者该名字属于某个依赖时，重命名会被拒绝）。它们会顺着导入关系进入项目中的其他文件。

## 字符串与列表

```telvax
fn demo(s: Text) -> u64:
    let words = s.split(" ")  # List[Text]
    words.push("extra")
    if s.contains("telvax") and not s.starts_with("#"):
        return words.len()
    0
```

`Text`：`len`、`is_empty`、`trim`、`to_lowercase`、`to_uppercase`、`contains`、`starts_with`、
`ends_with`、`replace`、`split`。`List`：`len`、`is_empty`、`push`、`contains`。`len` 是 `u64`，与有符号数一起使用时（`xs.len() > limit`、`xs.len() - 1`），两者都按 `Int` 读取。`x in xs` 就是
`xs.contains(x)`。不存在的方法是编译错误，并带有“你是不是想写”的提示。
这些方法在解释器中运行；原生后端目前只有 `len` 和 `is_empty`。

## 随机值

`std.random.uuid()` 每次都是一个新的标识符，例如 `"3f2b8c1e9a4d4b7f8e6a0c5d1b2a3f4e"`，
`std.random.int(1, 6)` 是 1 到 6 之间的整数（两端都包含）。它们来自操作系统，所以这样的 id 适合
用来命名只应由其所有者读取结果的任务。它们不需要任何权限，也没有随机种子。

## 可选值、map 与小函数

```telvax
fn demo(text: Text) -> Int:
    let counts: Map[Text, Int] = std.map.new()
    for word in text.split(" "):
        let seen = counts.get(word).unwrap_or(0)  # Int?
        counts.insert(word, seen + 1)
    match counts.get("telvax"):
        case Some(n):
            n
        case None:
            0
```

* `T?` 是 `Some(value)` 或 `None`；`match` 必须同时覆盖两者。方法：`is_some`、`is_none`、
  `unwrap`（对 `None` 会报错）、`unwrap_or`、`unwrap_or_else(() => v)`、`expect("message")`、
  `map`、`and_then`、`filter` 和 `ok_or(error)`（把它变成 `Result`）。有几个方法会返回它：`List.first`、`last`、
  `pop`、`get`、`Text.find`、`to_int`、`to_float`、`Map.get`、`remove`。在 JSON 中，`None` 是
  `null`，`Some(x)` 就是 `x`。
* `Map[K, V]` 的键可以是 `Text`、整数或 `Bool`。它用 `std.map.new()` 和一个
  类型标注创建，有 `insert`、`get`、`remove`、`contains_key`、`len`、`is_empty`、`keys` 和
  `values`。与 `List` 一样，副本共享同样的条目。键保持有序；JSON 对象使用
  字符串键。
* `List` 还有 `sort`（数字、字符串、布尔值）和 `reverse`；`Text` 有 `chars` 和 `lines`。
* 函数是值。函数的类型写得像去掉名字的函数头：
  `(Int, Int) -> Int`、`() -> Text`、不返回任何东西的写作 `(Int) -> ()`。名字可以保存
  函数（`let f = twice`），参数、字段、列表元素或返回值也可以：
  `fn apply(f: (Int) -> Int, x: Int) -> Int: f(x)`、`fn adder(n: Int) -> (Int) -> Int: x => x + n`。
  闭包的参数可以写明类型：`(x: Int) => x + 1`。用错误数量或错误类型的参数调用函数
  是编译错误，会显示两个类型。
* `x => x + 1` 和 `(a, b) => a + b` 是单行函数。需要更多内容时，请在函数内部写一个
  `fn`：`fn add(a: Int): a + k`。两者都按值捕获变量，所以对外部变量
  赋值是编译错误；共享的 `List` 和 `Map` 值仍然可以在
  其内部被修改。
* 这些在解释器中运行；原生后端目前还不能编译它们。

## 表与张量的简短写法

```telvax
type Sale:
    id: Int
    region: Text
    units: Int
    price: Float

fn main():
    let sales: Table[Sale] = read_csv("examples/dataframe/sales.csv")
    print(f"mean price: {sales.price.mean()}")

    let by_region: Table[{region: Text, revenue: Float}] = sales.sql("SELECT region, SUM(units * price) AS revenue FROM t GROUP BY region")
    for row in by_region:
        print(f"{row.region}: {row.revenue}")
```

* 标准库的每个函数和方法及其类型，都在[参考文档](https://github.com/telvax/telvax/blob/main/docs/reference/std.md)中。
* `print`、`read_csv`、`read_parquet` 和 `read_ipc` 不需要 `std.` 前缀。`print` 会换行；
  `std.io.print` 写出时不换行。
* 张量上的方法就是 `std.tensor` 的函数：`x.matmul(w).relu().softmax()`、
  `sales.price.mean()`。表有 `sql`。它们底层是同样的调用，所以
  `std.tensor.relu(x)` 仍然可用。
* 张量上的运算符：`a + b`、`a - b`、`a * b`、`a / b`（形状相同）以及 `a * k`、`k * a`，用来
  按数字缩放。
* `Table[{region: Text, revenue: Float}]` 就地写出行类型；不需要为查询结果
  另建一个一次性的类型。相同的列在任何地方都意味着相同的类型。
* `fn main():` 可以不返回任何东西。

参见 `examples/dataframe/`。同一个程序的 pandas 版本长度差不多，但
Telvax 的版本会在运行之前检查列名和类型。

## 错误、元组与迭代器

```telvax
fn split_pair(line: Text) -> Result[(Text, Text), Text]:
    let parts = line.split("=")
    if parts.len() != 2:
        return Err(f"bad line: {line}")
    Ok((parts[0].trim(), parts[1].trim()))

fn port_of(line: Text) -> Result[Int, Text]:
    let key, value = split_pair(line)?  # `?` returns the Err early
    match value.to_int():
        case Some(n):
            Ok(n)
        case None:
            Err(f"not a number: {value}")

fn main():
    let xs = [3, 1, 4, 1, 5]
    let big = xs.filter(x => x > 2).map(x => x * 10)  # [30, 40, 50]
    for i, x in xs.enumerate():
        print(f"{i}: {x}")
    let stack = [1, 2, 3]
    while stack.pop() is Some(top):
        print(f"{top}")
    if xs.first() is Some(first):
        print(f"first = {first}")
```

* `Result[T, E]` 有 `is_ok`、`is_err`、`unwrap` 和 `expect("message")`（两者都会带着错误
  终止程序）、`unwrap_or(v)`、`unwrap_or_else(e => v)`、`map`、`map_err`、`and_then`，以及 `ok()` /
  `err()`，它们给出一个 `Option`。`and_then` 必须返回错误类型相同的 `Result`；请先用
  `map_err` 改变错误类型。不存在的方法是编译错误。
* 对 `Result` 或 `Option` 使用 `expr?` 会从当前函数返回错误（或 `None`）；该
  函数的返回类型也必须是 `Result` 或 `T?`，并且错误类型必须一致。
* `if x is Pattern:` 和 `while x is Pattern:` 匹配一个模式并绑定其中的名字；它们可以和
  `break`、`continue` 一起使用，模式不匹配时运行 `else:`。
* 元组：类型 `(A, B)`、值 `(1, "a")`、字段 `.0` 和 `.1`，以及在
  `let a, b = pair`、`for` 循环和 `((a, b)) => ...`（把一个元组参数拆开）中的解构。
* `List` 的方法 `map`、`filter`、`any`、`all`、`find`、`enumerate`、`zip`、`sum`、`min`、`max`、
  `fold`、`for_each`、`skip` 和 `join`，以及 `Map.entries`。`min`、`max` 和 `find` 返回一个
  `Option`。这些在解释器中运行；原生后端目前还不能编译它们。

## 类型、方法、trait 与泛型

```telvax
trait Shape:
    fn area(self) -> Float: ...

    fn describe(self) -> Text:
        f"area {self.area()}"

type Circle(Shape):
    r: Float

    fn area(self) -> Float:
        self.r * self.r * 3.14159

type Stack[T]:
    items: List[T]

    fn push(self, x: T):
        self.items.push(x)
```

`type` 先列出它的字段，再列出它的方法。`type Circle(Shape)` 表示该类型实现了这个
trait；没有函数体的方法（`: ...`）是必须实现的，有函数体的是默认实现。`extend Circle:`
为在别处声明的类型添加方法。没有 `self` 的函数这样调用：`Circle.new(1.0)`。
名字后面的 `[T]` 声明一个类型参数；`T: Shape` 把它限定为实现了 `Shape` 的类型。
方法除了所属类型的类型参数外，还可以有自己的类型参数：
`fn map[U](self, f: (T) -> U) -> Stack[U]`。它们由调用推断出来（`s.map(x => x * 2)`），
每一种不同的取值都会生成各自的实例。trait 的方法不能有这样的参数。
调用泛型函数时，类型由参数推断。当无法推断时（比如
不接收任何参数的函数），或者想要显式指定时，请把它们写在括号前面：
`empty[Text]()`、`pair[Int, Text](1, "a")`。类型是 `Int` 或 `List[Text]` 这样的名字。
有三个限定不需要你自己的 trait：`T: Num`（`Int`、`Float` 以及带位宽的数字）、`T: Ord`
（数字和 `Text`，即 `<` 和 `>` 可以比较的值）和 `T: Eq`（除函数以外的一切）。要同时满足多个：
`T: (Num, Ord)`。不满足限定的类型，会在使用它的调用处报错，并且
你自己定义的名为 `Num`、`Ord` 或 `Eq` 的 trait 优先。

完整的程序见[用例](use-cases.md)。
