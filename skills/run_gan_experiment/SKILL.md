# Name: Remote GAN Experiment Runner

# Description

Use this skill when the user wants OpenClaw, especially from Telegram, to run or inspect EnlightenGAN / VF-Lens experiments on the remote GPU server and keep a local experiment log.

## Critical Rules

These rules override all lower-level workflow suggestions in this skill:

1. This skill only runs, monitors, summarizes, and records experiments by default.
2. Never edit code, patch bugs, install packages, change environments, change parameters, retry failed runs, or create commits unless the user explicitly asks for that action.
3. On any run failure, stop after collecting minimal read-only diagnostics and report the error to the user.
4. Before launching real training, always preview the exact launcher arguments and wait for user confirmation.
5. Every new run or dry-run must include `--name <name>` unless the user explicitly asks to use the launcher default name.
6. Reply in Chinese by default.
7. For result comparison, use the previous local `experiment_records.csv` row as the default baseline; treat git as context only, especially when commit messages are vague.

The current project is launched through:

```text
launch_experiment.py
```

`launch_experiment.py` calls `train_vf.py`. Treat both files as implementation references, not as permanent public APIs. Prefer discovering the current launcher interface with `python launch_experiment.py --list` and `python launch_experiment.py --help` when needed.

Current machine-readable artifacts:

- `experiment_meta.json` in each checkpoint directory.
- `machine_train_log` field inside `experiment_meta.json`.
- `train_log.jsonl`, normally at `checkpoints/<experiment>/train_log.jsonl`.
- `metrics.json`, normally under `checkpoints/<experiment>/eval_results/metrics.json`.

Legacy experiment artifacts may only include:

- `tensorboard/events.out.tfevents*`
- `opt.txt`
- `val_images/` or similarly named validation-image folders
- periodic checkpoint files such as `*_net_G.pth`, `*_net_D.pth`, epoch `.pth` files, `latest.pth`, or `vf_latest.pth`

Prefer these machine-readable files over parsing human stdout.

## Environment

- Remote host alias: `gpu`
- Remote host address: `10.12.42.23`
- Remote project path: `/home/asus/zpm/EnlightenGAN-master`
- Conda env: `vf10`
- Local skill workspace: the directory containing this `SKILL.md`
- Local records file: `experiment_records.csv`
- Local reports directory: `reports/`

Prefer `ssh gpu` when the local SSH config is available. If the alias fails, fall back to `ssh 10.12.42.23` or ask the user for the username if direct SSH requires one.

Use recoverable, non-destructive operations. Do not delete remote experiment outputs unless the user explicitly asks.

## High-Level Contract

This skill is responsible for:

- Translating the user's natural language request into launcher arguments.
- Showing key parameters before a run.
- Asking for confirmation before starting real training.
- Running the launcher remotely in a detachable session.
- Reporting launch status back to Telegram.
- Summarizing finished experiment results.
- Maintaining a local table and Markdown report for each recorded experiment.

This skill is not responsible for:

- Reimplementing training logic from `train_vf.py`.
- Hardcoding all defaults that already live in `launch_experiment.py`.
- Editing model code unless the user explicitly asks.
- Debugging, patching, or fixing experiment code after a run fails unless the user explicitly asks for a fix.

## Error Handling Boundary

This skill is an experiment runner and reporter by default, not an autonomous bug fixer.

When a remote experiment command fails, crashes, raises an exception, exits non-zero, produces CUDA/OOM/import/path/config errors, or shows traceback-like output:

- Do not edit local or remote code.
- Do not run repair commands.
- Do not change parameters and retry automatically.
- Do not create commits, patches, or workaround scripts.
- Do not install packages or modify environments.
- Capture and report the failure to the user.
- Stop and wait for explicit user instruction before taking any fixing action.

Required failure report:

```text
实验运行失败
- experiment:
- session:
- exit/status:
- error summary:
- key traceback/log lines:
- log path:
- checkpoint dir:
- 我没有自动修复代码。你要我继续排查或修复的话，请明确说。
```

