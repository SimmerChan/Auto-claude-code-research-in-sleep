---
name: run-experiment
description: Deploy and run ML experiments on local or remote Ascend NPU servers. Use when user says "run experiment", "deploy to server", "跑实验", or needs to launch training jobs.
argument-hint: [experiment-description]
allowed-tools: Bash(*), Read, Grep, Glob, Edit, Write, Agent
---

# Run Experiment

Deploy and run ML experiment on Ascend NPU: $ARGUMENTS

## Workflow

### Step 1: Detect Environment

Read the project's `CLAUDE.md` to determine the experiment environment:

- **Local NPU**: Look for local Ascend NPU setup info
- **Remote server**: Look for SSH alias, conda env, code directory

If no server info is found in `CLAUDE.md`, ask the user.

### Step 2: Pre-flight Check

Check NPU availability on the target machine:

**Remote:**
```bash
ssh <server> npu-smi info
```

**Local:**
```bash
npu-smi info
```

Free NPU = memory.used < 500 MiB.

### Step 3: Sync Code (Remote Only)

Check the project's `CLAUDE.md` for a `code_sync` setting. If not specified, default to `rsync`.

#### Option A: rsync (default)

Only sync necessary files — NOT data, checkpoints, or large files:
```bash
rsync -avz --include='*.py' --exclude='*' <local_src>/ <server>:<remote_dst>/
```

#### Option B: git (when `code_sync: git` is set in CLAUDE.md)

Push local changes to remote repo, then pull on the server:
```bash
# 1. Push from local
git add -A && git commit -m "sync: experiment deployment" && git push

# 2. Pull on server
ssh <server> "cd <remote_dst> && git pull"
```

Benefits: version-tracked, multi-server sync with one push, no rsync include/exclude rules needed.

### Step 3.5: W&B Integration (when `wandb: true` in CLAUDE.md)

**Skip this step entirely if `wandb` is not set or is `false` in CLAUDE.md.**

Before deploying, ensure the experiment scripts have W&B logging:

1. **Check if wandb is already in the script** — look for `import wandb` or `wandb.init`. If present, skip to Step 4.

2. **If not present, add W&B logging** to the training script:
   ```python
   import wandb
   wandb.init(project=WANDB_PROJECT, name=EXP_NAME, config={...hyperparams...})

   # Inside training loop:
   wandb.log({"train/loss": loss, "train/lr": lr, "step": step})

   # After eval:
   wandb.log({"eval/loss": eval_loss, "eval/ppl": ppl, "eval/accuracy": acc})

   # At end:
   wandb.finish()
   ```

3. **Metrics to log** (add whichever apply to the experiment):
   - `train/loss` — training loss per step
   - `train/lr` — learning rate
   - `eval/loss`, `eval/ppl`, `eval/accuracy` — eval metrics per epoch
   - `npu/memory_used` — NPU memory (via `torch_npu.npu.max_memory_allocated()`)
   - `speed/samples_per_sec` — throughput
   - Any custom metrics the experiment already computes

4. **Verify wandb login on the target machine:**
   ```bash
   ssh <server> "wandb status"  # should show logged in
   # If not logged in:
   ssh <server> "wandb login <WANDB_API_KEY>"
   ```

> The W&B project name and API key come from `CLAUDE.md` (see example below). The experiment name is auto-generated from the script name + timestamp.

### Step 4: Deploy

#### Remote (via SSH + screen)

For each experiment, create a dedicated screen session with NPU binding:
```bash
ssh <server> "screen -dmS <exp_name> bash -c '\
  eval \"\$(<conda_path>/conda shell.bash hook)\" && \
  conda activate <env> && \
  ASCEND_VISIBLE_DEVICES=<npu_id> python <script> <args> 2>&1 | tee <log_file>'"
```

#### Local

```bash
# Linux with Ascend NPU
ASCEND_VISIBLE_DEVICES=<npu_id> python <script> <args> 2>&1 | tee <log_file>
```

For local long-running jobs, use `run_in_background: true` to keep the conversation responsive.

### Step 5: Verify Launch

**Remote:**
```bash
ssh <server> "screen -ls"
ssh <server> "npu-smi info"
```

**Local:**
Check process is running and NPU is allocated.

### Step 6: Feishu Notification (if configured)

After deployment is verified, check `~/.claude/feishu.json`:
- Send `experiment_done` notification: which experiments launched, which NPUs, estimated time
- If config absent or mode `"off"`: skip entirely (no-op)

## Key Rules

- ALWAYS check NPU availability first — never blindly assign NPUs
- Each experiment gets its own screen session + NPU (remote) or background process (local)
- Use `tee` to save logs for later inspection
- Run deployment commands with `run_in_background: true` to keep conversation responsive
- Report back: which NPU, which screen/process, what command, estimated time
- If multiple experiments, launch them in parallel on different NPUs

## CLAUDE.md Example

Users should add their server info to their project's `CLAUDE.md`:

```markdown
## Remote Server
- SSH: `ssh my-npu-server`
- NPU: 4x Ascend 910B (32GB each)
- Conda: `eval "$(/opt/conda/bin/conda shell.bash hook)" && conda activate research`
- Code dir: `/home/user/experiments/`
- code_sync: rsync          # default. Or set to "git" for git push/pull workflow
- wandb: false              # set to "true" to auto-add W&B logging to experiment scripts
- wandb_project: my-project # W&B project name (required if wandb: true)
- wandb_entity: my-team     # W&B team/user (optional, uses default if omitted)

## Local Environment
- Linux with Ascend NPU
- Conda env: `ml` (Python 3.10 + PyTorch + CANN)
```

> **W&B setup**: Run `wandb login` on your server once (or set `WANDB_API_KEY` env var). The skill reads project/entity from CLAUDE.md and adds `wandb.init()` + `wandb.log()` to your training scripts automatically. Dashboard: `https://wandb.ai/<entity>/<project>`.
