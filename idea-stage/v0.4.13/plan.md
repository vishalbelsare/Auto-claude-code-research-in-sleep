# v0.4.13 实施 plan (2026-05-24)

类型：v0.4.10/v0.4.11/v0.4.12 残留 + 收尾 release。每项都在之前的 codex audit 里讨论过了，spec 明确，**不需要 plan-level codex review**，直接进实施，**全部完成后 codex final diff review**。

## Scope (5 必做)

### Core — Rust runtime/CLI

**1. P1.D per-server MCP timeout** (medium ~2-3h)
- `ScopedMcpServerConfig` 加 `request_timeout_secs: Option<u64>` 字段
- `parse_optional_mcp_stdio_server_config()` 解析 `requestTimeoutSecs`
- `McpStdioProcess::request()` 支持 per-server timeout：参数加 `override_timeout: Option<Duration>`，调 site 改成传 per-server > env > default
- 单元测试: per-server > global env > default 三档优先级

**2. JSON-RPC notifications id-skip** (medium ~2h)
- `mcp_stdio.rs::request()` read loop 改成：读到 `id == null` (notification) 时打 stderr `notification: <method>` 然后继续读，不算 response；直到读到 `response.id == request.id` 才 return
- 单元测试: notification interspersed + final response 正确返回
- 关掉 v0.4.10 known limitation

**3. meta_opt hook deploy** (medium ~3h)
- `build.rs` bundle meta_opt: `assets/tools/meta_opt/{log_event,check_ready}.sh` 已经会被 walkdir 扫到 (实际还没 sync 过来)
- `sync_main_skills.sh` whitelist 加 `meta_opt/log_event.sh` + `meta_opt/check_ready.sh`
- `aris-cli/main.rs::run_init` 新增：从 cache 拷 `meta_opt/*.sh` 到 `~/.claude/hooks/`，且写 `~/.claude/settings.json` 的 `hooks.PostToolUse` / `hooks.UserPromptSubmit` 引用这俩脚本（如果用户没配的话）
- 单元测试: cache → hooks/ 拷贝 path + settings.json merge

### Hygiene

**4. v0.4.12 targeted regression tests** (medium ~3h, 8 个)
- sandbox.rs: `strict_mode_overrides_llm_disable`, `strict_mode_ignores_all_overrides`, `default_mode_honors_llm_disable`, `strictMode_config_parse` (1 个在 config.rs)
- usage.rs: `pricing_provider_match_distinguishes_real_vs_userdefined` (qwen3.6 / kimi-k2.5 / glm-4 / my-kimi-clone)
- openai_executor.rs: `word_match_o_series_provider_prefix` + `is_stream_options_unknown_field_error_classification`
- client.rs: `event_is_meaningful_content_classification` + `eof_after_message_start_retries`
- tools/lib.rs: `reviewer_word_match_provider_prefix`

**5. main SKILL.md Gemini source-of-truth fix** (small ~30min)
- main 分支 PR: `skills/gemini-search/SKILL.md` line 30 DEFAULT_MODEL → `auto-gemini-3`, line 161 MCP call `model: 'auto-gemini-3'`
- main 分支 PR: `skills/research-lit/SKILL.md` line 361 `model: 'auto-gemini-3'`
- Push 到 main 后 v0.4.14 sync 不再 revert

## 分工 (并行 subagent)

| Agent | 范围 | 文件 |
|---|---|---|
| **A** | P1.D + JSON-RPC notifications | `runtime/src/{config,mcp_stdio}.rs` + 这 2 项的 unit tests |
| **B** | meta_opt hook deploy | `runtime/build.rs` 不动 (walkdir 已支持) + `tools/sync_main_skills.sh` whitelist + `aris-cli/src/main.rs::run_init` + unit test |
| **C** | 8 个 v0.4.12 regression tests | `sandbox.rs`/`config.rs`/`usage.rs`/`openai_executor.rs`/`client.rs`/`tools/lib.rs` `mod tests` 块 |
| **主 agent (我)** | main 分支 Gemini SKILL.md PR | main 分支 2 个 SKILL.md (跟 aris-code 不冲突) |

## Codex review timing

- 不做 plan review（scope 都讨论过了）
- subagent 完成 + 我整合 cargo build/test 通过后 → **codex round-1 final diff review**
- 修 finding (如有) → round-2 GO
- Release commit + tag + push

## 推到 v0.4.14 的 (避免 scope 蔓延)

- #240 README 章节层级（small-medium, 文档独立 release）
- v0.5.0+ 架构 (provider abstraction, Responses API, MCP transport, sandbox 加固)
