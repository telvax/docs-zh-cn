> 译自 [telvax/telvax@664d389](https://github.com/telvax/telvax/commit/664d3890800279185aac3fa7624632fd396b53fd)（`docs/guide/use-cases.md`）。

# 用例

Telvax 适合做什么，每种情形都配有一个程序，可以从仓库根目录运行。
下面的每个程序都在 [`examples/`](https://github.com/telvax/telvax/tree/main/examples) 中，并用所示的命令运行。

| 你想要 | 参见 | 运行方式 |
|---|---|---|
| 汇总一个 CSV 或 Parquet 文件 | [`dataframe/sales.tvx`](https://github.com/telvax/telvax/blob/main/examples/dataframe/sales.tvx) | `telvax run` |
| 在模型运行之前检查它的形状 | [`ml/forward.tvx`](https://github.com/telvax/telvax/blob/main/examples/ml/forward.tvx) | `telvax run` |
| 通过 HTTP 提供一个函数 | [`service/sentiment.tvx`](https://github.com/telvax/telvax/blob/main/examples/service/sentiment.tvx) | `telvax serve` |
| 构建 RAG 服务 | [`rag-service.tvx`](https://github.com/telvax/telvax/blob/main/examples/rag-service.tvx) | `telvax serve` |
| 处理文本 | [`text/word-count.tvx`](https://github.com/telvax/telvax/blob/main/examples/text/word-count.tvx) | `telvax run` |
| 不用异常来处理错误 | [`errors/parse-config.tvx`](https://github.com/telvax/telvax/blob/main/examples/errors/parse-config.tvx) | `telvax run` |
| 编写普通的数值代码 | [`basics/primes.tvx`](https://github.com/telvax/telvax/blob/main/examples/basics/primes.tvx) | `telvax run`，或用 `telvax build` 编译为原生代码 |

## 1. 表上的数据分析

`Table[Row]` 是一张 Arrow 表，它的列就是类型 `Row` 的字段。列名
和类型在程序运行之前就会被检查，读取列时会把它当作张量，而不复制数据。

```telvax
type Sale:
    id: Int
    region: Text
    units: Int
    price: Float

fn main():
    let sales: Table[Sale] = read_csv("examples/dataframe/sales.csv")
    print(f"mean price: {sales.price.mean()}")

    let by_region: Table[{region: Text, revenue: Float}] = sales.sql("SELECT region, SUM(units * price) AS revenue FROM t GROUP BY region ORDER BY revenue DESC")
    for row in by_region:
        print(f"{row.region}: {row.revenue}")
```

```sh
telvax run examples/dataframe/sales.tvx main
```

可用于报表、特征准备以及对数据文件的快速检查。拼错的列
（`sales.prise`）是一个编译错误（`no column prise on type Table[Sale]`），而不是运行到
一半才失败。SQL（和 Parquet）需要完整构建；`telvax-lite` 把表保存在内存中，并读取 CSV。

## 2. 形状经过检查的模型

张量的形状是类型的一部分，所以形状不匹配的矩阵乘法无法通过编译：

```telvax
fn layer1(x: Tensor[f32, 2, 3], w: Tensor[f32, 3, 4]) -> Tensor[f32, 2, 4]:
    x.matmul(w).relu()
```

```sh
telvax run examples/ml/forward.tvx main
telvax run examples/ml/forward.tvx main --device gpu      # needs a build with --features gpu
telvax run examples/ml/forward.tvx main --quantize int8   # int8 matrix products
```

把 `layer1` 中的权重改成 `Tensor[f32, 4, 4]`，`telvax check` 会在该调用处报告
``matmul inner dimensions differ: 3 and 4``。当某个大小只有在运行时才知道时，请使用符号维度（`Tensor[f32, N, 3]`）。
这里的权重来自 `std.tensor.from_vec`；真实的模型会从文件或表中加载它们。

## 3. 通过 HTTP 提供函数

`@endpoint` 把函数变成一条路由。参数的类型就是 JSON 请求体，并且会替你
进行验证；`Err(Status.BadRequest(...))` 会变成 400。

```telvax ignore
@endpoint(POST, "/v1/sentiment")
fn sentiment(input: Input) -> Result[Output, Status]:
    if input.text.is_empty():
        return Err(Status.BadRequest("empty text"))
```

```sh
telvax serve examples/service/sentiment.tvx
curl -XPOST localhost:8080/v1/sentiment -H 'content-type: application/json' -d '{"text":"great and good"}'
# {"label":"positive","score":2}
curl -XPOST localhost:8080/v1/sentiment -H 'content-type: application/json' -d '{"text":""}'
# {"error":"empty text"}   (HTTP 400)
```

要发布它，`telvax build --serve examples/service/sentiment.tvx -o service --dockerfile` 会生成一个
内嵌了程序的二进制文件，以及一个 Dockerfile。

## 4. 检索增强生成

[`examples/rag-service.tvx`](https://github.com/telvax/telvax/blob/main/examples/rag-service.tvx) 对问题做嵌入，在向量
存储中搜索，用命中的结果构造提示词，然后询问模型，全部放在一个 `@endpoint` 中。各阶段用
`|>` 串联：

```telvax ignore
let answer = prompt |> llm.complete() |> std.text.trim()
```

```sh
telvax serve examples/rag-service.tvx          # offline stand-ins for the model and the database
telvax serve examples/rag-service.tvx --live   # Ollama, OpenAI and PostgreSQL with pgvector
```

离线模式让你无需运行这些服务就能开发和测试服务。`--live`
则连接真实的服务。

## 5. 文本处理

`Text`、`List` 和 `Map` 都有你期望的方法，`List` 还有迭代器方法（`map`、
`filter`、`fold`、`sum`、`enumerate`、`zip` 等）：

```telvax
fn count_words(text: Text) -> Map[Text, Int]:
    let counts: Map[Text, Int] = std.map.new()
    for word in text.to_lowercase().split(" "):
        let seen = counts.get(word).unwrap_or(0)
        counts.insert(word, seen + 1)
    counts
```

```sh
telvax run examples/text/word-count.tvx main
```

## 6. 把错误当作值

这里没有异常。可能失败的函数返回 `Result[T, E]` 或 `T?`，`match`
必须覆盖所有情况，`?` 把错误传给调用者。

```telvax ignore
fn port_of(line: Text) -> Result[Int, Text]:
    let key, value = split_pair(line)?
```

```sh
telvax run examples/errors/parse-config.tvx main
```

## 7. 普通的数值代码，解释执行或原生编译

整数、浮点数、循环和你自己的类型，也可以像普通语言一样使用。解释器可以运行
任何代码；原生后端（`--features llvm`）编译 [README](https://github.com/telvax/telvax#language-notes) 中描述的
子集。

```sh
telvax run examples/basics/primes.tvx main
telvax build examples/basics/primes.tvx -o primes     # native, needs --features llvm
```

## 不止一个文件

* **多个文件和包：** `from util import helper` 和 `tvpm`，见[导览](tour.md#模块与包)。
* **多台机器：** `std.dist.map` 配合 `telvax worker`，见[导览](tour.md#分布式)。
* **从 Python 使用：** `python/` 模块无需复制即可交换 Arrow 表和 NumPy 数组。
* **在浏览器中：** playground 把检查器和解释器作为 WebAssembly 运行。
* **编辑器：** `telvax lsp` 和 VS Code 扩展提供诊断、悬停提示和补全。

## Telvax 目前还不适用的情形

它还处于 alpha 阶段。原生后端只覆盖一个很小的子集，PTX 和 StableHLO 的输出还没有在
NVIDIA GPU 或 TPU 上运行过，语法也仍可能改变。[`docs/design/status.md`](https://github.com/telvax/telvax/blob/main/docs/design/status.md)
列出了哪些已经验证过，以及是如何验证的。
