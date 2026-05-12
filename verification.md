# verification

- 日期：2026-05-12
- 执行者：Codex

## 验证摘要

已完成 `LeakyReLU.mlu` 静态检查与提交配置检查：

- `config` 内容为 `001`，只评估 Task #001。
- `LeakyReLU.mlu` 提供 `torch::Tensor bang_func(torch::Tensor x, double negative_slope)`。
- kernel 按一维连续内存计算 `relu(x) + negative_slope * (x - relu(x))`。

## 实验服务器验证

已在实验服务器 `paas.extrotec.com:30399` 完成真实 MLU 编译与功能验证：

- `cncc --version`：`cncc v4.15.5 clang version 11.1.0`
- `torch_mlu`：`1.24.1-torch2.5.0`
- `torch.mlu.is_available()`：`True`
- 编译：`LeakyReLU.mlu` 通过 `torch_mlu.utils.cpp_extension.load` 编译链接成功。
- 随机输入 `[16, 16384]`：与 PyTorch `leaky_relu` 对比，最大绝对误差 `0.0`。
- 边界样例 `[-2.0, -0.0, 0.0, 3.0, -5.5]`：最大绝对误差 `0.0`。

## 评测错误修复

评测运行时错误为 `expected scalar type Float but found Half`，说明评测输入使用 `torch.float16`。已新增 half kernel 并在 `bang_func` 中按 dtype 分发：

- `torch.float16` 输入：最大绝对误差 `3.0517578125e-05`，满足 `1e-2`。
- `torch.float32` 输入：最大绝对误差 `0.0`。

本机仍未安装或未暴露 `cncc`/`mlucc`，因此本机无法执行 BangC 编译与 MLU 测试。
