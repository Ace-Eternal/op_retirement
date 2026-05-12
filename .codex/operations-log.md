# operations-log

- 日期：2026-05-12
- 执行者：Codex

## 记录

1. 读取 `README.md`，确认提交格式：根目录保留 `config` 与题目同名 `.mlu` 文件，`config` 中 `001` 表示评估 LeakyReLU。
2. 读取 `LeakyReLU.mlu` 与 `config`，确认当前题目文件存在且 `config` 内容为 `001`。
3. 未发现可用的 `sequential-thinking`、`shrimp-task-manager`、`code-index` 工具入口，使用本地 shell、`rg`、`update_plan` 和 `apply_patch` 替代。
4. 修改 `LeakyReLU.mlu`：
   - 删除临时 `//test2` 标记。
   - 将 kernel 输入指针改为 `const float *`。
   - 将外部入口调整为题目契约 `torch::Tensor bang_func(torch::Tensor x, double negative_slope)`。
   - 在入口中使用 `x.contiguous()` 支持非连续输入。
   - 对空张量直接返回空输出。
   - 将 `negative_slope` 显式转换为 `float` 传入 kernel。
5. 检查本地编译工具：`cncc` 与 `mlucc` 均不可用，因此无法在本机完成 BangC 编译验证。
6. 通过 SSH 连接实验服务器 `paas.extrotec.com:30399`，确认 `/usr/local/neuware/bin/cncc` 可用，版本为 `cncc v4.15.5 clang version 11.1.0`。
7. 设置远端环境变量 `NEUWARE_HOME`、`PATH`、`LD_LIBRARY_PATH` 并启用 `/torch/venv3/pytorch`，确认 `torch_mlu 1.24.1-torch2.5.0` 且 `torch.mlu.is_available()` 为 `True`。
8. 首次远端编译发现 `torch_mlu::getCurMLUStream()` 未声明，定位到头文件 `framework/core/MLUStream.h` 并补充 include。
9. 重新远端编译 `LeakyReLU.mlu` 成功；使用临时 `binding.cpp` 执行 `[16, 16384]` 随机输入和边界样例测试，最大绝对误差均为 `0.0`。