It is allowed to run read-only diagnostic commands to collect error context, such as `tail`, `cat`, `find`, `tmux capture-pane`, `git status --short`, or reading `train_log.jsonl`. Keep diagnostics minimal and do not mutate files.

## Language And Telegram Style

- Reply to the user in Chinese by default.
- Keep command names, file paths, parameter names, metrics, and git hashes in their original English form.
- Use compact Telegram-friendly Chinese summaries.
- Do not use Markdown tables in Telegram replies.
- If the user writes in English, match the user's language for that conversation.

## Scenario A: Start A New Experiment

Trigger examples:

- "跑一个新实验"
- "启动训练"
- "跑 EnlightenGAN"
- "开个 baseline"
- "batch size 改成 16"
- "gamma=3"
- "试试更大的 perceptual loss"
- "跑 sweep"
- "dry run 一下"

### Step A1: Parse Intent

Extract:

- `preset`: default to launcher default if the user does not specify one.
- `name`: short snake_case experiment description. Always pass it as `--name <name>` for new runs and dry-runs unless the user explicitly asks to use the launcher default name. Let the launcher add the timestamp unless the user explicitly asks for `--no-timestamp`.
- overrides: only pass fields supported by `launch_experiment.py`.
- mode: normal run, dry-run, list presets, or result-only request.

Common natural-language mappings:

```text
batch size 16       -> --batchSize 16
perceptual loss 2.0 -> --lambda_perceptual 2.0
gamma 3             -> --vf_gamma 3.0
lr ratio 0.2        -> --d_lr_ratio 0.2
lr 1e-5             -> --lr 1e-5
epoch 50            -> --niter 50
no timestamp        -> --no-timestamp
```

Current launcher note: `seed`, `skip_eval`, `eval_image_dir`, `eval_test_image`, and `eval_json_path` exist in `ExperimentConfig` but are excluded from the current CLI override parser. Do not pass them as command-line flags unless the launcher is updated.

Name examples:

```text
gamma=3              -> gamma3
batch size 16        -> bs16
high perceptual loss -> high_perc
attention + ema      -> attn_ema
gamma sweep          -> sweep_gamma
```

Name rule:

- New experiment requests must include `--name <generated_or_user_name>`.
- If the user provides an explicit name, use that exact value after sanitizing obvious whitespace.
- If the user does not provide a name, generate one from the core change.
- Only omit `--name` when the user explicitly says to use the launcher's default name.

If a parameter is ambiguous or may not exist in the current launcher, check `python launch_experiment.py --help` remotely before proposing the command.

### Step A2: Preview And Ask For Confirmation

Before starting any real training, send a Telegram-friendly summary and ask for confirmation.

Required preview:

```text
Experiment Preview
- name: <description only; command must include --name; timestamp will be added by launcher>
- preset: <preset or launcher default>
- mode: normal | dry-run | sweep
- overrides:
  - key=value
- remote command:
  python launch_experiment.py <ARGS>
```

Ask:

```text
确认开始训练吗？回复“确认/yes/y”后我再启动。
```

If the user asked for dry-run only, confirmation is not required unless the command would still launch training.

### Step A3: Run Remotely In tmux

Because the current `launch_experiment.py` runs training in the foreground, wrap it in `tmux` from the skill side so Telegram can return immediately.

Use a session name derived from the experiment name:

```text
gan_<name>_<MMDD_HHMM>
```

Preferred command shape:

```bash
ssh gpu "cd /home/asus/zpm/EnlightenGAN-master && mkdir -p openclaw_logs && tmux new-session -d -s <SESSION> 'bash -lc \"source ~/anaconda3/etc/profile.d/conda.sh && conda activate vf10 && python launch_experiment.py <ARGS> 2>&1 | tee openclaw_logs/<SESSION>.log\"'"
```

If the conda path differs, try:

```bash
ssh gpu "bash -lc 'source ~/.bashrc && conda activate vf10 && cd /home/asus/zpm/EnlightenGAN-master && python launch_experiment.py <ARGS>'"
```

