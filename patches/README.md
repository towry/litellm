# LiteLLM Patch Set

此目录仅存部署所需、相对 `main` 施加之私有补丁。

## Files

- `series`: patch 施行顺序
- `0001-router-fallback-http-errors.patch`: 保留 Router 于 HTTP 错误时继续 fallback

## Rules

- 勿纳仅供本地 agent 使用之文档、提示或工作流说明
- 每枚 patch 只做一事，便于冲突定位与回滚

## Regenerate

在源码分支完成改动后，可用 `git diff -- <paths...> > patches/<name>.patch` 重生补丁，并更新 `series`。
