# Patch Debug Flow

- Intent: keep `deploy` as a patch-only branch. Source analysis / patch generation use a **source tree** that is never the `deploy` working tree itself.
- Do not edit LiteLLM source code directly on the `deploy` patch branch.
- Place deploy-ready patch files in `./patches` and update `./patches/series`.
- Do not store local-only agent docs or non-deploy artifacts in `./patches`.
- Do not write tests or run unit tests for this repo; only run syntax validation such as `python3 -m py_compile`.

## Source tree

Paths are relative to **this repository root** (the towry/litellm checkout).

Patches target **this fork's `origin/main`** (`towry/litellm` `main`). That is the same baseline CI uses.

CI (`.github/workflows/check-patches.yml`) checks out the patch branch, detaches to `origin/main`, restores `patches/` + `scripts/`, then runs `scripts/apply.sh` on that tree. Rebase and apply against that SHA only.

| Role | Path | Notes |
|------|------|--------|
| **Canonical** | `../litellm-main` | Detached worktree of **this repo's** `origin/main`. Create with `git fetch origin main && git worktree add --detach ../litellm-main origin/main`. |
| **Legacy compat** | `./litellm-main` | Gitignored in-repo worktree. Use only if it already exists and is this repo's `origin/main`. |
| Forbidden | `../litellm-upstream` or any `BerriAI/litellm` clone | Not the patch baseline. Ignore it even if present. |

Resolve:

1. `git fetch origin main` and record `git rev-parse origin/main`.
2. If `../litellm-main` exists and is this repo's `origin/main` → use it.
3. Else if `./litellm-main` exists and is this repo's `origin/main` → use it.
4. Else create the sibling worktree: `git worktree add --detach ../litellm-main origin/main`.
5. Say which path and which `origin/main` SHA you used.

Hard rules:

- **Never clone `BerriAI/litellm`.** These patches are not against latest upstream.
- **Never use `../litellm-upstream`** as the source tree, even if `.agents/setup` or an old orb left it there.
- Do not create `./litellm-main` unless a sibling worktree is impossible.
- Never commit or push from the source worktree. It is read-only analysis / apply scratch.
- Deploy patches land only under `./patches` on this repo (`deploy` branch).
- Done when deploy patches are updated in `./patches`.

-------

# LiteLLM Remote Hot-Patch

快速将本地 litellm fork 改动部署至远端服务器验证，无须经 CI。

## 前置

- SSH 别名 `agent` 已配（指向 VM-0-7-ubuntu）
- 远端已有完整部署（venv 在 `/opt/litellm/.venv`）

## 步骤

### 1. 定位改动文件

本地 litellm fork 在 `litellm/litellm/`，远端对应路径：

```
/opt/litellm/.venv/lib/python3.11/site-packages/litellm/
```

### 2. SCP 上传

```bash
scp litellm/litellm/<relative-path>.py agent:/tmp/<unique-name>.py
```

### 3. 覆盖并重启

```bash
ssh agent "sudo cp /tmp/<unique-name>.py /opt/litellm/.venv/lib/python3.11/site-packages/litellm/<relative-path>.py \
  && sudo chown litellm:litellm /opt/litellm/.venv/lib/python3.11/site-packages/litellm/<relative-path>.py \
  && sudo systemctl restart litellm"
```

### 4. 验证

```bash
ssh agent "sudo systemctl is-active litellm && curl -s -o /dev/null -w '%{http_code}' http://localhost:4000/health/readiness"
```

应返回 `active` 和 `200`。若服务启动较慢，等 10-15 秒再查。

## 实例

热补丁 Anthropic transformation 与 Responses API transformation：

```bash
scp litellm/litellm/llms/anthropic/chat/transformation.py agent:/tmp/anthropic_transformation.py
scp litellm/litellm/responses/litellm_completion_transformation/transformation.py agent:/tmp/responses_transformation.py

ssh agent "sudo cp /tmp/anthropic_transformation.py /opt/litellm/.venv/lib/python3.11/site-packages/litellm/llms/anthropic/chat/transformation.py \
  && sudo cp /tmp/responses_transformation.py /opt/litellm/.venv/lib/python3.11/site-packages/litellm/responses/litellm_completion_transformation/transformation.py \
  && sudo chown litellm:litellm /opt/litellm/.venv/lib/python3.11/site-packages/litellm/llms/anthropic/chat/transformation.py \
     /opt/litellm/.venv/lib/python3.11/site-packages/litellm/responses/litellm_completion_transformation/transformation.py \
  && sudo systemctl restart litellm"
```

## 注意

- 热补丁仅供快速验证，正式部署仍须经 CI（`.github/workflows/deploy-litellm.yml`）
- `systemctl restart` 后服务需约 10-15 秒完成 Prisma 迁移与启动
- 若须同时更新 deploy 脚本/service 文件/callbacks，参照 CI 流程 scp 至 `/tmp/litellm-deploy/` 后执行 `deploy.sh`
