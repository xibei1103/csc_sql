# CSC-SQL 3B Baseline 复现报告

## 复现日期
2026-04-29

## 实验目标
复现 CSC-SQL 论文中 3B 模型在 BIRD dev 上的执行准确率（论文报 65.28%）

## 实验环境
- AutoDL RTX 4090D 24GB
- Driver: 550.127.05, CUDA: 12.4
- PyTorch 2.6.0+cu124
- vLLM 0.8.5
- transformers 4.49.0
- Python 3.12

## 模型
- SQL Generate: cycloneboy/CscSQL-Grpo-Qwen2.5-Coder-3B-Instruct
- SQL Merge:    cycloneboy/CscSQL-Merge-Qwen2.5-Coder-3B-Instruct

## 数据集
- BIRD dev: 1534 题（dev_20240627 版本）
- prompt 文件: dev_bird.json（来自 cycloneboy/bird_train ModelScope 数据集）

## 实验配置
- N_SQL_GENERATE = 64
- N_SQL_MERGE = 8
- Temperature = 0.8
- Eval mode: major_voting
- Eval step: sql_pipeline (跳过 table_link)

## 结果对比

| 指标 | 我的结果 | 论文 | 差距 |
|---|---|---|---|
| Stage 1 major voting (n=64) | 60.30% | - | - |
| Stage 1 pass@64 | 80.31% | - | - |
| Stage 1 major_top2@64 | 70.73% | - | - |
| **Stage 2 major voting (n=8)** ⭐ | **62.78%** | **65.28%** | **-2.50pp** |
| Stage 2 pass@8 | 64.02% | - | - |
| Stage 2 major_top2@8 | 63.82% | - | - |

## 耗时
- Stage 1 (SQL Generate, n=64): 2:51:54
- Stage 2 (SQL Merge, n=8): 13:43
- 总计: ~3:13

## 花费
约 10-11 元（含装环境、踩坑、推理、评估）

## 主要观察
1. **CSC 方法有效性验证**：Merge 阶段把 60.30% → 62.78%，提升 2.48pp
2. **pass@64 = 80.31%**：64 个候选里有 80% 包含正确答案，说明模型能力够，问题在如何"选对"
3. **major_top2@64 = 70.73%**：前 2 名候选中 70.7% 包含正确答案，是 Merge 阶段能利用的上限
4. 与论文差 2.5pp，符合论文复现的正常波动范围（vllm/transformers 版本差异、采样随机性等）

## 主要踩坑记录

### 坑 1: PyTorch + CUDA 版本不匹配
pip 默认装最新 torch 2.11+cu130，但 AutoDL 镜像 driver 只支持 CUDA 12.4。
**解决**：降级到 torch 2.6.0+cu124。

### 坑 2: vllm 0.8.5 + transformers 最新版不兼容
报错 `Qwen2Tokenizer has no attribute all_special_tokens_extended`。
**解决**：transformers 降级到 4.49.0。

### 坑 3: pipeline_infer.py 用 os.system 启动子进程，错误不传播
子进程崩溃后主脚本继续，日志显示 "finish infer step" 误导。
**改进建议**（毕设可做）：用 subprocess.run(check=True) 替代 os.system。

### 坑 4: common_utils.py:910 ZeroDivisionError
当 Stage 1 失败时（产出 0 个 SQL），Stage 2 在 normalize 时除以 0 崩溃。
**改进建议**：加保护性 if 判断。

### 坑 5: dev_bird.json 不在 BIRD 官方数据里
作者用了预处理后的 prompt 文件 dev_bird.json，原始 BIRD 数据集只有 dev.json。
**解决**：从 cycloneboy/bird_train ModelScope 数据集单独下载。

## 下一步计划
1. 跑 7B baseline 对比（论文 69.19%）
2. 深入读 nl2sql_grpo.py 训练代码，准备改进
3. 设计第一个改进点（multi-turn merge revision，对应 MTIR 思想）
4. 在小规模数据上训练第一个改进版 merge 模型

## 输出文件位置
- 服务器: `/root/autodl-tmp/outputs/20260429_191444/`
- 本地备份: `baseline_3b_core_results.tar.gz`