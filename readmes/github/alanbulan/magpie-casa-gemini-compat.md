<p align="center">
  <img src="./assets/casa-logo.jpg" width="112" height="112" alt="CASA 厂商标识" />
</p>

<h1 align="center">CASA Gemini 工具与缓存兼容</h1>

<p align="center">面向 MagPie 的 CASA Gemini 兼容插件</p>
<p align="center">
  <a href="https://usemagpie.ai/docs/plugins">MagPie 插件规范</a> ·
  <a href="./docs/架构设计.md">架构设计</a> ·
  <a href="./docs/配置说明.md">配置说明</a> ·
  <a href="./docs/客户端接入.md">客户端接入</a> ·
  <a href="./docs/规范核对.md">规范核对</a> ·
  <a href="./docs/实测与AB.md">测试与实测</a>
</p>

CASA 的 Gemini 流式响应可能把工具名称和参数拆成不同调用，导致客户端收到空参数或无名称调用。本插件默认向 CASA 的 `/v1/chat/completions` 发送 `stream:false`，完整校验工具参数后，再按客户端请求的协议返回 JSON 或 SSE。等待期间只发送 SSE 注释心跳。

**仅处理 `casa/gemini-*`。** 其他供应商与 CASA 的 GPT、Claude 模型保持原路由。没有 GPT 降级通道。0.2.0 保留原 CASA 厂商、账户和模型名称，只将其 Gemini 端点接入由插件宿主管理的回环桥。兼容对象是当前 CASA（`apikey.casa`），厂商 LOGO 来自其官网，来源见 [资源说明](./assets/来源.json)。本项目与 MagPie、CASA 均无官方隶属关系。

## 能做什么

- Responses、Chat Completions、Anthropic Messages、Gemini Native 四种入口，共用协议无关的 IR 与缓存引擎。
- 支持声明式函数工具、并行工具、MCP 暴露的函数工具、Responses 命名空间、客户端工具搜索，以及自由文本工具的参数包装。
- 工具 Schema 深度排序、原始参数字符串透传、确定性历史调用 ID、可配置动态头冻结与后置。
- 会话指纹、短请求旁路隔离、粘性密钥与通道、TTL、有界状态和同会话串行保护。
- OpenAI 用量采用包含式，Anthropic 用量采用分离式；原生缓存计数优先，缺失时按字节 LCP 估算。
- 首业务帧前支持 502/503/404 备用 Gemini 通道切换；400 只裁剪配置匹配的末尾辅助块。
- 脱敏 Prefix Diff 日志、独立缓存 A/B 脚本与不启动 MagPie 的插件契约测试。

“工具通用”指客户端声明并执行的工具。厂商托管的 `web_search`、`file_search`、Google Search、代码执行与文档检索等需要独立执行器；插件遇到不支持的声明会明确报错。MagPie 自身提供的搜索与工具编排仍由 MagPie 负责。

## 0.2.0 更新

- 原生用量记录归属 `casa`，显示原 Gemini 密钥账户与真实模型名，不再使用内部 `__responses` 等后缀。
- 区分正文与结构化 `thinking` / `thought` / `extra_content.google.thought`；隔离开头的推理标签及带明确最终答复边界的推理前言。
- 保留真实推理 token：OpenAI 输出包含推理，Gemini 的 `candidatesTokenCount` 与 `thoughtsTokenCount` 分开，合计不重复。
- 移除逐 SSE 事件执行的路由中间件。插件页旧版“调用次数”是钩子次数，包含无关请求的空调用，不能作为付费请求数。

离线测试 87 项通过，Bun 额外复核其中 27 项。真实 Neon 合成前缀实验观察到同步输入 12045 → 12061 → 1957，随后流式明确返回 10137 个缓存 token。完整验收与局限见 [统计与正文修复](./docs/统计与正文修复.md)。

## 从 GitHub 安装与更新

兼容基线是 MagPie `0.1.4572`。先保留原 `casa` 供应商、Chat Completions 地址和名为 `gemini` 的已启用密钥。三个 Gemini 模型的 API 应在 MagPie 中设为 `gemini`。

```sh
magpie plugin add github:alanbulan/magpie-casa-gemini-compat
```

