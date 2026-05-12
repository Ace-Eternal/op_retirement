# AGENTS.md — OpenOperator BangC 算子比赛操作手册

本文件面向 Codex / AI 开发代理，用于在本仓库中持续完成 OpenOperator BangC 算子题目。执行者应优先遵循本文档，再参考 `README.md` 和具体题目说明。

## 1. 比赛目标与评测方式

本仓库用于提交 OpenOperator 算子比赛代码。每道题要求实现一个 BangC 算子，并通过统一入口函数暴露给评测脚本。

评测关注两件事：

1. 正确性：输出结果与 PyTorch 参考实现误差不大于 `1e-2`。
2. 性能：正确性通过后，按 BangC 硬件执行时间相对 PyTorch 的速度计算分数。

排行榜字段含义：

- `RANK`：当前排名。
- `USER`：队伍名和 GitHub 仓库。
- `SCORE`：性能分数，越高越好。
- `LATENCY`：延迟，越低越好。
- `DATE`：该最好成绩产生时间。

排行榜不是实时逐秒更新，通常 commit 评论先出现，排行榜按周期刷新并只记录团队历史最好成绩。

## 2. 仓库提交格式

根目录必须包含：

```bash
.
├── config          # 指定本次评估哪些题目
├── LeakyReLU.mlu   # 题目同名 BangC 代码文件
├── ...             # 其他题目的 .mlu 文件
└── README.md       # 可选说明
```

`config` 每行写一个三位题目编号。例如 Task #001：

```text
001
```

注意事项：

- 只评测少量题目，避免一次 push 触发过多评测。
- 如果没有 `config`，评测系统可能默认评估所有题目。
- 系统只接收 `main` 分支 push。
- 代码文件名必须与题目名称一致，这是评测脚本定位文件的依据。

## 3. 代码入口约定

每个 `.mlu` 文件必须自包含：

- 必要头文件引用。
- `__mlu_entry__` kernel 定义。
- 外部调用入口 `bang_func`。

入口函数格式来自题目说明，例如 Task #001：

```cpp
torch::Tensor bang_func(torch::Tensor x, double negative_slope);
```

通用要求：

- 函数名必须是 `bang_func`。
- 返回值必须是 `torch::Tensor`。
- 参数必须覆盖题目输入张量和 `__init__` 中的初始化参数。
- 输出 shape 应与参考 PyTorch 实现一致。
- 对输入先执行 `contiguous()`，避免非连续输入造成地址访问错误。
- 不要假设评测输入一定是 `float32`，必须检查并支持实际 dtype。

## 4. 推荐开发流程

每个新题按以下顺序推进：

1. 阅读题目说明，确认输入、输出、dtype、shape、初始化参数和 PyTorch 参考实现。
2. 修改 `config`，只保留当前题号。
3. 创建或更新题目同名 `.mlu` 文件。
4. 本地静态检查函数签名、文件名、`config`。
5. 上传实验服务器，先编译，再跑数值对比。
6. 确认 `float16` 和 `float32` 输入是否都能运行。
7. 提交到 GitHub `main` 分支。
8. 查看 commit 评论日志，若失败，先根据日志在实验服务器复现。
9. 通过后再查看快捷监控和排行榜。

常用提交命令：

```bash
git add <题目文件.mlu> config verification.md .codex
git commit -m "implement <OperatorName> bangc operator"
git push origin main
```

## 5. 实验服务器验证流程

实验服务器用于真实 MLU 编译与运行验证。连接地址：

```bash
ssh root@paas.extrotec.com -p 30399
```

不要把密码写入仓库文件、脚本或提交记录。需要自动化连接时，使用交互输入、临时环境变量或本地私有凭据管理。

远端常用环境：

```bash
export NEUWARE_HOME=/usr/local/neuware
export PATH=$NEUWARE_HOME/bin:$PATH
export LD_LIBRARY_PATH=$NEUWARE_HOME/lib64:$NEUWARE_HOME/lib:$LD_LIBRARY_PATH
source /torch/venv3/pytorch/bin/activate
```

已确认环境信息：

