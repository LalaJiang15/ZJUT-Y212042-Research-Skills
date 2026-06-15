# run_gan_experiment 工作流程总结

这个 skill 的目标是：通过 Telegram 让 OpenClaw 远程启动实验、查看进度、汇总结果，并在本地记录每次实验。

## 1. 发起实验

用户在 Telegram 里发送自然语言请求，例如：

```text
跑一个 gamma=3 batch size=16 的实验
```

OpenClaw 会解析出：

- 实验名：自动生成并传入 `--name`
- preset：没有指定时使用 launcher 默认值
- 临时参数：如 `--vf_gamma 3.0`、`--batchSize 16`
- 是否 dry-run 或 sweep

## 2. 参数预览与确认

真实训练前，OpenClaw 先返回预览：

```text
Experiment Preview
- name:
- preset:
- overrides:
- remote command:
```

用户确认后才会启动训练。

## 3. 远程启动训练

确认后，OpenClaw 通过 SSH 连接远程服务器：

```text
gpu / 10.12.42.23
```

进入项目目录：

```text
/home/asus/zpm/EnlightenGAN-master
```

然后在 `tmux` 中运行：

```bash
python launch_experiment.py <ARGS>
```

训练结束后，tmux session 通常自动释放，日志和实验产物会保留。

## 4. 查看进度

用户询问进度时，OpenClaw 会优先读取：

```text
checkpoints/<experiment>/train_log.jsonl
```

从中提取：

- 当前 epoch
- loss
- `val_mse`
- `best_val_mse`
- 是否完成或失败

如果没有机器日志，再 fallback 到 tmux/stdout 日志。

## 5. 汇总实验结果

用户说：

```text
看看最新实验结果
```

OpenClaw 会按 `checkpoints/` 下目录更新时间找到最新实验，并读取：

```text
experiment_meta.json
train_log.jsonl
eval_results/metrics.json
val_images/
checkpoint 文件
```

新实验优先用机器可读日志和 `metrics.json`；旧实验则兼容 `opt.txt`、TensorBoard event、`latest.pth`、`val_images/` 等产物。

没有来源的指标不会编造，会留空。

## 6. 本地记录

每次总结后，OpenClaw 会更新本地：

```text
experiment_records.csv
```

主要记录：

- 实验名
- 参数改动
- 指标
- `val_mse` / `best_val_mse`
- 默认和上一条实验的对比
- OpenClaw 建议
- 值得注意的现象

同时生成一份 Markdown 报告：

```text
reports/YYYY-MM-DD-<experiment>.md
```

## 7. 失败处理

如果实验报错，OpenClaw 只做只读诊断并汇报错误。

默认不会：

- 修代码
- 改参数重试
- 安装依赖
- 修改环境
- 自动提交 commit

失败时只返回错误摘要和日志位置，等待用户明确指示。

## 8. 核心原则

- 先预览，再确认，再训练
- 新实验必须带 `--name`
- 优先读取机器可读日志
- 结果默认和上一条本地记录对比
- git commit 只作为参考
- 失败只汇报，不自动修复
