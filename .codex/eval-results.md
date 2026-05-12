# eval-results

- 日期：2026-05-12
- 执行者：Codex

## 汇总

| 题号 | 题目 | 提交状态 | 实验服务器验证 | 远程评测评论 | 当前结论 |
| --- | --- | --- | --- | --- | --- |
| 001 | LeakyReLU | 已提交 | 通过 | 用户确认通过，上榜 | 通过 |
| 002 | matrix_scalar_multiplication | 已提交于 `88b0b66` | 通过 | API 已读取 | 通过 |
| 003 | LogSoftmax | 已提交于 `88b0b66` | 通过 | API 已读取 | 通过 |
| 070 | Sqrt | 已提交于 `88b0b66` | 本地正数样例通过 | API 已读取 | 失败，待修复 |
| 023 | Matrix_vector_multiplication | 已提交于 `f9ccaec` | 小规模通过 | API 已读取 | 失败，待修复 |
| 103 | MSE_Loss | 已提交于 `f9ccaec` | 通过 | API 已读取 | 通过 |
| 104 | KL_Divergence_Loss | 已提交于 `f9ccaec` | 通过 | API 已读取 | 通过 |
| 034 | Argmax_over_a_dimension | 已提交于 `037e7af` | 通过 | API 已读取 | 通过 |
| 051 | cumsum | 已提交于 `037e7af` | 小规模通过 | API 已读取 | 失败，待修复 |
| 075 | TopK | 已提交于 `037e7af` | 通过 | API 已读取 | 通过 |

## 04c41ea - Task 001 修复提交

- commit: https://github.com/Ace-Eternal/op_retirement/commit/04c41ea
- config: `001`
- 评论状态：公开 GitHub 页面显示 `0 commit comments`
- 外部确认：用户说明 Task 001 已通过；排行榜截图显示队伍已上榜。

| 题号 | 题目 | 结果 | 阶段 | 摘要 |
| --- | --- | --- | --- | --- |
| 001 | LeakyReLU | 通过 | 远程评测 | 用户确认已通过 |

### 已知历史失败

```text
[RUNTIME ERROR] BangC 推理失败: expected scalar type Float but found Half
```

处理结果：新增 half kernel，并在 `bang_func` 中按 dtype 分发。实验服务器复测 `torch.float16` 最大绝对误差 `3.0517578125e-05`，`torch.float32` 最大绝对误差 `0.0`。

## 88b0b66 - Basic 第一批

- commit: https://github.com/Ace-Eternal/op_retirement/commit/88b0b66d34a95fc59a2e94040240a2f4bc5ea946
- config: `002,003,070`
- 评论状态：已通过 GitHub REST API 从实验服务器出口读取到 `kernel-competition-bot` 评论。

API 请求：

```text
GET https://api.github.com/repos/Ace-Eternal/op_retirement/commits/88b0b66d34a95fc59a2e94040240a2f4bc5ea946/comments
Accept: application/vnd.github+json
User-Agent: Codex
```

本机出口曾触发 GitHub 未认证限流：

```text
API rate limit exceeded for 185.220.236.2.
```

随后改用实验服务器出口请求同一 API，返回 `STATUS 200`，评论数量 `1`。

| 题号 | 题目 | 结果 | 阶段 | 摘要 |
| --- | --- | --- | --- | --- |
| 002 | matrix_scalar_multiplication | 通过 | 远程评测 | score `0.3674045992632595`，diff `0.0015916824340820312`，latency `22368.8 us` |
| 003 | LogSoftmax | 通过 | 远程评测 | score `0.0009985939796766153`，diff `0.00390625`，latency `25035.2 us` |
| 070 | Sqrt | 失败 | 远程评测 | score `0.0`，diff `NaN`，latency `55.8 us` |

### 远程评测报告摘要

