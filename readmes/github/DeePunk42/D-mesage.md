# D-message

Magpie 本地插件。用 **Trace** 录制和搜索经过网关的 Chat Completions、OpenAI Responses 和 Anthropic Messages 请求与响应。用 **Repeater** 编辑消息、切换模型、查看差异并重放。每个标签独立保存草稿和运行历史。

## 安装

在已安装 Magpie 的电脑上运行：

```sh
magpie plugin add github:DeePunk42/D-mesage
```

在 Magpie 侧栏打开 **D-message**。首次打开时，插件通过 Magpie 生成并保存本机页面凭据。录制默认关闭。在 **Trace → Capture scope** 选择客户端，开启录制，再从该客户端通过 Magpie 发送请求。已在 Magpie 0.1.4520 上测试。

源码开发使用 Bun。在仓库根目录运行 `bun install --frozen-lockfile` 和 `bun test`。需要把工作目录注册为本机开发插件时，运行 `bun setup.js`。本地版与 GitHub 版使用同一个页面端口，请在同一台机器上只启用其中一个。

## 使用

1. 打开 **Trace** 查看网关记录，打开 **Repeater** 编辑或重放已录制的请求。
2. 客户端继续使用现有的 Magpie 网关地址。插件按网关提供的客户端标识捕获请求，可通过 `X-Magpie-Session` 区分会话。
3. 在 **Trace → Capture scope** 选择要录制的 Agent。收起时显示所选 Agent 的图标，展开时同时显示图标和名称。客户端候选合并 Magpie 网关的已连接 Agent 模型映射、本机网关调用历史和录制钩子观察到的客户端标识。打开选择器或通过 MCP 查询状态时刷新，成功结果缓存一分钟。网关接口暂不可用时沿用已有候选，也可以手动输入。客户端按字符顺序排列，默认取首项，开始录制后固定选择。首次安装尚无候选时可以手动添加，也可以开启录制等待首个客户端。客户端支持多选或全部本机客户端。所选客户端的所有模型都可以录制。应用范围对下一条请求生效，已开始的请求继续接收响应。重放详情中的 **Source agent** 取自原请求。列表标题显示 Agent 名，每行显示最后一条消息的真实角色和截断预览，支持按预览、客户端、会话和模型过滤。左侧搜索框旁可切换记录类型，点筛选图标可选择状态和排序。拖动分隔线调整列表与详情的宽度，双击恢复默认比例。右侧的 **Response** 默认展开回复消息，直接显示正文、解析后的 JSON 和工具调用。展开 **Full response JSON** 可查看完整响应。点 **Details** 查看协议、网关端点、会话和原请求编号。Responses 的 instructions、输入项和输出项，以及 Anthropic 的 system、消息和工具块均可直接查看；Full JSON 保留原始字段。折叠消息右侧以扳手标出工具名称；文本块右上角可切换美化或 Raw 原文。
4. 双击请求或点击 **Send to Repeater**。展开消息后可在标题行修改 role。点击正文即可原位编辑：美化视图保留解转义、缩进和高亮，Raw 视图编辑原始字符串。拖动请求与响应之间的分隔线调整比例，双击平分；聚焦分隔线后也可使用方向键。把指针移到消息分隔线的最左端（包括首尾），点击灰色 **+** 插入消息。点击垃圾桶可删除消息或内容块。键盘可用 Tab 聚焦插入点，Enter 插入。用 **{}** 编辑整条消息或块的 JSON。展开消息、切换预览和聚焦编辑器只改变当前视图。修改内容后会自动保存到本地 SQLite，**Saved** 表示草稿已写入，按 Esc 返回预览。每个标签独立支持撤销和重做（Ctrl/⌘ Z、Ctrl/⌘ Shift Z），包括增删、编辑和恢复快照。
5. 每个来源首次打开时新建标签，再次打开可继续编辑。**⋯ → Duplicate tab** 从当前草稿建立独立副本，**Rename tab** 修改名称。点击 **×** 保存并关闭；可在 **Recently closed** 中恢复。刷新或重启会恢复打开顺序、活动标签、选中的运行和编辑框中未写完的 JSON。**Delete draft** 永久删除标签草稿，已发送的运行仍能在 Trace 中找到。
6. 点击 Repeater 顶栏的模型名，可从 Magpie 按供应商分组的模型列表中搜索并选择，路由组单独列出。新标签默认使用来源请求的模型，菜单中标为 **Original**。选择随草稿保存，可撤销，也可在 **Changes** 中查看。**Send** 或 `⌘/Ctrl + Enter` 发起单次推理。每次都保存独立请求快照，结果显示在右侧。刷新或重开后，在 **Trace → All records → Repeater** 可以找回历次发送；每行带 Repeater 标记、运行编号和原请求编号。点开查看当次请求和响应，双击可从该快照继续编辑重放。选择运行可查看回答、工具调用、响应 JSON、请求快照和耗时。**Restore draft**可从任一旧请求再次编辑。发送过的运行和安装旧版时保存的实验可在运行历史中查看。模型请求按现有网关计费。