- `cncc v4.15.5 clang version 11.1.0`
- `torch_mlu 1.24.1-torch2.5.0`
- `torch.mlu.is_available()` 返回 `True`

推荐用 `torch_mlu.utils.cpp_extension.load` 编译 `.mlu` 文件，并用临时 `binding.cpp` 暴露 `bang_func` 做功能测试。临时测试文件不要提交到仓库。

## 6. Task #001 LeakyReLU 经验记录

题目要求：

```cpp
torch::Tensor bang_func(torch::Tensor x, double negative_slope);
```

参考实现：

```python
torch.nn.functional.leaky_relu(x, negative_slope=negative_slope)
```

输入规模：

- `batch_size = 16`
- `dim = 16384`
- 输入 shape 为 `[16, 16384]`

当前实现策略：

- 将输入视为一维连续数组。
- 多 core 按 `taskId` 切分元素区间。
- NRAM 分块处理。
- 使用公式：

```cpp
LeakyReLU(x) = relu(x) + negative_slope * (x - relu(x))
```

关键问题与修复：

1. 编译失败：`torch_mlu::getCurMLUStream()` 未声明。
   - 原因：缺少 torch_mlu stream 头文件。
   - 修复：添加 `#include "framework/core/MLUStream.h"`。

2. 评测运行失败：`expected scalar type Float but found Half`。
   - 原因：评测输入实际为 `torch.float16`，原实现只调用 `data_ptr<float>()`。
   - 修复：新增 half kernel，在 `bang_func` 中按 `input.scalar_type()` 分发 `Half` 和 `Float`。

实验服务器复测结果：

- `torch.float16`：最大绝对误差 `3.0517578125e-05`，满足 `1e-2`。
- `torch.float32`：最大绝对误差 `0.0`。
- Task #001 已通过远程评测并上榜。

## 7. dtype 与内存注意事项

评测可能使用 half 输入，即使题目示例里没有明确写 dtype。实现时至少考虑：

- `at::ScalarType::Half`
- `at::ScalarType::Float`

half 指针转换模式：

```cpp
reinterpret_cast<const half *>(input.data_ptr<at::Half>())
reinterpret_cast<half *>(output.data_ptr<at::Half>())
```

float 指针模式：

```cpp
input.data_ptr<float>()
output.data_ptr<float>()
```

分块对齐经验：

- float：64 个元素对齐，对应 256 字节。
- half：128 个元素对齐，对应 256 字节。
- 写回时只写真实 `len`，不要写 `aligned_len`。

## 8. 日志与文档

每次完成题目后更新：

- `verification.md`：记录题目验证摘要、服务器环境、测试结果、失败修复。
- `.codex/testing.md`：记录本地和远端验证命令及结果。
- `.codex/operations-log.md`：记录关键决策、失败原因、修复过程。
- `.codex/review-report.md`：记录审查结论和残余风险。

不要在这些文件中写入密码、token、webhook secret 或其他敏感凭据。

## 9. 排错优先级

遇到失败时按顺序判断：

1. 文件名是否与题目名称一致。
2. `config` 是否包含正确题号。
3. `bang_func` 签名是否与题目完全一致。
4. 是否缺少 `framework/core/MLUStream.h` 等必要头文件。
5. 输入 dtype 是否为 half，是否错误调用 `data_ptr<float>()`。
6. 输入是否非连续，是否已执行 `contiguous()`。
7. NRAM 分块是否越界，对齐长度和写回长度是否区分。
8. 输出 dtype 和 shape 是否与输入或参考实现一致。
9. 精度是否满足 `1e-2`。
10. 性能瓶颈是否来自过小分块、过少 core、重复搬运或不必要类型转换。

## 10. 后续优化方向

正确性优先，性能优化在通过评测后进行。常见优化方向：

- 增大单次处理元素数，但不能超过 NRAM 容量。
- 使用更合适的 taskDim / Union 类型。
- 避免全局内存多次读写。
- 尽量使用 BangC 向量指令组合，减少标量循环。
- 针对固定 shape 做简化，但不要破坏题目可能的边界输入。

每次优化后必须重新在实验服务器验证 `float16` 和 `float32` 正确性，再 push 远程评测。