```text
| `002_matrix_scalar_multiplication` | 0.367 | PASS (diff=1.59e-03) | 22368.800 us | ✅ |
| `003_LogSoftmax` | 0.001 | PASS (diff=3.91e-03) | 25035.200 us | ✅ |
| `070_Sqrt` | 0.000 | FAIL (diff=nan) | 55.800 us | ❌ |

汇总: 提交3题，通过2题
```

### 070 失败日志

```text
@@RESULT@@{"passed": false, "max_abs_diff": NaN, "torch_us": 27.9, "bangc_us": 55.8, "score": 0.0}
```

失败原因判断：`070_Sqrt` 的题目输入来自 `torch.randn(batch_size, dim)`，包含负数。实验服务器之前只测了正数输入，因此未覆盖 NaN 情况。远程平台对 NaN diff 判失败。后续修复应使负数输入行为与平台参考比对逻辑兼容，优先在实验服务器复现 `torch.randn` 全量输入。

## 未提交改动统计

当前本地存在第二批未提交题目：

| 题号 | 文件 | 状态 |
| --- | --- | --- |
| 023 | `Matrix_vector_multiplication_.mlu` | 已实现；实验服务器小规模 half/float 测试通过 |
| 103 | `MSE_Loss.mlu` | 已实现；初测因 kernel 启动参数失败，已改为 4 task 并限制 `taskId==0`，尚需复测 |
| 104 | `KL_Divergence_Loss.mlu` | 已实现；初测因 kernel 启动参数失败，已改为 4 task 并限制 `taskId==0`，尚需复测 |

第二批初测失败摘要：

```text
CN_ERROR_INVALID_VALUE
cnrtInvokeKernel: Launch kernel failed.
```

推断原因：规约 kernel 使用 `dim={1,1,1}` 搭配 `cnrtFuncTypeUnion1`，远端运行环境不接受该启动配置。已改为 `dim={4,1,1}`，只让 `taskId==0` 执行规约。

第二批复测结果：

| 题号 | dtype | 最大误差/绝对误差 | 结论 |
| --- | --- | --- | --- |
| 023 | `torch.float16` | `0.0` | 通过 |
| 023 | `torch.float32` | `0.00011444091796875` | 通过 |
| 103 | `torch.float16` | `0.0007038116455078125` | 通过 |
| 103 | `torch.float32` | `0.00030684471130371094` | 通过 |
| 104 | `torch.float16` | `0.0001233816146850586` | 通过 |
| 104 | `torch.float32` | `8.761882781982422e-06` | 通过 |

## Basic 第三批本地验证

| 题号 | 题目 | 提交状态 | 实验服务器验证 | 远程评测评论 | 当前结论 |
| --- | --- | --- | --- | --- | --- |
| 034 | Argmax_over_a_dimension | 待提交 | 通过 | 未提交无评论 | 待提交 |
| 051 | cumsum | 待提交 | 通过 | 未提交无评论 | 待提交 |
| 075 | TopK | 待提交 | 通过 | 未提交无评论 | 待提交 |

实验服务器复测结果：

| 题号 | dtype | 指标 | 结论 |
| --- | --- | --- | --- |
| 034 | `torch.float16` | indices 完全一致 | 通过 |
| 034 | `torch.float32` | indices 完全一致 | 通过 |
| 051 | `torch.float16` | 最大误差 `0.0` | 通过 |
| 051 | `torch.float32` | 最大误差 `0.00031280517578125` | 通过 |
| 075 | `torch.float16` | values 最大误差 `0.0`，indices 完全一致 | 通过 |
| 075 | `torch.float32` | values 最大误差 `0.0`，indices 完全一致 | 通过 |

## 037e7af - Basic 第三批

- commit: https://github.com/Ace-Eternal/op_retirement/commit/037e7af38fbf12c11227988fd86980e3dd0e4754
- config: `034,051,075`
- 评论状态：已通过 GitHub REST API 从实验服务器出口读取到 `kernel-competition-bot` 评论。

