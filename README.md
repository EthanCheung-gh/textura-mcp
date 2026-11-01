# @textura/mcp

Textura 的独立 MCP 工具库：通过桥接端口操作工作区，不依赖 React 与具体运行时。

本仓库是从 Textura 主仓库中抽取出来的等价实现，目录结构与源文件保持一致；抽取过程只做搬移与打包化，
没有新增工具、没有改动工具语义，也没有引入 stdio / 网络传输入口。

## 这个库是什么

- 提供 7 个 MCP 工具的定义与调用分发（`TOOL_DEFINITIONS`、`listToolDefinitions`、`callTool`），
  以及与工具语义一一对应的 `TexturaBridge` 端口（`listItems` / `searchItems` / `createItem` /
  `updateItem` / `setStatus` / `createEdge` / `getStats`）。
- `TexturaMcpServer` 实现 JSON-RPC 2.0 的一个子集：`initialize`、`ping`、`tools/list`、`tools/call`，
  协议版本固定为 `2024-11-05`，服务名 `textura-mcp`。
- `createInProcessClient(server)` 提供进程内客户端，省掉序列化与传输层，宿主可以直接
  `await client.callTool(...)`。
- 工作区读写不在本库内：所有副作用都落在宿主实现的 `TexturaBridge` 上。本库只负责参数校验、
  结果整形与协议编解码，因此可以同时被 Tauri 桌面前端、Node 脚本或其他运行时复用。

依赖面很小：运行时零依赖，只有 `typescript` 与 `vitest` 两个 devDependencies，不含 React、zustand
或 Textura 主仓库的任何模块。

## 安装与构建

```bash
npm install        # 安装 typescript / vitest
npm run typecheck  # tsc -p tsconfig.json --noEmit
npm run build      # tsc -p tsconfig.json，产物在 dist/
npm test           # pretest 先跑测试类型检查，再执行 vitest run
npm run clean      # 删除 dist/
```

构建产物：

- `dist/index.js` 与 `dist/index.d.ts`：包入口（ESM）；
- `dist/{types,errors,tools,server,client}.js|.d.ts`：与 `src/` 一一对应；
- 测试文件不参与构建（`tsconfig.json` 中 `exclude: ["src/**/*.test.ts"]`），也不进入发布包；
  测试的类型检查由 `npm test` 的 `pretest` 阶段完成（`tsc -p tsconfig.test.json`），
  需要单独执行时用 `npm run typecheck:test`。

发布相关字段：`files` 限定为 `dist`、`src`、`!src/**/*.test.ts`、`README.md`、`LICENSE`——
`src` 保留给使用方查阅源码，测试文件用取反模式从 tarball 里排除（`npm pack --dry-run` 共 21 个文件）。
这里没有用根 `.npmignore`：存在 `files` 白名单时根 `.npmignore` 不参与排除，写了也不生效，
所以排除规则只放在 `files` 里，保持单一来源。
`publishConfig.access` 为 `public`，用于把 `@textura/mcp` 这类 scope 包按公开包发布（不发布时可以忽略）。

### 作为本地依赖被宿主消费

宿主把本库当作普通 npm 依赖即可，例如在宿主的 `package.json` 中：

```json
{
  "dependencies": {
    "@textura/mcp": "file:../textura-mcp"
  }
}
```

宿主位于 `.../sandbox/Textura`、本库位于同级的 `.../sandbox/textura-mcp` 时，`file:../textura-mcp`
就指向本仓库。注意两点：

1. 本包的 `main` / `types` / `exports` 都指向 `dist/`，宿主消费的是**构建产物**而不是 `src/`。所以宿主首次
   安装前、以及本库源码改动后，都需要执行 `npm run build`（或在宿主侧用 `predev` / `prebuild` / `pretest`
   钩子代为构建，Textura 当前就是这样接的），否则宿主解析到的是过期的 `dist`（甚至不存在）。注意这条只在
   **重新执行** dev / build / test 时才会自动重建：正在运行的 vite dev 进程不会自己感知库源码变化，
   改完源码需要重启该进程（或让钩子重新构建）才能看到新行为；只改 `src/` 而不再构建，
   已经启动的宿主进程看到的仍然是旧的 `dist`。
2. `file:` 依赖在 npm 7+ 下以符号链接方式安装（`node_modules/@textura/mcp` 直接指向本仓库），所以本库
   重新 `npm run build` 后宿主立即看到新的 `dist`，通常不必重新 `npm install`；需要重新安装的只是首次接入，
   以及宿主侧把 `file:` 依赖拷成实体目录（而非链接）的情况。

也可以把本仓库作为 npm workspace 或 `npm link` 的另一端使用，导入方式不变：`import ... from '@textura/mcp'`。

## 快速上手

宿主只需实现 `TexturaBridge` 的 7 个方法。下面是一个内存版最小实现：

