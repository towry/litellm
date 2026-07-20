# Patch Debug Flow

- Intent: keep `deploy` as a patch-only branch. Source analysis / patch generation use a **source tree** that is never the `deploy` working tree itself.
- Do not edit LiteLLM source code directly on the `deploy` patch branch.
- Place deploy-ready patch files in `./patches` and update `./patches/series`.
- Do not store local-only agent docs or non-deploy artifacts in `./patches`.
- Do not write tests or run unit tests for this repo; only run syntax validation such as `python3 -m py_compile`.

## Source tree

Paths are relative to **this repository root** (the towry/litellm checkout).

**Canonical layout** (Amp Orbs and new setups): sibling BerriAI clone at `../litellm-upstream`.  
**Legacy compat only**: `./litellm-main` (gitignored) was an older local worktree workflow — still accept if present so old machines keep working; do not create it by default and do not treat it as the target layout.

| Role | Path | Notes |
|------|------|--------|
| **Canonical** | `../litellm-upstream` | BerriAI `main`; **read-only**. On orb this is the expected source tree. |
| **Legacy compat** | `./litellm-main` | Old local fork `main` worktree; gitignored. Use only if it already exists and `../litellm-upstream` is missing. |
| Optional | User-named path | Only if the prompt names another tree. |

Resolve:

1. If `../litellm-upstream` exists → use it (preferred; orb / standard).
2. Else if `./litellm-main` exists → use it (compat with prior local layout only).
3. Else if the task needs a source tree → clone BerriAI to `../litellm-upstream` (or use a user-named path). Do **not** default to `git worktree add ./litellm-main`.

Rules:

- Never commit/push in `../litellm-upstream` (or any BerriAI checkout). `origin` should contain `BerriAI/litellm`; prefer `main`.
- Deploy patches land only under `./patches` on this repo (`deploy` branch).
- Say which path you used when starting work.
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
