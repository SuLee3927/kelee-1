# WeChat Fork Save Commands

这份文档说明如何给 Cyberboss 的微信侧增加两个“游戏存档”式命令：

- `/fork`：把当前 Codex 线程 fork 成一个备份线程，但微信继续停留在当前线程。
- `/forks`：查看当前微信绑定与当前 workspace 下最近 5 条 fork 记录。

目标体验是：用户可以随手保存当前对话世界线，之后像看游戏存档一样查最近分支；除非用户显式发送 `/switch <threadId>`，否则当前微信会话不会被切到 fork 出来的备份线程。

## Safety Rules

实现时必须遵守这些约束：

1. `/fork` 只读当前绑定线程 id，不写当前绑定线程 id。
2. `/fork` 不调用 `setThreadIdForWorkspace(...)`。
3. `/fork` 不调用 `clearThreadIdForWorkspace(...)`。
4. fork 成功前不写历史记录。
5. fork 成功但历史写入失败时，要告诉用户“备份已创建，但历史记录写入失败”，并显示备份线程 id。
6. 只有 `/switch <threadId>` 可以切换当前微信绑定线程。
7. 修改前建议备份涉及文件，尤其是 `app.js`、`command-registry.js`、Codex runtime adapter 文件。

## Data Model

新增一个本地 fork 历史文件，建议放在 Cyberboss state dir 下：

```text
.cyberboss/fork-history.json
```

每条记录建议包含：

```json
{
  "id": "uuid",
  "runtimeId": "codex",
  "bindingKey": "workspaceId:accountId:senderId",
  "workspaceRoot": "/absolute/workspace/root",
  "sourceThreadId": "current-thread-id",
  "forkThreadId": "forked-backup-thread-id",
  "forkedAt": "2026-06-19T06:23:10.123Z"
}
```

`/forks` 查询时按 `bindingKey + workspaceRoot + runtimeId` 过滤，并按 `forkedAt` 倒序返回最近 5 条。

## Files To Touch

推荐最小改动范围：

```text
src/core/config.js
src/core/command-registry.js
src/core/app.js
src/core/fork-history-store.js
src/adapters/runtime/codex/rpc-client.js
src/adapters/runtime/codex/index.js
docs/commands.md
test/command-cli.test.js
```

## Implementation Steps

### 1. Add Config Path

在 `src/core/config.js` 的 state file 配置附近增加：

```js
forkHistoryFile: path.join(stateDir, "fork-history.json"),
```

### 2. Add Fork History Store

新增 `src/core/fork-history-store.js`。

它需要提供：

```js
class ForkHistoryStore {
  add(record) {}
  listRecent({ bindingKey, workspaceRoot, runtimeId, limit = 5 }) {}
}
```

行为要求：

- JSON 文件不存在时返回空记录。
- `add()` 写入后保存文件。
- 至少保留最近 100 条，UI 只展示最近 5 条。
- 记录必须校验 `runtimeId`、`bindingKey`、`workspaceRoot`、`sourceThreadId`、`forkThreadId`、`forkedAt`。

### 3. Register WeChat Commands

在 `src/core/command-registry.js` 的 Workspace & Thread action group 中增加：

```js
{
  action: "thread.fork",
  summary: "Fork the current thread as a save point without switching",
  terminal: [],
  weixin: ["/fork"],
  status: "active",
},
{
  action: "thread.forks",
  summary: "Show the latest five fork save points",
  terminal: [],
  weixin: ["/forks"],
  status: "active",
},
```

可选：在 `actionEmoji()` 中给它们加图标。

### 4. Add Codex RPC Support

在 `src/adapters/runtime/codex/rpc-client.js` 增加：

```js
async forkThread({ threadId }) {
  const normalizedThreadId = normalizeNonEmptyString(threadId);
  if (!normalizedThreadId) {
    throw new Error("thread/fork requires a non-empty threadId");
  }
  return this.sendRequest("thread/fork", { threadId: normalizedThreadId });
}
```

可以用不存在的 thread id 做协议验证。正确情况下应该返回类似：

```text
no rollout found for thread id ...
```

