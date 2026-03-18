# Patch Debug Flow

- Intent: keep `deploy` as a patch-only branch while debugging and patch generation happen in a `main` worktree.
- Ignore `./litellm-main` in git.
- If `./litellm-main` does not exist, create it with `git worktree add ./litellm-main main`.
- Do code analysis and source edits inside `./litellm-main`.
- Generate patches from `./litellm-main`, then place deploy-ready patch files in `./patches` and update `./patches/series`.
- Do not edit LiteLLM source code directly on the `deploy` patch branch.
- Do not store local-only agent docs or non-deploy artifacts in `./patches`.
- Done when source changes live in `./litellm-main`, deploy patches are updated in `./patches`, and `./litellm-main` stays ignored.