```ts
import {
  TexturaMcpServer,
  createInProcessClient,
  type CreateEdgeInput,
  type CreateItemInput,
  type ListItemsQuery,
  type McpItem,
  type TexturaBridge,
  type UpdateItemPatch,
} from '@textura/mcp';

const items = new Map<string, McpItem>();
let seq = 0;

function update(id: string, patch: UpdateItemPatch): McpItem | null {
  const current = items.get(id);
  if (!current) return null;
  const next: McpItem = { ...current, ...patch, updatedAt: new Date().toISOString() };
  items.set(id, next);
  return next;
}

const bridge: TexturaBridge = {
  listItems(query: ListItemsQuery = {}) {
    let list = [...items.values()];
    if (query.kind) list = list.filter((item) => item.kind === query.kind);
    if (query.status) list = list.filter((item) => item.status === query.status);
    if (query.parentId !== undefined) {
      list = list.filter((item) => item.parentId === query.parentId);
    }
    return list.slice(0, query.limit ?? 50);
  },
  searchItems(query, limit = 50) {
    const needle = query.toLowerCase();
    return [...items.values()]
      .filter((item) => item.title.toLowerCase().includes(needle))
      .slice(0, limit);
  },
  createItem(input: CreateItemInput) {
    seq += 1;
    const now = new Date().toISOString();
    const item: McpItem = {
      id: `item-${seq}`,
      kind: input.kind,
      title: input.title,
      notes: input.notes ?? '',
      status: input.status ?? 'next',
      parentId: input.parentId ?? null,
      areaId: input.areaId ?? null,
      tags: input.tags ?? [],
      createdAt: now,
      updatedAt: now,
    };
    items.set(item.id, item);
    return item;
  },
  updateItem: update,
  setStatus(id, status) {
    return update(id, { status });
  },
  createEdge(input: CreateEdgeInput) {
    return { id: `edge-${input.from}-${input.to}` };
  },
  getStats() {
    const all = [...items.values()];
    const byKind: Record<string, number> = {};
    const byStatus: Record<string, number> = {};
    for (const item of all) {
      byKind[item.kind] = (byKind[item.kind] ?? 0) + 1;
      byStatus[item.status] = (byStatus[item.status] ?? 0) + 1;
    }
    return { items: all.length, byKind, byStatus };
  },
};

const server = new TexturaMcpServer(bridge);
const client = createInProcessClient(server);

await client.initialize();
const tools = await client.listTools(); // 7 个工具定义
const result = await client.callTool('create_item', {
  kind: 'action',
  title: '整理季度复盘',
  tags: ['复盘'],
  status: 'next',
});

console.log(tools.length, result.structuredContent);
```

如果宿主已经有自己的 JSON-RPC 传输，也可以跳过客户端，直接调用协议入口：

```ts
const response = server.handle({
  jsonrpc: '2.0',
  id: 1,
  method: 'tools/call',
  params: { name: 'get_stats', arguments: {} },
});
```

`new TexturaMcpServer(bridge, { name, version })` 可以覆盖 `initialize` 返回的 `serverInfo`
（默认 `{ name: 'textura-mcp', version: '0.1.0' }`）。

## 工具清单

| 工具 | 入参要点 | 语义与返回 |
| --- | --- | --- |
| `list_items` | 可选 `kind`（9 种事项类型）、`status`（8 种状态）、`parentId`（字符串或 `null`）、`limit`（1–200） | 按类型、状态、父级过滤后返回 `{ count, items }`。`parentId: null` 表示只看顶层事项；`limit` 超过 200 或小于 1 直接报参数错误。 |
| `search_items` | `query`（必填、非空字符串）、可选 `limit`（1–200） | 在标题、备注与标签中搜索关键词，返回 `{ count, items }`；不传 `limit` 时使用 `DEFAULT_LIMIT`（50）。 |
| `create_item` | `kind`、`title` 必填；可选 `notes`、`parentId`（字符串或 `null`）、`status`、`tags`（字符串数组）、`areaId`（字符串或 `null`） | 新增事项，返回 `{ item }`。`tags` 会逐项 trim 并丢弃空串；`parentId` / `areaId` 传空字符串等价于 `null`。 |
| `update_item` | `id` 必填；`patch` 必填，至少包含一个可更新字段：`title`、`notes`、`status`、`tags`、`areaId`、`parentId` | 局部更新，返回 `{ item }`；`patch` 为空对象时报参数错误；事项不存在时返回 `isError: true` 与 `未找到事项：<id>`。`areaId` / `parentId` 传 `null` 表示清空。 |
| `set_status` | `id`、`status` 必填 | 设置事项状态，返回 `{ item }`；事项不存在时返回 `isError: true` 与 `未找到事项：<id>`。按工具契约，切到 `done` 时应同步完成时间、切离 `done` 时清空完成时间——这一写入由宿主的 `TexturaBridge.setStatus` 负责，本库只做参数校验与结果整形。 |
| `link_items` | `from`、`to` 必填；可选 `type`（`okr` / `flow` / `depends` / `related`）、`label` | 建立关联边，返回 `{ edge }`（桥接端口返回 `{ id }`）。 |
| `get_stats` | 无参数 | 统计工作区事项总数与按类型、状态的分布，返回 `{ stats }`。 |

