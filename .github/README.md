# CI 提交标记

在最新一条 Git commit 消息的标题或正文中，使用独立的 `--skip-tests` 或
`--sanitize` 标记控制 Linux x64 CI 测试。它们是提交消息标记，不是 `git commit`
自身的选项，例如：

```bash
git commit -m "fix: native class constants --sanitize"
git commit -m "docs: clarify configuration --skip-tests"
```

Push 和手动运行检查所选提交的消息；PR 检查源分支的最新提交消息，而不是 GitHub
自动生成的合并提交消息。只读取最新一条消息，不扫描整个历史。标记需与其他文本
以空白分隔，可放在正文中。

## `--skip-tests`

- 跳过 Linux x64 的 PHPUnit、EXT/LIB 和 PHPT 测试作业。
- 编译作业仍然执行，包括 Linux、macOS 和 Windows 的构建及已有构建冒烟检查。
- 发布 tag 始终执行测试，忽略 `--skip-tests`。
- 普通分支或 PR 同时带两个标记时，`--skip-tests` 优先，不运行 ASan PHPT。

## `--sanitize`

- 保留普通测试，额外运行 Linux x64 PHP 8.4/8.5 ZTS 的 ASan PHPT 作业。
- 每个原生 PHPT 产物在编译、链接时通过 `--sanitize address` 启用
  AddressSanitizer；仅设置环境变量或传入测试工具旧有的 `--asan` 不会给产物插桩。
  编译时同时启用 `--debug`，方便报告定位到生成代码。
- 禁用 Zend 私有分配器（`USE_ZEND_ALLOC=0`），让地址检查能看到单次堆分配。
  启用遇错终止，关闭泄漏检查；这里检查地址错误，不把它当作独立的泄漏检查作业。
- 每次运行先执行一个真实 C++ 堆越界探针，验证 ASan 已启用且报告会让 PHPT 失败。
  报告不能被宽泛的 `EXPECTF`、`XFAIL` 或测试中的环境覆盖隐藏。
- 失败时保存测试日志、ASan 报告和生成的 C++/头文件；ASan 作业的 artifact 名带
  `-asan` 后缀，与普通测试区分。探针证据单独存入 `asan-canary-*` artifact。
- ASan 作业复用普通的 bootstrap 编译器及 PHP/PHPX 依赖，对每个 PHPT 生成的原生
  代码插桩；不表示 PHP/PHPX 的所有预编译代码也经过插桩。
- 发布包只来自普通测试作业，ASan 测试不会重复打包或上传发布产物。

CI 使用 runner 的逻辑 CPU 数作为 PHPT worker 数；多个 worker 并行时，每个测试的
嵌套编译及 Make/CMake/Cargo 环境保持单任务，避免并发相乘。顺序运行时，编译器
使用 `PHP_COMPILER_JOBS` 配置，未配置则使用 1。用户机器仍须遵守 `AGENTS.md` 的资源限制。

## 手动运行 PHPT

```bash
php run-tests.php -q -j1 --sanitize address tests/compiler/native-class
php run-tests.php -q -j1 --sanitize address,undefined tests/compiler/native-class
```

`run-tests.php --sanitize <list>` 接受 `address`、`undefined` 或两者的逗号列表，
不支持与 `--no-aot` 或 WASM 后端组合。参数会传递给并行 worker 和每个测试的编译命令，
测试自身的 `--AOT_ARGS--` 不能覆盖所要求的 sanitizer。原有 `--asan` 仅设置 PHP
测试环境；要检查 TypePHP 生成代码，请使用新的 `--sanitize address`。