For dry-run:

```bash
ssh gpu "cd /home/asus/zpm/EnlightenGAN-master && bash -lc 'source ~/anaconda3/etc/profile.d/conda.sh && conda activate vf10 && python launch_experiment.py <ARGS> --dry-run'"
```

### Step A4: Parse Launch Output

After starting, collect enough status to report:

```bash
ssh gpu "tmux has-session -t <SESSION> && tmux capture-pane -pt <SESSION> -S -120"
```

Extract when present:

- full experiment name printed as `Experiment: ...`
- checkpoint directory
- preset
- command
- meta path
- dry-run message

### Step A5: Telegram Reply For Started Runs

Use compact Telegram-friendly text, no Markdown tables.

Required sections:

```text
Experiment Started
- experiment: <full experiment name if known, otherwise pending from launcher output>
- preset: <preset>
- session: <tmux session>
- overrides:
  - key=value

Progress
- attach: ssh gpu -t 'tmux attach -t <SESSION>'
- log: ssh gpu 'tail -f /home/asus/zpm/EnlightenGAN-master/openclaw_logs/<SESSION>.log'
```

If dry-run:

```text
Dry Run
- 本次只打印命令，没有启动训练。
```

## Scenario B: Check Progress

Trigger examples:

- "看一下训练进度"
- "现在跑到哪了"
- "看 log"
- "这个实验还在跑吗"

Steps:

1. If the user named a session or experiment, use it. Otherwise list recent sessions:

```bash
ssh gpu "tmux ls || true"
```

2. Capture recent output:

```bash
ssh gpu "tmux capture-pane -pt <SESSION> -S -160"
```

3. If the session ended, inspect the remote log:

```bash
ssh gpu "tail -n 120 /home/asus/zpm/EnlightenGAN-master/openclaw_logs/<SESSION>.log"
```

Summarize:

- running or finished
- current epoch from the latest `epoch_end` or `train_step` event in `train_log.jsonl`
- latest `val_mse` and `best_val_mse` from the latest `epoch_end` event
- latest core losses from the latest `train_step.metrics`
- warnings/errors/NAN if visible in JSONL metrics or fallback text logs
- next useful action

Preferred progress commands:

```bash
ssh gpu "cat <EXP_DIR>/experiment_meta.json"
ssh gpu "tail -n 80 <TRAIN_LOG_JSONL>"
```

Fallback to tmux/stdout only when the JSONL log path is unknown or missing.

## Scenario C: Analyze Or Record Results

Trigger examples:

- "看看最新实验结果"
- "分析一下刚刚的实验"
- "记录这次实验"
- "最新评估结果咋样"
- "总结这次实验"
- "最近哪个实验最好"

### Step C1: Locate Experiment

If the user specified an experiment name, use it. Otherwise locate the newest checkpoint by directory modification time:

```bash
ssh gpu "find /home/asus/zpm/EnlightenGAN-master/checkpoints -mindepth 1 -maxdepth 1 -type d -printf '%T@ %p\n' | sort -nr | head -1 | cut -d' ' -f2-"
```

Let:

```text
EXP_DIR=/home/asus/zpm/EnlightenGAN-master/checkpoints/<experiment>
```

Non-standard folder names are allowed. Do not require a timestamp or naming pattern. If the newest directory has no recognizable experiment artifacts, skip it and use the next newest directory.

Recognizable experiment artifacts include:

- new-format markers: `experiment_meta.json`, `train_log.jsonl`, `vf_latest.pth`, `vf_best.pth`, or `eval_results/metrics.json`
- legacy markers: `opt.txt`, `latest.pth`, any `*.pth`, `tensorboard/events.out.tfevents*`, or `val_images/*`

### Step C2: Read Remote Artifacts

Read these files when present, in this priority order:

```bash
ssh gpu "cat <EXP_DIR>/experiment_meta.json"
ssh gpu "tail -n 200 <TRAIN_LOG_JSONL>"
ssh gpu "find <EXP_DIR> -path '*eval_results*' -name 'metrics.json' -print | sort | tail -1"
ssh gpu "cat <METRICS_JSON>"
ssh gpu "find <EXP_DIR>/val_images -maxdepth 1 -type f | sort | tail -20"
```