枚举取值：

- 事项类型 `ITEM_KINDS`：`objective`、`keyresult`、`project`、`action`、`sop`、`step`、`milestone`、`note`、`reference`
- 事项状态 `ITEM_STATUSES`：`inbox`、`next`、`active`、`waiting`、`scheduled`、`someday`、`done`、`dropped`
- 关联边类型 `EDGE_TYPES`：`okr`、`flow`、`depends`、`related`
- 分页上限：`MAX_LIMIT = 200`，`DEFAULT_LIMIT = 50`

成功结果同时给出两份等价内容：`content[0].text` 是 JSON 字符串（2 空格缩进），
`structuredContent` 是对应的原始对象，宿主可以按需要取用。

## 错误与协议约定

两层错误要分开看：

- 工具层（`ToolInputError`）：参数非法时**不**抛 JSON-RPC 错误，而是返回工具结果
  `{ content: [{ type: 'text', text: <错误信息> }], isError: true }`。`update_item` / `set_status`
  找不到事项时也走这条路径。桥接端口内部抛出的其它异常同样被收敛为 `isError: true` 的工具结果。
- 协议层（`McpError`）：以 JSON-RPC `error` 对象返回。

| 情况 | 返回 |
| --- | --- |
| `initialize` | `{ protocolVersion: '2024-11-05', capabilities: { tools: { listChanged: false } }, serverInfo }` |
| `ping` | `{}` |
| `tools/list` | `{ tools: ToolDefinition[] }`（每次返回副本，外部修改不会污染内部定义） |
| `tools/call`，工具名未知 | JSON-RPC 错误 `-32602`（`未知工具：<name>`） |
| `tools/call`，缺少 `params` 对象或工具名 | JSON-RPC 错误 `-32602` |
| 未知方法 | JSON-RPC 错误 `-32601`（`未知方法：<method>`） |
| `jsonrpc` 非 `'2.0'` 或 `method` 非字符串 | JSON-RPC 错误 `-32600`（`Invalid Request`） |
| 其它未预期异常 | JSON-RPC 错误 `-32603` |
| 请求没有 `id`（通知） | `handle()` 返回 `null`，不产生响应 |

`createInProcessClient` 会把 JSON-RPC 错误转成抛出的 `McpError`（带 `code`），
因此 `await client.request('resources/list')` 会以 `code: -32601` 拒绝；工具层的 `isError`
不会被转成异常，仍作为正常结果返回。

## 模块与产物策略

- 纯 ESM：`package.json` 中 `"type": "module"`，`exports` 只声明 `types` 与 `import` 两个条件，
  没有 CommonJS 入口。Node 侧请用 `import`（或 `await import()`）加载。
- 源码内的相对导入统一带 `.js` 扩展名（如 `import { callTool } from './tools.js'`），
  这是为了让 `tsc` 以 `module: ESNext` + `moduleResolution: bundler` 编译出的 `dist/*.js`
  在纯 Node ESM 下可以直接解析——`tsc` 不会替无扩展名的相对导入补 `.js`，而 Node ESM 要求显式扩展名。
  源文件本身仍是普通 TypeScript，`npm test` 与 `npm run typecheck` 不受影响。
- 编译配置：`target` / `lib` 为 ES2022，`strict`、`noUnusedLocals`、`noUnusedParameters`、
  `noFallthroughCasesInSwitch`、`verbatimModuleSyntax` 开启，`declaration: true`，
  `outDir: dist`、`rootDir: src`，测试文件从构建中排除。
- 冒烟验证方式（构建后）：

  ```bash
  node --input-type=module -e "import('@textura/mcp').then((m) => console.log(m.TOOL_DEFINITIONS.length, m.MCP_SERVER_NAME))"
  ```

  在包内执行时也可以直接 `import('./dist/index.js')` 验证产物可加载。

## 边界说明

- 不负责网络传输：没有 HTTP / SSE / WebSocket 实现，`handle()` 只做一次请求-响应编解码，
  传输层由宿主提供（进程内直接调用、自定义 IPC 或未来的 stdio 封装都可以）。
- 不负责持久化：不读写文件、不管理备份，数据全部落在宿主实现的 `TexturaBridge` 上。
- 不启动 stdio 服务、不提供可执行入口：本包没有 `bin` 字段，定位是被宿主 `import` 的库。
- 不做鉴权、限流、并发控制与日志，这些属于宿主职责。
- 协议只覆盖 `initialize` / `ping` / `tools/list` / `tools/call`；`resources/*`、`prompts/*`、
  采样等能力不在范围内，调用会得到 `-32601`。

## 许可

Apache-2.0，见 [LICENSE](./LICENSE)。
