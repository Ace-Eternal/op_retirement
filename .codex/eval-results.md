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
| 023 | Matrix_vector_multiplication | 未提交 | 通过 | 未提交无评论 | 待提交 |
| 103 | MSE_Loss | 未提交 | 通过 | 未提交无评论 | 待提交 |
| 104 | KL_Divergence_Loss | 未提交 | 通过 | 未提交无评论 | 待提交 |

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