For legacy experiments, also inspect:

```bash
ssh gpu "cat <EXP_DIR>/opt.txt"
ssh gpu "find <EXP_DIR> -maxdepth 3 -type f \( -name 'events.out.tfevents*' -o -name '*.pth' -o -path '*/val_images/*' \) | sort | tail -80"
```

Resolve `<TRAIN_LOG_JSONL>` from `experiment_meta.json.machine_train_log` first. If missing, use:

```text
<EXP_DIR>/train_log.jsonl
```

Important: metrics are expected under the experiment checkpoint directory, commonly:

```text
checkpoints/<experiment>/eval_results/metrics.json
```

Do not assume a project-root `eval_results/metrics.json`.

Parse `train_log.jsonl` as JSON Lines. Important events:

- `launch_start`: experiment name, checkpoint dir, command, config, git commit.
- `train_start`: total epochs, train size, batch size, learning rates, device.
- `train_step`: epoch, step, global step, losses in `metrics`, learning rates.
- `epoch_end`: epoch, `val_mse`, `best_val_mse`, epoch time, checkpoint paths, image tags.
- `train_end`: final `best_val_mse`, final checkpoint paths.
- `launch_train_end`: launcher return code.

Use JSONL values as the source of truth for training status, `val_mse`, `best_val_mse`, epoch count, and loss stability. Use human text logs only as fallback.

Legacy analysis rule:

- If `train_log.jsonl` and `metrics.json` are missing, do not skip the experiment if legacy artifacts exist.
- Use `opt.txt` for config and parameter reconstruction.
- Use checkpoint filenames and modification times to infer the latest saved epoch.
- Use validation images to summarize available qualitative evidence, but do not invent numeric image-quality metrics from images.
- If TensorBoard tooling is available, parse `events.out.tfevents*` for scalar summaries such as losses, `val/fixed_mse`, or any recorded metric names.
- If TensorBoard tooling is unavailable, report that numeric training curves are present but not parsed, and include the event-file path.
- If no explicit PSNR/SSIM/LPIPS/FID/NIQE file exists, leave those metric fields empty in the CSV and base the summary on available losses/checkpoints/validation images.

### Step C3: Read Git Context

Prefer the commit recorded in `experiment_meta.json`. If it exists:

```bash
ssh gpu "cd /home/asus/zpm/EnlightenGAN-master && git show --stat --oneline <COMMIT>"
ssh gpu "cd /home/asus/zpm/EnlightenGAN-master && git log -1 --pretty=format:'%H%n%s%n%b' <COMMIT>"
```

Fallback to current HEAD only if metadata has no commit:

```bash
ssh gpu "cd /home/asus/zpm/EnlightenGAN-master && git log -1 --pretty=format:'%H%n%s%n%b'"
ssh gpu "cd /home/asus/zpm/EnlightenGAN-master && git show --stat --oneline HEAD"
```

Also check whether the remote working tree was dirty if that matters:

```bash
ssh gpu "cd /home/asus/zpm/EnlightenGAN-master && git status --short"
```

Git interpretation rule:

- Treat git commit information as context, not as the main comparison axis.
- If the commit title/message is vague or meaningless, assume it may only be bug fixes or small value changes.
- For experiment comparison, primarily compare against the previous row in local `experiment_records.csv`.
- Use commit stats only to explain possible causes when the commit message or changed files clearly mention model, loss, data, evaluation, scheduler, or discriminator changes.

### Step C4: Update Local Records

Maintain `experiment_records.csv` in the local skill workspace.

Create it with this header if missing:

```csv
Date,Experiment,Preset,Overrides,PSNR,SSIM,LPIPS,FID,NIQE,ValMSE,BestValMSE,ComparedTo,DeltaSummary,Commit,GitStatus,ReportPath,OpenClawAdvice,Reflection
```

