# Patch Debug Flow

- Intent: keep `deploy` as a patch-only branch while debugging and patch generation happen in a `main` worktree.
- Ignore `./litellm-main` in git.
- If `./litellm-main` does not exist, create it with `git worktree add ./litellm-main main`.
- Do code analysis and source edits inside `./litellm-main`.
- Generate patches from `./litellm-main`, then place deploy-ready patch files in `./patches` and update `./patches/series`.
- Do not edit LiteLLM source code directly on the `deploy` patch branch.
- Do not store local-only agent docs or non-deploy artifacts in `./patches`.
- Do not write tests or run unit tests for this repo; only run syntax validation such as `python3 -m py_compile`.
- Done when source changes live in `./litellm-main`, deploy patches are updated in `./patches`, and `./litellm-main` stays ignored.

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