| 题号 | 题目 | 结果 | 阶段 | 摘要 |
| --- | --- | --- | --- | --- |
| 034 | Argmax_over_a_dimension | 通过 | 远程评测 | score `0.0023620746613809054`，diff `0.0`，latency `12108.0 us` |
| 051 | cumsum | 失败 | 远程评测 | score `0.0`，diff `0.0625152587890625`，latency `11949.6 us` |
| 075 | TopK | 通过 | 远程评测 | score `0.017699115044247787`，diff `0.0`，latency `949.2 us` |

### 远程评测报告摘要

```text
| `034_Argmax_over_a_dimension` | 0.002 | PASS (diff=0.00e+00) | 12108.000 us | ✅ |
| `051_cumsum` | 0.000 | FAIL (diff=6.25e-02) | 11949.600 us | ❌ |
| `075_TopK` | 0.018 | PASS (diff=0.00e+00) | 949.200 us | ✅ |

汇总: 提交3题，通过2题
```

### 051 失败日志

```text
@@RESULT@@{"passed": false, "max_abs_diff": 0.0625152587890625, "torch_us": 37.7, "bangc_us": 11949.6, "score": 0.0}
```

失败原因判断：实验服务器验证使用 `torch.allclose(..., atol=1e-2, rtol=1e-2)`，但远程平台看最大绝对误差，half 路径长序列前缀和累计误差达到 `0.0625`。后续修复应复现全量 half 输入并调整累加/输出策略。

## Basic 第四批本地验证

| 题号 | 题目 | 提交状态 | 实验服务器验证 | 远程评测评论 | 当前结论 |
| --- | --- | --- | --- | --- | --- |
| 039 | BatchNorm | 待提交 | 通过 | 未提交无评论 | 待提交 |
| 056 | gather | 待提交 | 通过 | 未提交无评论 | 待提交 |
| 109 | Scatter_add | 待提交 | 通过 | 未提交无评论 | 待提交 |

实验服务器复测结果：

| 题号 | dtype | 指标 | 结论 |
| --- | --- | --- | --- |
| 039 | `torch.float16` | 最大误差 `0.0` | 通过 |
| 039 | `torch.float32` | 最大误差 `2.384185791015625e-07` | 通过 |
| 056 | `torch.float16` | 最大误差 `0.0` | 通过 |
| 109 | `torch.float16` | 输出为 float32，最大误差 `0.0` | 通过 |
| 109 | `torch.float32` | 最大误差 `0.0` | 通过 |

## Basic 第五批本地验证

| 题号 | 题目 | 提交状态 | 实验服务器验证 | 远程评测评论 | 当前结论 |
| --- | --- | --- | --- | --- | --- |
| 005 | average_pooling_2d | 待提交 | 通过 | 未提交无评论 | 待提交 |
| 009 | conv_standard_1D | 待提交 | 通过 | 未提交无评论 | 待提交 |
| 111 | Masked_select | 待提交 | 通过 | 未提交无评论 | 待提交 |

实验服务器复测结果：

| 题号 | dtype | 指标 | 结论 |
| --- | --- | --- | --- |
| 005 | `torch.float16` | 最大误差 `0.0` | 通过 |
| 005 | `torch.float32` | 最大误差 `0.0` | 通过 |
| 009 | `torch.float16` | 输出为 float32，最大误差 `9.5367431640625e-07` | 通过 |
| 009 | `torch.float32` | 最大误差 `9.5367431640625e-07` | 通过 |
| 111 | `torch.float16` | shape 一致，最大误差 `0.0` | 通过 |
| 111 | `torch.float32` | shape 一致，最大误差 `0.0` | 通过 |

## Basic 第六批本地验证

| 题号 | 题目 | 提交状态 | 实验服务器验证 | 远程评测评论 | 当前结论 |
| --- | --- | --- | --- | --- | --- |
| 004 | batched_matrix_multiplication | 待提交 | 小尺寸通过 | 未提交无评论 | 待提交 |
| 012 | conv_transposed_2D__asymmetric_input__square_kernel | 待提交 | 小尺寸通过 | 未提交无评论 | 待提交 |
| 135 | Dilated_conv_2D | 待提交 | 小尺寸通过 | 未提交无评论 | 待提交 |

