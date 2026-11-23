> 译自 [telvax/telvax@4c5a64d](https://github.com/telvax/telvax/commit/4c5a64ddd71f1b78a06434638f6d7ef77a6be530)（`docs/reference/cli.md`）。

# 命令行

`telvax` 是一个带有多个命令的程序：`telvax <命令> [选项]`。`telvax --version` 打印版本，
`telvax --help` 打印下面的列表。`telvax doctor` 会告诉你手头这个构建能做什么。包管理器 `tvpm` 在[最后](#tvpm)。

## 用法概览

```text
usage:
  telvax check [--json] [file.tvx...]                       no file: the package in this directory
  telvax lint <file.tvx> [--allow|--warn|--deny rule]... [--deny-warnings] [--list-rules]
                                                            unused variables and functions, unreachable cases
  telvax fmt [--check] [--diff] [file.tvx|dir]...             format in place (`-` reads stdin; no path: this directory)
  telvax schema <file.csv|.parquet|.arrow> [--name Row]      the `type` for the columns of a data file
  telvax doc [--public] [-o out.md] [file.tvx|dir]...      Markdown reference from the types, signatures and `#` comments
  telvax test [file.tvx|dir] [name] [--allow list] [--coverage] [--fail-under N]
                                                            run the `test_*` functions (a directory: `test_*.tvx`, `*_test.tvx`)
  telvax run <file.tvx> [function] [args...] [--allow list] [--quantize int8|int8-channel|int4-channel] [--device gpu]
  telvax profile <file.tvx> [function] [args...] [--allow list] [--top N] [--json f] [--collapsed f] [--speedscope f]
                                                            time per function, line and std operation, and memory
  telvax doctor                                             what this build can do and how it is set up
  telvax repl [file.tvx...] [--allow list]                  type definitions, variables and expressions and see the result
  telvax dap                                                debug adapter over stdio (breakpoints, stepping, variables)
  telvax lsp                                                language server over stdio
  telvax worker [--listen host:port] [--token T]            run calls for a coordinator
  telvax serve <file.tvx> [--port N] [--host H] [--seed docs.jsonl] [--live] [--allow list] [--quantize int8|int8-channel|int4-channel] [--device gpu] [--metrics] [--log-format json|text] [--log-level debug|info|warn|error] [--cors origins] [--rate-limit N/min] [--security-headers] [--process-time-header] [--shutdown-wait N] [--error-format telvax|detail] [--watch]
  telvax build --serve <file.tvx> [-o out] [--dockerfile]   single-binary server (+ Dockerfile)
  telvax jit <file.tvx> <function> [args...]                 (native, needs --features llvm)
  telvax build <file.tvx> [-o out] [--emit exe|obj|llvm-ir|mlir|ptx|stablehlo] [-O 0-3] [--sm sm_80]
                                                    (all but stablehlo need --features llvm)

--allow takes a list of fs.read, fs.write, fs, env, net, all, none. Without it everything is allowed.
```

这一块是 `telvax --help` 的输出，有一个测试会检查它是否仍然如此。

## 退出状态

| 状态 | 含义 |
|---|---|
| 0 | 命令完成了被要求的事。 |
| 1 | 失败：程序有错误、测试失败、文件需要格式化、文件不存在。原因会打印在 `error:` 之后。 |
| 2 | 调用方式有误：未知的命令（如果有相近的命令会给出建议）或不匹配的选项。 |

## 多个命令共有的选项

`--allow 列表`
: 程序拥有的权限：`fs.read`、`fs.write`、`fs`（两者）、`env`、`net`、`all` 或 `none`，用逗号分隔
  （`--allow fs.read,net`）。需要某个未授予的权限的函数会报错并停止。不带 `--allow` 时
  所有权限都允许。`run`、`test`、`profile`、`repl` 和 `serve` 接受该选项。（`lint` 有自己的
  `--allow 规则`，见下文。）

`--quantize int8 | int8-channel | int4-channel | none`
: 用整数运算执行 `f32` 矩阵乘法。参见[导览](../guide/tour.md)中的“量化”一节。`run`、`serve` 和 `build` 接受该选项。

`--device cpu | gpu`
: 张量运算在哪里执行；`gpu` 需要带有 `gpu` 特性的构建和一块 GPU（`telvax doctor` 会说明该构建是否支持）。`run` 和 `serve` 接受该选项。

`--workers 主机:端口,主机:端口`
: 把 `std.dist.map` 的调用发送给运行着 `telvax worker` 的机器。环境变量 `TELVAX_WORKERS` 是
  默认值。`run` 和 `serve` 接受该选项。

## 命令

### `telvax check`

解析并对指定的文件做类型检查；不指定文件时，检查当前目录中的包（带有
`telvax.toml` 的那个）。打印每个错误及其位置，有任何错误则失败。
`--json` 把诊断信息打印为 JSON，每条一个对象，含有 `file`、`line`、`column`、`end_line`、`end_column`、
`severity`、`message` 和 `notes`，供编辑器和脚本使用。

### `telvax lint`

检查一个文件，然后报告没有被用到的代码：变量、函数、永远不会被执行到的分支。
`--list-rules` 打印每条规则及其检查的内容。规则的级别（`allow`、`warn`、`deny`）来自
`telvax.toml` 的 `[lint]` 表，然后是 `--allow 规则`、`--warn 规则` 和 `--deny 规则` 这些选项。
`--deny-warnings` 让警告也使命令失败。

### `telvax fmt`

把文件改写成 Telvax 唯一的格式，并对每个被改动的文件打印 `formatted <文件>`。给文件夹表示
其下所有的 `.tvx` 文件；不给路径表示当前目录；`-` 从标准输入读取源码，
并把结果写到标准输出。`--check` 不改动任何内容，如果有文件会被改动就失败，这正是
CI 作业所需要的。`--diff` 显示会有什么变化。

### `telvax schema`

读取 `.csv`、`.parquet` 或 `.arrow` 文件，并打印其各列的 `type`，可以粘贴到
`let t: Table[Sales] = read_csv("sales.csv")` 的上方。CSV 会被完整读取：表头给出列名，值给出
类型（`Int`、`Float`、`Bool`、`Date`、`Timestamp`，否则是 `Text`），含有空值的列写成 `T?`。对于
Parquet 和 Arrow，使用文件自带的模式，包括哪些列可以为空。类型以文件名命名
（`sales_2024.csv` 得到 `Sales2024`）；`--name Row` 可以指定另一个名字。名字不是合法字段名的列
（`unit price`）会被报告，而不会被改名。

### `telvax doc`

为指定文件（给文件夹表示其下所有的 `.tvx` 文件）中的函数、类型、枚举和 trait 生成 Markdown 参考文档，
并带上直接写在每一项上方的 `#` 注释行。`--public` 省略以 `_` 开头的名字。`-o out.md` 写入文件，而不是标准输出。

### `telvax test`

运行文件中名为 `test_...` 的函数；测试在函数返回时通过，在以错误停止时失败
（`assert`、`assert_eq`、`fail`，以及任何运行时错误）。给文件夹会运行其下所有的 `test_*.tvx` 和 `*_test.tvx` 文件。第二个
词 `name` 只运行名字中包含它的测试。打印 `N passed, M failed`，有任何测试失败则命令失败。`--coverage` 还会报告测试执行了程序的哪些行；`--fail-under N`（0 到 100）把它变成一道
门槛：如果执行到的行数少于 `N`%，命令就失败。`std.test` 的函数，例如
`mock`，在[库参考](https://github.com/telvax/telvax/blob/main/docs/reference/std.md)中。

### `telvax run`

调用程序中的一个函数：`telvax run app.tvx main`，或者只写 `telvax run app.tvx` 来调用 `main`。函数名之后的词
依次是它的参数。`Text` 类型的参数直接取这个词；其他类型取 JSON（`3`、
`true`、`[1, 2]`、`{"a": 1}`）。函数返回的值会被打印，除非它什么也不返回。

### `telvax profile`

像 `run` 一样运行一个函数，并报告时间和内存花在了哪里：按函数、按行、按 `std` 操作。
`--top N` 把报告限制在耗时最长的 `N` 条语句。`--json 文件`、`--collapsed 文件`（供火焰图工具使用）和
`--speedscope 文件` 也会以这些格式写出报告。

### `telvax doctor`

打印版本、它能读取的语法版本、这个构建能做什么（`tables`、live 服务、GPU、原生
后端），以及相关的环境变量（`TELVAX_LIVE`、`OLLAMA_HOST`、`OPENAI_API_KEY`、
`TELVAX_WORKERS`、`TELVAX_CLUSTER_TOKEN`）。密钥或令牌只显示为 `set` 或 `not set`，绝不显示它的值。请把
输出放进 bug 报告里。

### `telvax repl`

启动一个交互式会话：输入类型定义、变量和表达式，并查看结果。命令行上指定的文件会先被加载。
playground 里也是同一个会话。在会话中，`:help` 列出它的命令。

### `telvax dap`

调试适配器（断点、单步执行、变量），通过标准输入和输出使用 Debug Adapter Protocol。由编辑器启动；
VS Code 扩展就是这样做的。你不需要手动运行它。

### `telvax lsp`

语言服务器，通过标准输入和输出使用 Language Server Protocol。由编辑器启动；
VS Code 扩展就是这样做的。你不需要手动运行它。

### `telvax worker`

为协调者运行调用，这样 `--workers` 就能把 `std.dist.map` 分散到多台机器上。它监听
`--listen 主机:端口`（默认 `127.0.0.1:7077`），并且在设置了 `--token T` 或环境变量
`TELVAX_CLUSTER_TOKEN` 时，只响应持有相同令牌的协调者。

### `telvax serve`

通过 HTTP 提供程序的 `@endpoint` 函数，请求会被校验，响应写成 JSON，
直到被停止（Ctrl-C 或 SIGTERM）。它监听 `--host`（默认 `127.0.0.1`）和 `--port`（默认 `8080`）。没有
`@endpoint` 函数的程序会报错。`--live` 使用真实的 Ollama、OpenAI 和 PostgreSQL 服务，而不是
离线的替身（并且需要 `net`）。`--seed docs.jsonl` 在第一个请求之前填充向量库：每行一个 JSON
对象，含有文本字段 `collection`、`id` 和 `text`；每段文本都会被嵌入（embedding）并存储。

`--watch` 用于开发。它监视程序的文件（入口文件、它导入的模块、它用
`std.fs.embed` 嵌入的文件，以及它所在项目的 `telvax.toml` 和 `telvax.lock`），当其中一个改变时，检查
程序，并在相同的主机和端口上重新启动服务器。如果检查失败，错误会连同行号一起打印出来，
正在运行的服务器保持不变，所以一个笔误不会让服务宕掉。新的服务器会再次运行 `on_start`，
旧服务器在内存中保存的内容会丢失。如果 `--watch` 启动时程序有错误，它会等待改动，而不是退出。
Ctrl-C 和 SIGTERM 会同时停止服务器和 `--watch`。其他选项原样传给服务器。

`--metrics` 以 Prometheus 格式提供 `GET /metrics`：`http_requests_total{method,path,status}`、
直方图 `http_request_duration_seconds{method,path}`、程序用
`std.metrics.count(name)` 创建的计数器，以及用 `std.metrics.set(name, value)` 设置的仪表（以最后一个值为准），各自使用自己的名字
（`name`，带标签时是 `name{label="value"}`）和自己的 `# TYPE`。一个名字要么是计数器，要么是仪表，不能两者都是。`path` 是程序中写下的路由，所以
序列的数量不会随 URL 里的 id 增长。`--log-format json`（或 `text`）为每个请求向
标准错误写一行：时间、级别、请求 id、方法、不含查询字符串的路径、状态和耗时。状态低于 400 时级别是
`info`，4xx 是 `warn`，5xx 是 `error`；`--log-level`（或 `TELVAX_LOG_LEVEL`）是会被写出的最低级别，
默认 `info`，所以 `--log-level warn` 只保留失败的请求。`std.log` 的行
遵循同样的格式和级别。使用其中任何一个选项时，每个
响应都有一个 `x-request-id` 头：如果客户端发送的 id 简短且朴素就用它，否则生成一个新的；
函数可以用 `req.header("x-request-id")` 读取它。用 `telvax build --serve` 启动的程序接受同样的
选项，或者 `TELVAX_METRICS=1` 和 `TELVAX_LOG_FORMAT=json`；选项优先于环境变量。

在运行时失败的函数（索引越界、除以零）会以 `500` 和 `Internal server error` 响应，两种
错误格式都是如此，因为解释器的消息会暴露程序内部的名字。这条消息连同请求 id 写到标准错误，
形如 `error GET /path 500 request_id=...: runtime error: ...` 的一行（使用 `--log-format json` 时是一个 JSON 对象）。
返回 `Err(Status.Internal(message))` 的函数仍然会发送它的 `message`，因为这是程序自己选的。

还有三个选项作用于每个响应。`--cors 来源`（`*`，或用逗号分隔的来源）让来自
允许的来源的浏览器页面能够读取响应：它会添加 `access-control-allow-origin`，并以 `204` 响应预检 `OPTIONS`；
不支持凭据。`--rate-limit N/单位`（`second`、`minute` 或 `hour`）允许同一个客户端在
每个时间窗口内发出 `N` 个请求，超过之后以 `429` 和 `retry-after` 响应；客户端指的是连接的地址，使用 `--trust-proxy` 时则是 `x-forwarded-for` 的
第一个地址。`/healthz` 从不受限。`--shutdown-wait N`（或 `TELVAX_SHUTDOWN_WAIT`）是被停止的服务器
等待其请求和后台任务的秒数（默认 30）。`--security-headers` 会给没有设置它们的响应添加
`x-content-type-options: nosniff`、`x-frame-options: DENY`、`referrer-policy: no-referrer` 和 `cache-control: no-store`。
相应的环境变量是 `TELVAX_CORS`、`TELVAX_RATE_LIMIT`、`TELVAX_SECURITY_HEADERS`
和 `TELVAX_TRUST_PROXY`。限流的计数保存在内存中，按进程计算。

这些选项也可以写在程序里，放在入口文件的 `serve:` 块中：

```
serve:
    metrics: True
    log_format: "json"
    log_level: "info"
    cors: "*"
    rate_limit: "100/min"
    security_headers: True
    process_time_header: True
    trust_proxy: False
    shutdown_wait: 30
    error_format: "detail"
    on_start: [init_db]
```

每一行是上面的一个选项（`--log-format` 对应 `log_format`），取一个固定的值：`True` 或 `False`、整数
秒数，或者一段文本。一行也可以改为在服务器启动时读取一个环境变量，`env("NAME")` 或
`env("NAME", "default")`，例如 `cors: env("CORS_ORIGINS", "*")`：值是该变量的文本，变量未设置或为空时
取默认值（没有默认值时这一行被忽略），读取方式与选项一致。`True or False` 类型的选项
在值为 `1` 或 `true` 时开启；它的默认值是 `1`、`true`、`0` 或 `false`。`telvax check` 会拒绝选项本身
会拒绝的默认值；选项拒绝的变量值（`LOG_LEVEL=loud`）会让启动停止，并报出
`serve: LOG_LEVEL="loud" is not a value of `log_level`: ...`。不做任何转换：`RATE_LIMIT_PER_MINUTE=100` 不等于
`100/min`。`telvax check` 会报告不在列表中的名字、类型错误或选项会拒绝的值、
重复写了的名字、第二个 `serve:` 块，以及不在入口文件中的块。环境变量
（`TELVAX_METRICS` 等）优先于块，选项优先于环境变量。`--port`、`--host`、`--live`、
`--allow`、`--seed`、`--quantize` 和 `--device` 不在块里：它们取决于程序在哪里运行。由
`telvax build --serve` 生成的二进制文件会保留这个块。

`middleware: [名字]` 列出标记为 `@middleware` 的函数，形式为 `fn name(req: Request, next: Next) -> Response`
（`Request`、`Response` 和 `Next` 来自 `std.http`），最外层的在前。它们在内置选项的内侧运行，并包裹每个
请求，包括各条路由，以及 `/health`、`/metrics`、`/openapi.json` 和 `/docs`。不调用 `next` 的中间件
自己给出响应；调用它的中间件会得到链上其余部分的 `Response`，可以修改其状态和头
（正文原样传递），并且 `next` 只能调用一次。失败的中间件以 `500` 响应。`telvax check`
会拒绝不是 `@middleware` 函数的名字、错误的签名和重复写了的名字，并对入口文件中列表里没有提到的
`@middleware` 给出警告。它们在解释器中运行，开销比内置
选项更大。

`on_start: [名字]` 是唯一一行不是字面量的：它是一个顶层函数列表，这些函数没有参数，各自返回
空值或 `Result[(), Text]`，在打开端口之前按书写顺序运行一次（所以在它们完成之前 `/health` 不会
响应）。它们使用程序的权限运行（`--live`、`--allow`）。返回
`Err` 或失败的函数会中止启动：消息以 `on_start: NAME failed: MESSAGE` 的形式写到标准错误，
退出状态为 1。`telvax check` 会报告不是函数的名字、带参数或返回类型不符的函数，
以及重复写了的名字。函数可以来自另一个文件（`from db import init_db`）。它们在 `telvax run`
或 `telvax test` 下不会运行。

### `telvax build --serve`

生成一个同时包含程序和服务器的可执行文件：`telvax build --serve app.tvx -o app`。它复制运行该命令的
`telvax`，并把程序附加在后面，所以它和 `telvax` 一样大。启动时它
提供这个程序的服务：监听 `0.0.0.0`，如果设置了 `PORT` 就用它，`--port` 和 `--host` 仍然优先。`HOST` 设置
地址，`TELVAX_LIVE=1` 开启 `--live`。`app --healthcheck [/路径]` 向这台机器上的服务器请求
`/路径`（省略时是 `/healthz`；端口是 `--port`、`PORT` 或 8080），收到 2xx 响应时以 0 退出，否则以 1 退出：容器
里没有 `curl` 可用。`--dockerfile` 还会在程序旁边为它写一个 `Dockerfile`：它复制整个
目录（其他模块和 `std.fs.embed` 读取的文件必须在其中；用 `.dockerignore` 排除
其余内容），基于 `ghcr.io/telvax/telvax`（或 `--build-arg TELVAX_IMAGE=...` 指定的镜像）构建，并带有一个调用 `--healthcheck` 的
`HEALTHCHECK`。

### `telvax build`

用原生后端（LLVM 和 MLIR）编译程序，所以需要用 `--features llvm` 构建的 `telvax`；
已发布的二进制文件不是这样构建的，`telvax doctor` 会说明。`--emit` 选择输出：`exe`（默认，以
文件名命名的程序）、`obj`（目标文件）、`llvm-ir` 和 `mlir`（打印）、`ptx`（打印；`--sm sm_80` 选择 GPU
代际）以及 `stablehlo`（打印）。`-o` 指定输出文件名。`-O 0` 到 `-O 3` 是优化级别
（默认 2）。`--emit ptx` 和 `--emit stablehlo` 不需要原生后端。

### `telvax jit`

用原生后端编译一个函数并立即调用它：`telvax jit app.tvx norm 3.0 4.0`。需要 `--features llvm`。

## `tvpm`

`tvpm` 是包管理器。它读取 `telvax.toml`，写入 `telvax.lock`，并在装好依赖的情况下运行对应的 `telvax` 命令。
`tvpm --help` 打印命令列表。`tvpm` 在当前目录或最近的、带有 `telvax.toml` 的
上级目录中找到项目。退出状态：命令完成了被要求的事时为 0，失败时为 1（原因
打印在 `error:` 之后），未知命令为 2。

### 注册表

注册表是一个目录或一个 `http(s)://` URL，布局相同：`index/<名字>.json` 为每个已发布的
版本保存一行 JSON（`name`、`vers`、`deps`、`cksum`、`yanked`），`packages/<名字>-<版本>.nvpkg` 是一个 gzip 压缩的 tar，其中有
`telvax.toml`、`src/**/*.tvx`，以及（如果存在）`README.md` 和 `LICENSE*` 文件。任何静态文件托管都可以。

`--registry <位置>`（放在命令之前或之后）或环境变量 `TVPM_REGISTRY` 指定使用哪一个。没有
指定时，来自注册表的依赖会报错并说明这一点。下载的包解压到 `$TVPM_HOME`，或
`~/.tvpm`，并对照索引检查它们的校验和。

`tvpm search`、`publish` 和 `yank` 读写的是**目录**型注册表；它们不能用于 URL。

### 命令

`tvpm new <名字> [--lib]`
: 创建文件夹 `<名字>`，其中有 `telvax.toml`、`src/main.tvx`（使用 `--lib` 时是 `src/lib.tvx`）和一个 `.gitignore`。名字
  由小写字母、数字和单个 `_` 组成，以字母开头，并且不能是 `std`、`crate`、`self` 或 `super`。

`tvpm init`
: 在当前目录中做同样的事，并以该目录命名。如果 `telvax.toml` 已经存在则失败。

`tvpm add <名字>[@req] [--path DIR]`
: 向 `telvax.toml` 添加一个依赖。`name@1.2` 写入该要求；不带要求时，`tvpm` 在注册表中查找最新的、没有被撤回（yanked）的版本，并写入 `^<版本>`。`--path DIR` 依赖一个目录（相对于项目），其中的包
  具有该名字。它不会锁定或下载：请运行 `tvpm install`。

`tvpm remove <名字>`
: 从 `telvax.toml` 中删除一个依赖。

`tvpm lock`
: 解析版本并写入 `telvax.lock`。它会打印每个被锁定的包。

`tvpm install`
: 如果 `telvax.lock` 缺失或与 `telvax.toml` 不一致就先锁定，然后下载尚未安装的注册表包。

`tvpm update [名字...]`
: 把锁定的版本移到符合要求的最新版本。给出名字时，只有这些注册表包会移动，其他的保持
  锁文件中的版本，除非某个要求迫使它们变化。每个变化都会打印 `updated name 1.0.0 -> 1.1.0`。

`tvpm tree`
: 打印依赖树。

`tvpm search <文本>`
: 列出目录型注册表中名字包含 `文本` 的包，每个包带有它最新的、没有被撤回的版本。

`tvpm publish`
: 像 `telvax check` 那样检查包，打包，并把它添加到目录型注册表。如果包没有通过
  检查、该版本已经发布（请先提高 `version`），或者它有路径依赖，就会失败：已发布的包
  只能依赖注册表中的包。

`tvpm yank <名字> <版本> [--undo]`
: 把一个已发布的版本标记为已撤回（yanked）。新的解析会跳过它，已经写有它的 `telvax.lock` 仍然可以安装它。
  `--undo` 把它重新标记为可用。

`tvpm check`、`run [fn]`、`serve [选项]`、`build [选项]`
: 必要时先锁定并安装，然后对入口文件运行 `telvax check`、`run`、`serve` 或 `build`，并把其余的词
  作为参数传入。不带函数的 `run` 调用 `main`。`tvpm` 启动与它放在一起的 `telvax`，或者 `TELVAX_BIN` 中的那个，或者
  `PATH` 中的那个。

### `telvax.toml`

```toml
[package]
name = "app"                  # required
version = "0.1.0"             # required, semantic version
edition = "2026"              # the syntax the package is written in; "2026" when absent
description = "..."           # optional
entry = "src/app.tvx"         # optional; default src/main.tvx, then src/lib.tvx

[dependencies]
mathx = "^1.2"                         # from the registry
shared = { path = "../shared" }        # from a directory
pinned = { version = "=1.4.2" }        # the same as a string; not together with `path`

[lint]                                 # what `telvax lint` does for each rule: allow, warn or deny
unused-variable = "deny"
unused-function = "allow"
```

版本要求与 Cargo 的相同：`1.2` 和 `^1.2` 表示 `>=1.2.0, <2.0.0`，`~1.2` 表示 `>=1.2.0, <1.3.0`，`=1.2.3`
表示只有这个版本，`>=1, <3` 把两者组合起来。每个包名只解析为一个版本；当两个包对第三个包需要
不同的范围时，`tvpm` 选择两者都适用的最新版本，如果没有则失败。这个 `telvax` 读不了的 `edition` 是一个错误，错误信息会说明它能读哪些。

### `telvax.lock`

`tvpm lock` 写入它；不要编辑它。它为项目本身和它依赖的每个包各有一个 `[[package]]`，包含
`name`、`version`、`source`（`registry` 或 `path+<dir>`；项目本身没有）、注册表包的 `checksum`，以及
`dependencies`，一个名字列表。请把它提交：这样每台机器使用的版本都相同。当 `telvax.toml` 中有
它没有覆盖的依赖或要求时，它就过期了，这时 `install`、`check`、`run`、`serve` 和 `build` 会重新锁定。
