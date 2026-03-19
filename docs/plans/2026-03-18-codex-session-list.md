# Codex 全量会话命令收获记录

**日期：** 2026-03-18

## 背景结论

- 现有 `/list` 不是“读取所有 Codex 会话”。
- `core` 侧只是调用当前项目/当前工作区 agent 的 `ListSessions()`。
- Codex agent 的原实现会扫描本机 `~/.codex/sessions` 或 `CODEX_HOME/sessions`，但会按当前 `work_dir` 过滤，所以本质上列的是“当前项目对应的 Codex 会话”。

## 本次改动

- 在 `core` 新增可选能力接口 `GlobalSessionLister`，避免在 `core` 硬编码 `codex` agent 名称。
- 在 `agent/codex` 增加 `ListAllSessions()`，复用原有扫描逻辑，但取消 `work_dir` 过滤。
- 新增命令 `/codex-session-list [page]`：
  - 列出本机全部 Codex transcript
  - 支持分页
  - 输出短 session ID，便于人工排查
  - 明确提示这是“全局诊断列表”，`/switch` 仍只作用于当前项目会话
- 同步更新：
  - `core/i18n.go`
  - `docs/usage.md`
  - `docs/usage.zh-CN.md`
  - `core/engine_test.go`

## 这次的关键收获

- 这个需求的核心不是单纯“再加一个命令”，而是先把“当前项目会话”和“本机全量会话”两个语义拆开。
- 由于项目架构明确要求 `core` 不感知具体 agent，所以正确做法不是在 `core` 里写 `if agent.Name() == "codex"`，而是补一个 capability interface。
- `/codex-session-list` 不能复用 `/list` 的“可切换”语义，否则会误导用户；因为全量列表里的很多会话并不属于当前 `work_dir`。

## 验证结果

- 通过：
  - `env GOCACHE=/tmp/go-build GOTMPDIR=/tmp go test ./core -run 'TestCmd(List_UsesLegacyTextOnPlatformWithoutCardSupport|CodexSessionList_UnsupportedAgent|CodexSessionList_Success)'`
  - `env GOCACHE=/tmp/go-build GOTMPDIR=/tmp go build ./...`
- 受沙箱限制未完成：
  - `go test ./core/...`
  - `go test ./agent/codex/...`
- 原因：
  - 部分现有测试会使用 `httptest` 监听本地端口，在当前沙箱下会触发 `bind: operation not permitted`。

## 运行态注意点

- 当前 `~/.cc-connect/daemon.json` 记录的 daemon 二进制仍指向全局 pnpm 安装版本：
  - `/Users/admin/Library/pnpm/global/5/.pnpm/cc-connect@1.2.1/node_modules/cc-connect/bin/cc-connect`
- 这意味着即使仓库代码已经修改完成，直接重启现有 daemon 也不会加载这次仓库改动。
- 要让新命令真正在线生效，需要：
  1. 用当前仓库代码构建新的 `cc-connect` 二进制
  2. 用该二进制重新 `daemon install --force --work-dir ~/.cc-connect`
  3. 再执行 daemon restart

## 相关文件

- `core/interfaces.go`
- `agent/codex/codex.go`
- `agent/codex/list.go`
- `core/engine.go`
- `core/i18n.go`
- `core/engine_test.go`
- `docs/usage.md`
- `docs/usage.zh-CN.md`