实验服务器小尺寸复测结果：

| 题号 | dtype | 指标 | 结论 |
| --- | --- | --- | --- |
| 004 | `torch.float16` 输入，float32 输出 | 最大误差 `0.0` | 通过 |
| 012 | `torch.float16` 输入，float32 输出 | 最大误差 `9.5367431640625e-07` | 通过 |
| 135 | `torch.float16` 输入，float32 输出 | 最大误差 `0.0` | 通过 |

## 43a4d81 - Basic 第四批

- commit: https://github.com/Ace-Eternal/op_retirement/commit/43a4d81194c7125f30cada68775c11e8d2f93fba
- config: `039,056,109`
- 评论状态：已通过 GitHub REST API 从实验服务器出口读取到 `kernel-competition-bot` 评论。

| 题号 | 题目 | 结果 | 阶段 | 摘要 |
| --- | --- | --- | --- | --- |
| 039 | BatchNorm | 通过 | 远程评测 | score `0.0019665160501577757`，diff `0.0019540786743164062`，latency `2711699.2 us` |
| 056 | gather | 通过 | 远程评测 | score `1.8259259259259257`，diff `0.009765625`，latency `54.0 us` |
| 109 | Scatter_add | 通过 | 远程评测 | score `8.470761994514519e-05`，diff `2.384185791015625e-07`，latency `828733.0 us` |

### 远程评测报告摘要

```text
| `039_BatchNorm` | 0.002 | PASS (diff=1.95e-03) | 2711699.200 us | ✅ |
| `056_gather` | 1.826 | PASS (diff=9.77e-03) | 54.000 us | ✅ |
| `109_Scatter_add` | 0.000 | PASS (diff=2.38e-07) | 828733.000 us | ✅ |

汇总: 提交3题，通过3题
```

## f9ccaec - Basic 第二批

- commit: https://github.com/Ace-Eternal/op_retirement/commit/f9ccaec54ed3f58878c5033b4c4b87942d179fdd
- config: `023,103,104`
- 评论状态：已通过 GitHub REST API 从实验服务器出口读取到 `kernel-competition-bot` 评论。

| 题号 | 题目 | 结果 | 阶段 | 摘要 |
| --- | --- | --- | --- | --- |
| 023 | Matrix_vector_multiplication_ | 失败 | 远程评测 | score `0.0`，diff `0.2408447265625`，latency `450401.4 us` |
| 103 | MSE_Loss | 通过 | 远程评测 | score `0.0001332952489764829`，diff `8.71419906616211e-05`，latency `189054.0 us` |
| 104 | KL_Divergence_Loss | 通过 | 远程评测 | score `0.00230584859777113`，diff `3.4332275390625e-05`，latency `57159.0 us` |

### 远程评测报告摘要

```text
| `023_Matrix_vector_multiplication_` | 0.000 | FAIL (diff=2.41e-01) | 450401.400 us | ❌ |
| `103_MSE_Loss` | 0.000 | PASS (diff=8.71e-05) | 189054.000 us | ✅ |
| `104_KL_Divergence_Loss` | 0.002 | PASS (diff=3.43e-05) | 57159.000 us | ✅ |

汇总: 提交3题，通过2题
```

### 023 失败日志

```text
@@RESULT@@{"passed": false, "max_abs_diff": 0.2408447265625, "torch_us": 938.2, "bangc_us": 450401.4, "score": 0.0}
```

失败原因判断：实验服务器只用较小 `K=4096` 做过验证；远程评测使用题目全尺寸 `K=131072`，朴素 float 累加与参考结果误差扩大到 `0.2408`。后续修复应在实验服务器按全尺寸复现，考虑分块规约顺序、累加精度和 bfloat16 来源数据。
