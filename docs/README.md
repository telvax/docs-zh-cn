> 译自 [telvax/telvax@2253a54](https://github.com/telvax/telvax/commit/2253a54139c057b14d91eeded2259ee66e3a2148)（`README.md`）。

<div align="center">

# Telvax

**一门面向数据、机器学习流水线和 AI 服务的静态类型编译型语言。**

[![CI](https://github.com/telvax/telvax/actions/workflows/ci.yml/badge.svg)](https://github.com/telvax/telvax/actions/workflows/ci.yml)
[![Release](https://img.shields.io/github/v/release/telvax/telvax?include_prereleases&label=release)](https://github.com/telvax/telvax/releases)
[![License](https://img.shields.io/badge/license-MIT%20%2F%20Apache--2.0-blue)](#许可证)

[文档](https://telvax.github.io/telvax/) ·
[导览](guide/tour.md) ·
[标准库](https://github.com/telvax/telvax/blob/main/docs/reference/std.md) ·
[命令行](https://github.com/telvax/telvax/blob/main/docs/reference/cli.md) ·
[讨论区](https://github.com/telvax/telvax/discussions)

</div>

Telvax 是一门编程语言。你把程序写在以 `.tvx` 结尾的文本文件里，然后用
`telvax` 命令运行它们，这个命令只需安装一次。它面向处理数据（表、CSV、
Parquet）、机器学习（张量）和 AI 服务（响应 JSON 请求的 Web 服务器）的程序。
如果你从未用过它，请从[快速开始](#快速开始)入手：它不需要任何其他软件。

Telvax 读起来像 Python（靠缩进划分块，没有花括号和分号），检查起来像编译型
语言。张量的形状是类型系统的一部分，管道运算符（`|>`）用来组合各个阶段，
标有 `@endpoint` 的函数会变成一条 HTTP 路由，并自动生成校验和 JSON 处理。

```telvax
import std.ai.embeddings.OllamaEmbedder
import std.http.Status

let embedder = OllamaEmbedder.new("nomic-embed-text")

type QueryPayload:
    query: Text
    top_k: i32

fn embed(text: Text) -> Tensor[f32, 768]:
    text |> std.text.clean_whitespace() |> std.text.truncate(512) |> embedder.embed()

@endpoint(POST, "/v1/embed")
fn handle(payload: QueryPayload) -> Result[Tensor[f32, 768], Status]:
    if payload.query.is_empty():
        return Err(Status.BadRequest("empty query"))
    Ok(embed(payload.query))
```

这是一个服务：它需要一个正在运行的 [Ollama](https://ollama.com) 才能响应。完整的 RAG 服务在
[`examples/rag-service.tvx`](https://github.com/telvax/telvax/blob/main/examples/rag-service.tvx) 中，它用模型的替身离线运行。
不需要任何准备的程序见[快速开始](#快速开始)。

> **早期 alpha。** 适合试用这门语言并反馈意见。库和工具中可能有 bug，
> 也可能有不兼容的改动；2026 版的语法已经冻结（[稳定性](https://github.com/telvax/telvax/blob/main/docs/guide/stability.md)）。
> 哪些可以工作、哪些还没有在真实硬件上运行过，列在 [`docs/design/status.md`](https://github.com/telvax/telvax/blob/main/docs/design/status.md) 中。

## 亮点

* **类型中包含张量形状。** `Tensor[f32, 2, 3]` 带有自己的形状，所以形状不匹配的 `matmul` 是编译错误。
* **带类型的表。** `Table[Row]` 是一张 Arrow 表，它的列就是 `Row` 的字段；可以读取 CSV、Parquet 和 Arrow，并运行 SQL。
* **无需胶水代码的服务。** `@endpoint` 把函数变成经过校验的 JSON 路由；`telvax build --serve` 生成一个二进制文件和一个 Dockerfile。
* **错误就是值。** `Result`、`Option`、`T?`、`?` 运算符以及 `if x is Some(v)`。
* **自带工具。** 格式化器、能给出精确位置的类型检查器、REPL、带覆盖率的测试运行器、性能分析器、调试器（DAP）、语言服务器、VS Code 扩展、包管理器（`tvpm`）和 Jupyter 内核。
* **解释器与原生后端。** 可以立即运行程序，也可以用 MLIR 和 LLVM 编译；还提供面向加速器的 StableHLO 和 PTX 生成器。

## 你可以构建什么

| 用例 | 示例 |
|---|---|
| 用带类型的列和 SQL 汇总 CSV / Parquet 文件 | [`examples/dataframe/sales.tvx`](https://github.com/telvax/telvax/blob/main/examples/dataframe/sales.tvx) |
| 张量形状在运行前就被检查的模型 | [`examples/ml/forward.tvx`](https://github.com/telvax/telvax/blob/main/examples/ml/forward.tvx) |
| 把函数变成经过校验的 JSON HTTP 路由 | [`examples/service/sentiment.tvx`](https://github.com/telvax/telvax/blob/main/examples/service/sentiment.tvx) |
| RAG 服务：嵌入、向量搜索、LLM | [`examples/rag-service.tvx`](https://github.com/telvax/telvax/blob/main/examples/rag-service.tvx) |
| 用 `Map`、`List` 和迭代器方法处理文本 | [`examples/text/word-count.tvx`](https://github.com/telvax/telvax/blob/main/examples/text/word-count.tvx) |
| 把错误当作值：`Result`、`Option`、`?`、`if x is Some(v)` | [`examples/errors/parse-config.tvx`](https://github.com/telvax/telvax/blob/main/examples/errors/parse-config.tvx) |

每一个都在 [`guide/use-cases.md`](guide/use-cases.md) 中附有运行它的命令。

## 安装

```sh
curl -fsSL https://raw.githubusercontent.com/telvax/telvax/main/scripts/install.sh | sh
```

安装完成后不会打开任何东西：Telvax 是命令行工具，没有窗口，也不是应用。脚本
会打印一行信息，并把 `telvax` 和 `tvpm`（包管理器）这两个程序放到 `~/.telvax/bin`。
在 macOS 和 Linux 上，它不会修改你的 `PATH`，所以请自己把这个文件夹加进去（要长期生效，请把这一行写入 `~/.zshrc` 或 `~/.bashrc`），然后检查：

```sh
export PATH="$HOME/.telvax/bin:$PATH"
telvax --version        # prints: telvax, then the version you installed
```

如果 shell 回答 `command not found: telvax`，说明缺少 `PATH` 这一行。如果没有适用于
你的系统的二进制文件，脚本会从源码构建（需要 [Rust](https://rustup.rs)），`cargo` 会把
程序放到 `~/.cargo/bin`。手动构建：`cargo install --locked --path cli`。

每个系统发布两种构建：完整版 `telvax`（约 50 MB，带 SQL 和 Parquet）
和 `telvax-lite`（约 10 MB）。lite 版本包含其余的全部功能，内存中的表和 CSV
也在其中；`std.table.sql` 和 Parquet 在 lite 版中会报出清楚的错误。`TELVAX_LITE=1` 会让安装脚本选择 lite 版。

每次发布还会发布适用于 `linux/amd64` 和 `linux/arm64` 的 Docker 镜像，包含 `telvax` 和 `tvpm`：

```sh
docker run --rm -v "$PWD:/work" -w /work ghcr.io/telvax/telvax run app.tvx main
```

标签是发布的版本号（见[发布页](https://github.com/telvax/telvax/releases)）和 `latest`。

在 Windows（PowerShell）上：

```powershell
irm https://raw.githubusercontent.com/telvax/telvax/main/scripts/install.ps1 | iex
```

在 Windows 上，脚本会把该文件夹加入你的 `PATH`（请打开新的终端）。Windows 构建是新的，只由 CI 编译和测试过，没有人工测试过。原生
后端（`--features llvm`）在 Windows 上还不可用。

## 快速开始

安装 Telvax（见[安装](#安装)），然后把下面的内容保存为 `hello.tvx`：

```telvax
fn main():
    let names = ["Ada", "Alan", "Grace"]
    for name in names:
        print(f"hello, {name}")
```

```sh
telvax run hello.tvx main
```

它会打印：

```text
hello, Ada
hello, Alan
hello, Grace
```

`main` 是要运行的函数。`telvax check hello.tvx` 只读取文件并检查类型，然后打印
`` `hello.tvx` ok ``；有错误时会报告行号和列号。更多命令：

```sh
telvax check hello.tvx          # parse and type-check; diagnostics point at the exact span
telvax repl                     # type definitions and expressions and see the result
telvax fmt --check              # every .tvx file under this directory
telvax test tests/ --coverage   # run the `test_*` functions
telvax serve examples/rag-service.tvx   # HTTP server on :8080 (offline model stand-ins)
```

所有命令和选项都在 [`docs/reference/cli.md`](https://github.com/telvax/telvax/blob/main/docs/reference/cli.md) 中。对于 notebook，
`pip install "telvax[kernel]"` 会添加 Jupyter 内核和 `%%telvax` magic（[`python/README.md`](https://github.com/telvax/telvax/blob/main/python/README.md)）。要从克隆的仓库构建，
需要 nightly 版 Rust（由 `rust-toolchain.toml` 选定）：`cargo build --release -p telvax_cli`。

## 文档

| 阅读 | 用途 |
|---|---|
| [导览](guide/tour.md) | 这门语言，从第一个程序到模块和测试 |
| [用例](guide/use-cases.md) | 数据框、模型、服务和 RAG，附有运行每一个的命令 |
| [标准库](https://github.com/telvax/telvax/blob/main/docs/reference/std.md) | `std` 的每个函数和方法 |
| [命令行](https://github.com/telvax/telvax/blob/main/docs/reference/cli.md) | `telvax` 和 `tvpm` 的每个命令 |
| [稳定性](https://github.com/telvax/telvax/blob/main/docs/guide/stability.md) | 哪些已冻结，哪些可能改变 |
| [状态](https://github.com/telvax/telvax/blob/main/docs/design/status.md) | 现有的内容、验证过的内容及方式，以及尚未完成的内容 |
| [语法](https://github.com/telvax/telvax/blob/main/docs/design/sintaxe-pythonica.md) | 语法的设计 |

文档同时也是一个网站：<https://telvax.github.io/telvax/>。

## 语言说明

* **值。** 你定义的类型和 `Text` 是值；`List` 是引用（副本共享元素）。
  张量是不可变的值。没有类型的数字字面量根据它们的用法获得类型。
* **张量。** `Tensor[f32, 2, 3]` 带有自己的形状：形状不匹配的 `std.tensor.matmul`
  是编译错误。符号维度（`[N]`）接受任意大小。
* **表。** `Table[Row]` 是一张 Arrow 表，它的列就是类型 `Row` 的字段；
  `t.score` 是零拷贝张量形式的列。`head`、`tail`、`sort_by`、`sort_by_desc`、`select`、`group_by`、`filter`、`concat` 和 `split`
  是方法；`sql` 运行查询。`read_ipc` / `write_ipc` 使用 Arrow IPC（Feather）文件，读取时做内存映射；
  `read_flight` / `write_flight` 与 Arrow Flight 服务器通信（用 `--features flight` 构建）。类型为 `T?` 的列可以包含空值，`Date` / `Timestamp` 列是 Arrow 的时间类型，其
  值是 ISO-8601 的 `Text`。
* **模块。** `import util.helper`、`from crate.geometry import Point, norm`、
  `from mathx.vec import dot`（一个依赖）。以 `_` 开头的名字是私有的，不能
  被导入。
* **原生子集。** 整数、浮点数、布尔值、字符串、你定义的类型、数字列表、静态形状的
  `f32`/`f64` 张量、循环以及 f-string（整数、布尔值、字符串）。箭头函数（`x => ...`）、`Table`、SQL 以及 `std` 的大部分
  只在解释器中运行。

## 仓库布局

| 领域 | 位置 | 说明 |
|---|---|---|
| 编译器前端 | `compiler/{lexer,parser,ast,semantic}` | `logos` + `chumsky`，带错误恢复；带类型的 HIR；字面量类型推断；静态检查张量形状 |
| 解释器与运行时 | `runtime/` | 基于 Arrow 的张量和表，CSV / Parquet / SQL（DataFusion），真实的 Ollama / OpenAI / pgvector 客户端，PostgreSQL 上的 SQL |
| 原生后端 | `compiler/{ir,codegen}` | MLIR（`melior`）和 LLVM（`inkwell`）22；JIT 和可执行文件；`matmul` 的 int8 量化。可选：`--features llvm` |
| HTTP 服务 | `server/`、`cli/` | `@endpoint` 变成 Axum 路由；`telvax build --serve` 把程序嵌入一个二进制文件 |
| 模块与包 | `compiler/modules`、`tools/package-manager` | 文件之间的 `import`，带 registry 的 `tvpm`，解析器和 lockfile |
| 编辑器支持 | `tools/lsp`、`editors/vscode` | 诊断、悬停、上下文补全、跳转到定义、推断类型提示 |
| Web playground | `playground/` | 编译为 WebAssembly 的检查器和解释器，带控制台（REPL） |
| 加速器 | `compiler/accel/` | StableHLO 和 PTX 生成器（尚未在 TPU / NVIDIA 硬件上运行）；GPU 执行位于 `runtime`（`gpu` 特性） |
| Python | `python/` | PyO3 模块；零拷贝的 Arrow、NumPy 和 CUDA 数组交换；Ray 执行器 |
| 分布式 | `tools/cluster` | 用于 `std.dist.map` 的 `telvax worker` + `--workers` |
| 基准测试 | `benchmarks/` | alpha 目标：RAG 开销、内存占用、摄取吞吐量 |

[`docs/design/status.md`](https://github.com/telvax/telvax/blob/main/docs/design/status.md) 把原始工程规范中的每一项
对应到现有的内容、验证过的内容及方式，以及尚未完成的内容。

## 测试

Telvax 程序有自己的测试运行器：名为 `test_...` 的函数是一个测试，`fixture_...`
函数给它提供参数，`cases_...` 用多组值运行它，`std.test.mock` 替换一个
函数，`--coverage` 和 `--fail-under N` 报告并守住测试所运行的行数。
[导览](guide/tour.md#测试)里有详细说明。要测试本仓库：

```sh
cargo test --workspace                       # no LLVM needed
. scripts/llvm-env.sh && cargo test --workspace --features llvm
cd python && maturin develop && python -m pytest tests
scripts/build-playground.sh --node && node --test playground/test
cd editors/vscode && npm ci && npm test
```

## 参与贡献

请先阅读 [CONTRIBUTING.md](https://github.com/telvax/telvax/blob/main/CONTRIBUTING.md)。对语法、类型系统或标准库的改动，
都从 [`telvax/rfcs`](https://github.com/telvax/rfcs) 中的一份 RFC 开始。

## 许可证

采用 MIT（`LICENSE-MIT`）或 Apache 2.0（`LICENSE-APACHE`）双重许可：你可以按自己的选择，
在其中任意一种许可下使用 Telvax。

## 社区

问题和想法请发到[讨论区](https://github.com/telvax/telvax/discussions)；bug 和
功能请求请提交到 [issues](https://github.com/telvax/telvax/issues)。请先阅读[行为准则](https://github.com/telvax/telvax/blob/main/CODE_OF_CONDUCT.md)，并按照 [SECURITY.md](https://github.com/telvax/telvax/blob/main/SECURITY.md) 中的说明，
私下报告安全问题。

## 其他仓库

| 仓库 | 内容 |
|---|---|
| [`telvax/rfcs`](https://github.com/telvax/rfcs) | 对语言和工具的改动提案 |
| [`telvax/docs-pt-br`](https://github.com/telvax/docs-pt-br)、[`docs-es`](https://github.com/telvax/docs-es)、[`docs-zh-cn`](https://github.com/telvax/docs-zh-cn) | 导览、用例和本 README 的翻译 |
| [`telvax/website`](https://github.com/telvax/website) | 公开网站（尚未构建；文档网站是这里的 `site/`） |
| [`telvax/workflows`](https://github.com/telvax/workflows) | 共享的 GitHub Actions 工作流（目前还没有） |
| [`telvax/telvax-grammar`](https://github.com/telvax/telvax-grammar) | TextMate 语法的副本，供从仓库获取语法的工具使用（GitHub Linguist） |
| [`telvax/.github`](https://github.com/telvax/.github) | 组织主页和默认的社区文件 |