7. 清除历史时，在 **Trace** 列表底部点击 **Clear**，再点 **Clear all history** 确认。清除范围包括筛选隐藏的录制、重放以及打开和已关闭的草稿；录制开关与捕获范围继续生效。

## 数据与范围

- 点击 Trace 的搜索框可选择 `request:`、`response:` 或 `tool:` 前缀。输入关键词后按空格或 Enter 生成标签，再添加下一项；同一记录须满足全部条件。用双引号搜索含空格的短语，如 `request:"blue sky" response:ok`。点击标签可编辑，点击 × 可删除。点击结果会定位并高亮命中字段，`‹` / `›` 切换匹配位置。
- Repeater 左侧 **Request → Changes** 用统一 diff 显示当前草稿相对初始请求的增删行，未发送的修改也可查看。从 Trace 打开的录制或重放以该条记录的请求为基准，复制标签沿用同一来源。发送或在右侧切换历史运行都不会改变基准。

- 入站 hooks 保存观察点的整个 JSON 对象；JSON 的排版、HTTP headers 和最终上游 token 序列不在捕获范围内。同一轮中并发请求使用独立 ID。如果其他插件会改写请求，请留意插件运行顺序。
- 异步响应回调提供 Magpie 合并后的回复。Magpie 0.1.4520 对 Responses 流直接使用 `response.completed.response`。如果正文或工具调用只在之前的 `response.output_item.done` 事件中出现，Trace 会显示请求已完成、录制到的输出项为空，并显示网关报告的输出 token 数。原始响应 JSON 可展开查看。[网关源码补丁](patches/magpie-0.1.4520-responses-output.patch) 在终态缺少输出项时按顺序补入流中的已结束项，适用于构建该版本 Magpie。仅重载 D-message 不会应用网关补丁。
- 录制默认关闭，开关保存在本机。数据库文件为 `traces.sqlite`。macOS 默认目录为 `~/Library/Application Support/D-message/`，Windows 为 `%LOCALAPPDATA%\D-message`，Linux 为 `$XDG_DATA_HOME/D-message`（默认 `~/.local/share/D-message`）。可用 `D_MESSAGE_DATA_DIR` 指定数据目录。请求正文可能包含文件内容和凭据，按当前用户权限保存。认证请求头不会传给 hook。
- 从 1.4.0 升级并保留历史时，先停用插件，等待页面服务退出，再将原数据目录整体移动到新的默认位置。目录包含 SQLite 数据库及其可能存在的 WAL 文件，源码安装还可能包含 `page.key`。如果新目录已有数据，可通过 `D_MESSAGE_DATA_DIR` 明确选择要打开的目录。
- 模型流量一直通过原 Magpie 网关。仅界面使用本机回环服务 34891，自带鉴权；客户端继续使用网关端口。重放使用独立客户端标识和会话，按记录的协议把修改后的原生请求体发送到本机网关。Chat 使用 /v1/chat/completions，Responses 使用 /v1/responses（来自 Codex 后端路由的请求使用 /backend-api/codex/responses），Anthropic 使用 /v1/messages 和标准 anthropic-version。普通 JSON 和 SSE 均可读取。仅发送单次请求，客户端工具调用结果供查看；服务商托管的工具按该服务商的请求语义处理。Magpie 的 hook 不提供原始认证或 beta 请求头，依赖原账号、beta 标记或 previous_response_id 的请求仍受上游上下文限制。切换模型后仍按原协议发送；所选模型是否支持原请求中的字段和工具取决于网关和供应商。
- 录制超过 16 分钟仍没有完成回调，或拥有该请求的进程已退出时，Trace 显示 **Completion unconfirmed**。此时模型的实际结果未知。如果后来收到回调，Trace 会更新状态及回复。原始请求和实验快照独立存储。
- 在本目录运行 `bun test` 执行隔离的端到端和流解析测试。

