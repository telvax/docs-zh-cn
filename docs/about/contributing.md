> 译自 [telvax/telvax@91dedc3](https://github.com/telvax/telvax/commit/91dedc31f176df7cfca19a1f9b7325639f60a324)（`CONTRIBUTING.md`）。

# 为 Telvax 做贡献

感谢你试用 Telvax。在 0.0.x 阶段，最有用的贡献是一份 bug 报告，或是一个行为与你预期不符的程序。

## 报告问题

请创建一个 issue，并包含：

* `telvax --version` 的输出和你的操作系统；
* 能复现问题的最小 `.tvx` 程序；
* 你运行了什么（`telvax run ...`、`telvax serve ...`）以及完整的输出。

也欢迎提出语言方面的想法，最好附上一小段你希望能够写出的代码作为示例。

对语法、类型系统或标准库的修改，要先在 [telvax/rfcs](https://github.com/telvax/rfcs) 中以 RFC 的形式提出：流程见它的 README。这里的 pull request 要链接到已被接受的 RFC。文档的翻译位于 `telvax/docs-pt-br`、`docs-es` 和 `docs-zh-cn`；`docs/` 中的英文文件仍然是源文件。

## 构建

需要 nightly 版 Rust；`rust-toolchain.toml` 会选择它。

```sh
cargo build --release -p telvax_cli      # target/release/telvax
cargo test --workspace                  # default build, no LLVM needed
```

可选部分：

| 特性 | 它增加什么 | 需要 |
|---|---|---|
| `--features telvax_cli/llvm` | 原生代码，`--emit ptx` | 带 MLIR 的 LLVM 22（`. scripts/llvm-env.sh`） |
| `--features telvax_cli/gpu` | `--device gpu` | 支持 Metal、Vulkan 或 DX12 的 GPU |

Python 绑定（`python/`）、VS Code 扩展（`editors/vscode`）、Wasm playground（`playground/`）和文档站点（`site/`，按版本发布，见其 README）各自有自己的 README 和测试。

## 修改如何进入 `main`

`main` 是受保护的：任何东西都不会被直接推送到它，维护者也不例外。工作在分支上进行，通过 pull request 进入 `main`，而且 pull request 必须通过所有检查（格式、clippy、Linux 和 Windows 上的测试、LLVM 和 GPU 构建、lite 构建、Python 绑定、真实浏览器中的 playground、真实 VS Code 中的扩展、Docker 镜像、`cargo deny` 和密钥扫描）才能合并。合并采用 squash，所以 pull request 的标题会成为提交信息：保持简短，并使用祈使句（`feat: add ...`）。

```sh
git switch -c fix/short-name
# ...edit, commit...
git push -u origin fix/short-name
gh pr create --fill
gh pr merge --auto --squash      # merges by itself once every check is green
```

你在工作时仍然可以在本地运行这些检查（见下文），但算数的是通过检查的 pull request。

## 发送 pull request 之前

```sh
cargo fmt --all
cargo clippy --workspace --all-targets
cargo test --workspace
cargo deny check
```

* 文档中的代码是经过测试的（`cli/tests/docs.rs`）：```` ```telvax ```` 代码块必须通过 `telvax check` 和 `telvax fmt --check`，如果有 `fn main` 还会被运行。程序中只是一个片段的部分请标记为 ```` ```telvax ignore ````。`examples/` 下的文件接受同样的检查。
* 随修改一起添加测试。编译器的测试放在对应的 crate 旁边（`compiler/*/tests`），运行时的测试放在 `runtime/tests`。
* 提交信息保持简短并使用祈使句，使用现有的前缀：`feat:`、`fix:`、`docs:`、`test:`、`refactor:`、`style:`、`ci:`、`build:`、`chore:`。
* 如果修改改变了语言，请更新 `docs/guide/tour.md` 和 `CHANGELOG.md`。
* `tests/editions/` 中的程序是冻结的（见 `docs/guide/stability.md`）。修改不得让其中任何一个失败，也不得编辑其中任何一个。如果无法避免，那它就是一个破坏性修改：需要一个新的 edition 和一个 `feat!:` 标题。新的语法要有一个新的冻结程序，并在该目录的 `CHECKSUMS` 中加一行。
* `docs/design/status.md` 说明了哪些东西在真实硬件上运行过、哪些没有。请保持它的真实：不要因为某个东西能编译就把它标为可用。

## 各部分在哪里

见 README 中的表格。语言本身的简要地图：`compiler/lexer`（词法单元）、`compiler/parser`（语法）、`compiler/semantic`（类型和检查）、`runtime`（解释器）、`compiler/ir` 和 `compiler/codegen`（原生代码）、`compiler/accel`（PTX 和 StableHLO）。

新的标准库函数要放在 `compiler/semantic/src/stdlib.rs`（它的类型）和 `runtime/src/std_impl.rs`（它做什么）中。同时还要在 `compiler/semantic/src/stdlib_docs.rs` 中描述它：没有描述，测试就会失败，而 `docs/reference/std.md` 就是用 `UPDATE_REFERENCE=1 cargo test -p telvax_hir --test reference` 根据它生成的。

## 发布

`main` 会**每周自动发布一次**（周一，UTC 07:00，巴西利亚时间 04:00），自上次发布以来合并的每个 pull request 的**标题**（一条 [Conventional Commit](https://www.conventionalcommits.org)）决定版本号，所以不需要编辑任何文件。一次发布需要一个小时，并且一次发布包含这一周的所有合并：自最新标签以来**每个**提交的标题都会计入，取最大的升级幅度（先有 `feat:` 再有 `fix:`，是一次 minor 发布）。如果想更早发布，请手动运行 Release workflow：

| 标题 | 版本 | 示例 |
|---|---|---|
| `feat!: ...`，或正文中有 `BREAKING CHANGE:` 一行 | major（`1.2.3` 到 `2.0.0`；major 为 0 时，升 minor） | `feat!: rename the pipe operator` |
| `feat: ...` | minor（`0.1.2` 到 `0.2.0`） | `feat: add traits` |
| `fix:`、`perf:`、`refactor:`、`build:`、`revert:` 或没有前缀 | patch（`0.1.2` 到 `0.1.3`） | `fix: empty table in sql` |
| `docs:`、`ci:`、`test:`、`style:`、`chore:` | 不发布 | `docs: add a use case` |

在标题中加上 `[skip release]`，这次合并就不计入任何版本；它不会取消其他合并。如果想自己指定版本，请把根 `Cargo.toml` 中的 `version` 提高到最新标签之上。发布说明是该版本在 `CHANGELOG.md` 中对应的章节（如果你写了的话），后面跟着已合并 pull request 的列表。

## 安全

见 [SECURITY.md](security.md)。
