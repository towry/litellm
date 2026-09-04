# LiteLLM Patch Set

此目录仅存部署所需、相对 `main` 施加之私有补丁。

## Files

- `series`: patch 施行顺序
- `0001-router-fallback-http-errors.patch`: 保留 Router 于 HTTP 错误时继续 fallback
- `0011-budget-limiter-skip-missing-provider.patch`: budget limiter 在 `custom_llm_provider` 缺失（如 anthropic `/v1/messages` passthrough）时跳过 provider 级记账，而非 `raise` 中断整个回调
- `0020-openai-strip-images-when-supports-vision-false.patch`: 当 `model_info.supports_vision` 显式为 `false` 时，将 chat `image_url` / responses `input_image` 替换为文本占位（客户端给出的 url 原样写入，不解析 path）
- `0021-openai-stream-include-usage-empty-choices.patch`: `include_usage` 合成尾帧 `choices: []`，对齐 OpenAI 规范（BerriAI/litellm#28735 / PR #28736 未合入）
- `0022-openai-metadata-drop-internal-api-base.patch`: `enable_preview_features` 转发 OpenAI metadata 时丢掉内部 `api_base`，避免 openai-compat 上游（Umans flash）把路由戳当成请求元数据后 503

## Rules

- 勿纳仅供本地 agent 使用之文档、提示或工作流说明
- 每枚 patch 只做一事，便于冲突定位与回滚

## Regenerate

在源码分支完成改动后，可用 `git diff -- <paths...> > patches/<name>.patch` 重生补丁，并更新 `series`。