这说明方法名和参数结构是对的，只是线程不存在。

### 5. Expose Runtime Adapter Method

在 `src/adapters/runtime/codex/index.js` 增加：

```js
async forkThread({ threadId }) {
  const runtimeClient = ensureClient();
  await this.initialize();
  const response = await runtimeClient.forkThread({ threadId });
  const forkThreadId = extractThreadId(response);
  if (!forkThreadId) {
    throw new Error("thread/fork did not return a thread id");
  }
  return {
    sourceThreadId: threadId,
    forkThreadId,
    response,
  };
}
```

注意：这里也不要写 session store。

### 6. Wire Into CyberbossApp

在 `src/core/app.js`：

1. 引入 store：

```js
const { ForkHistoryStore } = require("./fork-history-store");
```

2. constructor 中初始化：

```js
this.forkHistoryStore = new ForkHistoryStore({ filePath: config.forkHistoryFile });
```

3. `dispatchChannelCommand()` 中增加：

```js
case "fork":
  await this.handleForkCommand(normalized);
  return;
case "forks":
  await this.handleForksCommand(normalized);
  return;
```

4. 新增 `handleForkCommand(normalized)`：

流程：

```text
build bindingKey
resolve workspaceRoot
read sourceThreadId from sessionStore.getThreadIdForWorkspace(...)
if no active thread: tell user to send a normal message first
call runtimeAdapter.forkThread({ threadId: sourceThreadId, workspaceRoot })
extract forkThreadId
write fork history
reply to WeChat with source thread, backup thread, timestamp, and "/switch <forkThreadId>"
```

硬性要求：这个 handler 内不能出现 `setThreadIdForWorkspace(...)`。

5. 新增 `handleForksCommand(normalized)`：

流程：

```text
build bindingKey
resolve workspaceRoot
list forkHistoryStore.listRecent({ bindingKey, workspaceRoot, runtimeId, limit: 5 })
reply to WeChat with numbered records
```

建议返回格式：

```text
最近 5 个 Fork 存档

1. 2026-06-19 14:23
   当前：source-thread-id
   备份：fork-thread-id
   切换：/switch fork-thread-id
```

空状态：

```text
还没有 Fork 存档。发送 /fork 可以保存当前线程。
```

### 7. Update Docs

在 `docs/commands.md` 中补充：

```text
/fork
/forks
```

并说明：

```text
/fork creates a backup thread without switching the current WeChat binding
/forks shows the latest five fork backups for the current workspace
```

### 8. Add Tests

至少测试 fork history store：

- 可以新增记录。
- 按 `bindingKey + workspaceRoot + runtimeId` 过滤。
- 按时间倒序返回。
- `limit = 5` 生效。

可选测试 app handler：

- 无 active thread 时返回提示。
- runtime 不支持 fork 时返回提示。
- fork 成功后没有调用 `setThreadIdForWorkspace(...)`。
- fork 成功但历史写入失败时仍显示 forked thread id。

## Verification

推荐验证命令：

```bash
npm run check
node --test test/command-cli.test.js
```

如果跑全量测试：

```bash
node --test test/*.test.js
```

注意：某些本地环境可能已有与贴纸路径、旧 mock 或 Weixin config 相关的既有失败。判断本功能是否安全时，重点看：

- `npm run check` 是否通过。
- fork history store 测试是否通过。
- `/fork` handler 内是否没有 `setThreadIdForWorkspace(...)`。

## Runtime Support

当前这套设计依赖 Codex app-server 的 RPC：

```text
thread/fork
```

Claude Code runtime 未必支持同等 fork 能力。如果当前 runtime 没有 `runtimeAdapter.forkThread`，`/fork` 应该返回：

```text
Fork is not available for runtime: <runtimeId>
```

不要用“新线程摘要备份”冒充真正 fork。那是另一种功能，不能混在这里。

## Operational Notes

添加功能后，需要重启 Cyberboss bridge，微信命令才会生效。

用户使用方式：

```text
/fork
/forks
/switch <forkThreadId>
```

`/fork` 是保存进度，`/switch` 才是读取存档。
