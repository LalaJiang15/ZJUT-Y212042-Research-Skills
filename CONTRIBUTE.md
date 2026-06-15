# Contributing to run_gan_experiment

这个文档说明如何维护 `run_gan_experiment` skill。

## 目标

这个 skill 的职责是通过 Telegram/OpenClaw 远程运行、监控、总结和记录实验。

默认定位：

```text
experiment runner + reporter
```

不是：

```text
autonomous debugger / bug fixer
```

## 重要边界

维护时必须保留这些规则：

- 真实训练前必须先预览参数并等待用户确认
- 新实验和 dry-run 必须带 `--name`
- 失败时只汇报错误，不自动修代码
- 不自动安装依赖
- 不自动修改远程环境
- 不自动改参数重试
- 不自动提交 commit
- git commit 只作为上下文，不作为默认主要对比依据
- 结果默认和本地上一条实验记录对比
- 默认中文回复

如果要改变这些边界，必须明确记录原因。

## 主要文件

```text
SKILL.md
```

详细执行规则。OpenClaw 触发 skill 时主要读取这里。

```text
skill_card.yaml
```

skill 简介卡片，用于快速了解能力、触发词和边界。

```text
WORK_SUMMARY.md
```

流程总结文档，适合人快速阅读。

```text
launch_experiment.py
train_vf.py
```

当前实验代码参考。它们不是 skill 的一部分，但 skill 会根据它们的接口和产物更新规则。

## 维护原则

### 1. 优先更新规则，不要硬编码旧假设

`launch_experiment.py` 和 `train_vf.py` 后续可能变化。

如果 launcher 参数、日志格式或输出目录变化，优先更新：

- `SKILL.md`
- `skill_card.yaml`
- `WORK_SUMMARY.md`

不要让 skill 依赖已经过时的路径或参数。

### 2. 优先机器可读产物

分析实验时优先读取：

```text
experiment_meta.json
train_log.jsonl
eval_results/metrics.json
```

只有这些不存在时，才 fallback 到：

```text
TensorBoard events
opt.txt
*.pth
val_images/
tmux/stdout logs
```

### 3. 不要编造指标

如果没有明确来源，不要填写：

- PSNR
- SSIM
- LPIPS
- FID
- NIQE

旧实验没有这些指标时，应在记录中留空，并在报告中说明原因。

### 4. 错误处理只能只读诊断

实验失败时可以做：

```bash
tail
cat
find
tmux capture-pane
git status --short
```

不能做：

```bash
git commit
pip install
conda install
修改代码
修改参数后重试
```

除非用户明确要求。

## 测试建议

修改 skill 后，建议按以下顺序测试。

### 1. dry-run 测试

在 Telegram 或 OpenClaw 中发送：

```text
dry run 一个 gamma=3 的实验
```

预期：

- 返回参数预览
- 命令包含 `--name`
- 命令包含 `--dry-run`
- 不启动真实训练

### 2. 参数解析测试

发送：

```text
跑一个 gamma=3 batch size=16 perceptual loss=2.0 的实验
```

预期解析：

```text
--vf_gamma 3.0
--batchSize 16
--lambda_perceptual 2.0
```

真实训练前必须等待确认。

### 3. 进度查询测试

发送：

```text
看一下训练进度
```

预期：

- 优先读取 `train_log.jsonl`
- 汇总 epoch、loss、val_mse、best_val_mse
- 如果没有 JSONL，再 fallback 到 tmux/stdout

### 4. 结果记录测试

发送：

```text
看看最新实验结果
```

预期：

- 按 `checkpoints/` 目录更新时间找到最新实验
- 读取机器可读产物
- 更新 `experiment_records.csv`
- 生成 `reports/YYYY-MM-DD-<experiment>.md`
- 默认和上一条记录对比

### 5. 失败边界测试

模拟或遇到失败时，预期：

- 只汇报错误摘要
- 不自动修代码
- 不自动改参数重试
- 明确告诉用户“没有自动修复”

## CSV 字段

当前推荐表头：

```csv
Date,Experiment,Preset,Overrides,PSNR,SSIM,LPIPS,FID,NIQE,ValMSE,BestValMSE,ComparedTo,DeltaSummary,Commit,GitStatus,ReportPath,OpenClawAdvice,Reflection
```

如果旧 CSV 字段较少，追加前应迁移到新表头，并保留旧数据。

## 更新 checklist

修改完成前检查：

- [ ] `SKILL.md` 的 Critical Rules 仍然在前面
- [ ] 失败处理仍然是 report-only
- [ ] 新实验仍然要求 `--name`
- [ ] 真实训练前仍然要求确认
- [ ] 结果分析优先机器可读日志
- [ ] 旧实验兼容规则仍然存在
- [ ] `skill_card.yaml` 与 `SKILL.md` 没有冲突
- [ ] `WORK_SUMMARY.md` 仍然能简短说明流程

## 建议的后续改进

- 将 `experiment_records.csv` 升级为 `.xlsx` 或 SQLite
- 增加实验标签
- 增加参数 diff
- 增加最佳实验排行榜
- 自动生成趋势图
- 让 `launch_experiment.py` 输出更完整的 JSON status
- 让评估脚本统一输出稳定格式的 `metrics.json`