更新时可以将 GitHub 地址固定到本次提交：`magpie plugin add github:alanbulan/magpie-casa-gemini-compat#提交SHA`。MagPie 会替换同一个包并保留选项；仅重复添加未固定的地址可能沿用 Bun 锁文件中的旧提交，更新后应核对版本与提交号。

安装后执行已安装包里的配置助手。它自动备份，只更改 CASA 的 Gemini 端点和本插件选项，不启动或重启 MagPie：

```sh
node ~/.config/magpie/plugins/node_modules/magpie-casa-gemini-compat/scripts/configure-native.mjs enable
```

此文件路径用于**配置已从 GitHub 安装的包**，不是本地源代码安装。使用自定义 `XDG_CONFIG_HOME` 时调整上面的配置目录。无需再登录一个新的供应商。客户端继续选 `casa/gemini-4-carbon`、`casa/gemini-4-hydrogen` 或 `casa/gemini-4-neon`。

桥只监听 `127.0.0.1:41327`，由原有 MagPie 的 Bun 插件宿主持有，验证原 Gemini 账户密钥，停用的密钥不可使用；向上游沿用 MagPie 实际选择的账户。插件宿主退出时端口自动释放，没有系统启动项。

**停用或卸载之前，先恢复原端点**，使用启用时输出的备份目录：

```sh
node ~/.config/magpie/plugins/node_modules/magpie-casa-gemini-compat/scripts/configure-native.mjs restore /你的备份目录
magpie plugin off github:alanbulan/magpie-casa-gemini-compat
```

历史请求不迁移、不双写。旧版记录仍在 `casa-gemini-compat`，启用新版后的请求在 `casa`。插件页的钩子次数与用量页的请求数是不同指标。

## 配置与运行数据

默认配置位于 [config.json](./config.json)，严格结构见 [config.schema.json](./config.schema.json)。需要自定义时，将配置复制到插件目录之外，通过 MagPie `plugins.json` 中该插件条目的 `options.configPath` 指向它；不要改动其他插件条目。完整示例见 [配置说明](./docs/配置说明.md)。

默认是轨道 A（同步整包），每 8 秒发送 `: keep-alive` 注释。轨道 B 需要显式 `streamVerified:true`；当前 CASA 工具拆分故障下不建议启用。

默认读取 `$XDG_CONFIG_HOME/magpie/providers.json`，未设置时读取 `~/.config/magpie/providers.json`。日志位于该配置目录下的 `plugin-work/casa-gemini-compat/diagnostics.jsonl`；`options.runtimeDir` 可以改位置。日志记录哈希、计数、结构掩码与重试轨迹，不记录真实密钥、用户正文或工具参数。会话默认只在内存保存，重启后会重新冷校准；可选 `sessions.statePath` 会把会话正文写入权限受限的本地状态文件，需自行管理保留时间。

缓存 token 缺少原生计数时是**估算值**，用于维持客户端合理的上下文口径，不能替代供应商账单，也不能保证避免所有自动压缩。旁路请求识别是基于延续性和前缀锚点的启发式规则；相同前缀的无显式键会话可能共享槽位，建议客户端提供会话键。

## 开发与诊断

```sh
npm ci --ignore-scripts
npm test
npm run check:plugin
# 已有 Bun 时可核对宿主所用运行时，不需要启动 MagPie
bun test test/plugin.test.mjs
```

`npm run probe`、`npm run smoke`、`npm run ab` 会发起真实 CASA 请求，可能产生用量，均只使用合成测试内容。它们不会安装或启动 MagPie；具体参数、结果解读与客户端配置见 [文档](./docs/实测与AB.md)。另提供可选 [独立本地 HTTP 入口](./docs/客户端接入.md)，它与原厂商桥是两种独立调试入口。

反馈问题时请附 MagPie 版本、模型名、客户端与入口协议、是否使用 SSE、脱敏错误码及诊断中的重试轨迹。不要上传 `providers.json`、`plugin-auth.json`、真实对话或密钥。维护者可以先用离线夹具复现。

## 许可与鸣谢

代码采用 [MIT 许可](./LICENSE)。CASA LOGO 为第三方厂商标识，不在本项目代码许可授权范围内。感谢 [MagPie](https://github.com/yetone/magpie) 的插件与中间件接口。