If an older CSV already exists with fewer columns, read it normally and migrate it to the current header before appending. Preserve all existing rows and leave newly added fields empty for old records.

Append one row per recorded experiment.

Guidelines:

- Use ISO date: `YYYY-MM-DD`.
- Store `Overrides` as compact `key=value; key=value`.
- Store missing metrics as empty cells.
- For legacy experiments without JSON metrics, keep PSNR/SSIM/LPIPS/FID/NIQE empty instead of guessing.
- `ComparedTo` should usually be the immediately previous local record, unless the user asks for another baseline.
- `DeltaSummary` should summarize changes versus `ComparedTo`, for example `PSNR +0.12; SSIM -0.003; BestValMSE -0.0008`.
- `OpenClawAdvice` should be short and actionable.
- `Reflection` should capture anything worth remembering: surprising behavior, instability, visual artifacts, promising direction, or reason this run matters.

Comparison rule:

- Default baseline is the previous row in `experiment_records.csv`.
- If the previous row is missing, say this is the first local record.
- If the user asks "最近哪个最好", use all local records.
- If the user asks for a named baseline, compare against that named record.
- Do not over-explain from git commit messages when the commit is uninformative.

### Step C5: Generate Local Markdown Report

Create:

```text
reports/YYYY-MM-DD-<experiment>.md
```

Report template:

```markdown
# Experiment Report

## Experiment
- name:
- date:
- preset:
- overrides:
- remote dir:

## Metrics
- PSNR:
- SSIM:
- LPIPS:
- FID:
- NIQE:
- val_mse:
- best_val_mse:

## Git
- commit:
- title:
- dirty state:
- changed files:
- interpretation weight:

## Training Log
- status:
- source:
- stability:
- latest epoch:
- latest losses:
- warnings:

## Trend
- compared with previous local record:
- metric deltas:
- best-so-far notes:

## Interpretation
- likely reason for metric changes:
- visual-quality hypothesis:

## OpenClaw Advice
- next run:
- parameters to sweep:

## Reflection
- worth remembering:
```

### Step C6: Telegram Reply For Results

Use compact Telegram-friendly text, no Markdown tables.

Required sections:

```text
Experiment Result
- experiment: <name>
- PSNR:
- SSIM:
- LPIPS/FID/NIQE if present:
- val_mse/best_val_mse if present:
- status from train_log.jsonl:

Config
- preset:
- overrides:
- commit:

Trend
- compared with previous local record:
- metric deltas:

Read
- likely explanation:
- risks:

Next
- suggested next experiment:

Recorded
- CSV: experiment_records.csv
- report: reports/YYYY-MM-DD-<experiment>.md
```

If results are missing, say exactly what is missing and where you looked. Then suggest the next diagnostic command.

## Scenario D: Compare Experiments

Trigger examples:

- "最近哪个实验最好"
- "对比这两次"
- "哪个参数方向更有希望"

Use local `experiment_records.csv` first, then fetch missing remote artifacts if needed.

Compare:

- PSNR, SSIM, LPIPS, FID, NIQE
- val_mse and best_val_mse
- preset and overrides
- commit changes only when they are meaningful
- stability notes
- OpenClaw advice and reflections

Return a short ranking and one recommended next run.

## Reliability Rules

- Always preview and confirm before launching real training.
- On run failure, report the error and stop. Never auto-fix code, change parameters, reinstall dependencies, or retry unless the user explicitly asks.
- Always quote paths and session names when they may contain special characters.
- Prefer metadata from `experiment_meta.json` over guessing.
- Prefer `train_log.jsonl` over stdout or tmux capture for progress and training-status analysis.
- Prefer the commit recorded at launch time over current HEAD.
- When git commits are vague, compare primarily against the previous local experiment record.
- If a command fails, report the failure and the exact missing artifact or permission problem.
- Do not paste huge logs into Telegram. Summarize and include the command to inspect the full log.
- Do not use Markdown tables in Telegram replies.
- Keep local CSV and reports under this skill workspace.