## 本地 HTTP MCP

打开 **Settings → MCP server**，勾选启用，指定端口并保存。默认地址为 `http://127.0.0.1:34892/mcp`。端口可选 1024–65535，修改立即生效；绑定失败时保留旧监听。设置跨插件重启保存。MCP 与页面、模型网关各自独立，只监听 IPv4 回环地址。

点击 **Copy client config**；若客户端以表单配置，选择 **Streamable HTTP**，填写页面显示的 URL，添加 `Authorization: Bearer <访问令牌>` 请求头。持有令牌的本机客户端可以读取录制正文、切换录制并使用已有模型产生成本。令牌无需传给工具参数。

| 工具 | 输入 | 返回 |
| --- | --- | --- |
| `trace_record` | `action` 可取 `start`、`stop`、`status`、`configure`；`start` 可带 `scope`，`configure` 必须带 | 当前开关、范围、候选与游标 |
| `trace_list` | 可选 `after`（不含该游标）、`session`、`limit` | 顺序排列的摘要、下一游标、`has_more` |
| `trace_get` | `ref`，可选 `view`（summary/request/response）、`path`（JSON Pointer） | 快照内容或单个字段，含最新状态 |
| `trace_edit` | `ref` 和 `ops`（test、add、remove、replace） | 新的 `draft_ref`、源引用及变更路径 |
| `trace_repeat` | `ref`、`request_key` | 一个独立的 `run_ref`；用 trace_get 查询结果 |

例如，通过 `trace_record` 传入 `{"action":"configure","scope":{"agents":["claude","codex"]}}` 设置捕获范围。使用 `"agents":"all"` 选择全部本机客户端。只接受网关标记为 `local` 或 `magpie-key` 的 Chat、Responses 和 Anthropic Messages 请求，自动排除 D-message 自身重放。`status` 返回 `choices` 和当前 `scope`。页面和 MCP 共享设置，重启后保留。

`trace:`、`draft:`、`run:` 后接 UUID。可从任一种快照派生草稿。所有 patch 一次提交，任何操作失败都不保存；原始录制保留。数组可用 `/messages/-` 追加。初次录制用 start 返回的 cursor 作为列表的 after，随后推进到列表返回的 cursor。已知请求的完成情况按 ref 单独轮询。停止录制后已开始的请求继续接收响应。

重放示例：对已选的请求用 `trace_edit` 替换 `/messages/0/content`，将返回的 `draft_ref` 传给 `trace_repeat`，再用 `trace_get(view="response")` 查询该 run。使用同一引用与同一 request_key 重试得到同一次运行，换键会产生新推理。失败及中断的运行同样保留。工具调用结果仅展示，重放不会触发客户端执行工具。MCP 创建的独立草稿可以从原请求的 Repeater 查看，历史运行和 MCP 发起的重放可从 Trace 查看。每个 Repeater 标签的运行列表保留自己发送的重放。
