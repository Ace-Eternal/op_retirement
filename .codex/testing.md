# testing

- 日期：2026-05-12
- 执行者：Codex

## 本地验证

| 命令 | 结果 | 说明 |
| --- | --- | --- |
| `Get-Content -Raw config` | 通过 | 内容为 `001`，本次提交只请求评估 Task #001。 |
| `git diff -- LeakyReLU.mlu config` | 通过 | diff 仅修改 `LeakyReLU.mlu`，`config` 未改变。 |
| `Get-Command cncc` | 未通过 | 本机未发现 BangC 编译器命令。 |
| `Get-Command mlucc` | 未通过 | 本机未发现备用 MLU 编译器命令。 |

## 风险说明

当前本机缺少寒武纪 BangC 编译与运行环境，无法在本机执行真实单元测试、烟雾测试和性能测试。

## 实验服务器验证

服务器：`paas.extrotec.com:30399`

环境设置：

```bash
export NEUWARE_HOME=/usr/local/neuware
export PATH=$NEUWARE_HOME/bin:$PATH
export LD_LIBRARY_PATH=$NEUWARE_HOME/lib64:$NEUWARE_HOME/lib:$LD_LIBRARY_PATH
source /torch/venv3/pytorch/bin/activate
```

| 验证项 | 结果 | 说明 |
| --- | --- | --- |
| `cncc --version` | 通过 | `cncc v4.15.5 clang version 11.1.0`。 |
| `torch.mlu.is_available()` | 通过 | 返回 `True`。 |
| `torch_mlu.utils.cpp_extension.load(..., sources=['LeakyReLU.mlu'])` | 通过 | `LeakyReLU.mlu` 可编译并链接为 `.so`。 |
| `[16, 16384]` 随机输入功能测试 | 通过 | 与 `torch.nn.functional.leaky_relu` 对比，`MAX_ABS 0.0`，`ALLCLOSE_1E_2 True`。 |
| 负数、零、正数边界样例 | 通过 | `SAMPLE_MAX_ABS 0.0`。 |

远端测试使用临时 `binding.cpp` 暴露 `bang_func`，该文件仅用于验证，不属于提交内容。
