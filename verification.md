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

## Basic 第一批验证

本批完成题目：

- `002_matrix_scalar_multiplication`：`matrix_scalar_multiplication.mlu`
- `003_LogSoftmax`：`LogSoftmax.mlu`
- `070_Sqrt`：`Sqrt.mlu`

实验服务器复测结果：

| 题号 | dtype | 最大绝对误差 | 结论 |
| --- | --- | --- | --- |
| 002 | `torch.float16` | `0.001953125` | 通过 |
| 002 | `torch.float32` | `0.0` | 通过 |
| 003 | `torch.float16` | `0.0078125` | 通过 |
| 003 | `torch.float32` | `1.9073486328125e-06` | 通过 |
| 070 | `torch.float16` | `0.00048828125` | 通过 |
| 070 | `torch.float32` | `5.960464477539063e-08` | 通过 |

当前 `config` 设置为：

```text
002
003
070
```

## Basic 第二批验证

本批完成题目：

- `023_Matrix_vector_multiplication_`：`Matrix_vector_multiplication_.mlu`
- `103_MSE_Loss`：`MSE_Loss.mlu`
- `104_KL_Divergence_Loss`：`KL_Divergence_Loss.mlu`

实验服务器复测结果：

| 题号 | dtype | 最大误差/绝对误差 | 结论 |
| --- | --- | --- | --- |
| 023 | `torch.float16` | `0.0` | 通过 |
| 023 | `torch.float32` | `0.00011444091796875` | 通过 |
| 103 | `torch.float16` | `0.0007038116455078125` | 通过 |
| 103 | `torch.float32` | `0.00030684471130371094` | 通过 |
| 104 | `torch.float16` | `0.0001233816146850586` | 通过 |
| 104 | `torch.float32` | `8.761882781982422e-06` | 通过 |

当前 `config` 设置为：

```text
023
103
104
```

## Basic 第三批验证

本批完成题目：

- `034_Argmax_over_a_dimension`：`Argmax_over_a_dimension.mlu`
- `051_cumsum`：`cumsum.mlu`
- `075_TopK`：`TopK.mlu`

实验服务器复测结果：

| 题号 | dtype | 指标 | 结论 |
| --- | --- | --- | --- |
| 034 | `torch.float16` | indices 完全一致 | 通过 |
| 034 | `torch.float32` | indices 完全一致 | 通过 |
| 051 | `torch.float16` | 最大误差 `0.0` | 通过 |
| 051 | `torch.float32` | 最大误差 `0.00031280517578125` | 通过 |
| 075 | `torch.float16` | values 最大误差 `0.0`，indices 完全一致 | 通过 |
| 075 | `torch.float32` | values 最大误差 `0.0`，indices 完全一致 | 通过 |

当前 `config` 设置为：

```text
034
051
075
```

## Basic 第四批验证

本批完成题目：

- `039_BatchNorm`：`BatchNorm.mlu`
- `056_gather`：`gather.mlu`
- `109_Scatter_add`：`Scatter_add.mlu`

实验服务器复测结果：

| 题号 | dtype | 指标 | 结论 |
| --- | --- | --- | --- |
| 039 | `torch.float16` | 最大误差 `0.0` | 通过 |
| 039 | `torch.float32` | 最大误差 `2.384185791015625e-07` | 通过 |
| 056 | `torch.float16` | 最大误差 `0.0` | 通过 |
| 109 | `torch.float16` | 输出为 float32，最大误差 `0.0` | 通过 |
| 109 | `torch.float32` | 最大误差 `0.0` | 通过 |

当前 `config` 设置为：

```text
039
056
109
```
