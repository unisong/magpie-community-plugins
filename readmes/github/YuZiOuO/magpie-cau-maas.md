# CAU MaaS — Magpie 插件

将 [maas.cau.edu.cn](https://maas.cau.edu.cn)(中国农业大学 MaaS 平台,New API 套壳,OpenAI 兼容)接入 Magpie,支持统一身份认证登录与用量显示。

## 安装

```bash
magpie plugin add github:YuZiOuO/cau-maas   # 从 GitHub 安装
magpie plugin login cau-maas                           # 登录
magpie quota                                           # 查看用量
```

## 登录方式

1. **统一身份认证登录(推荐)**:输入学号/工号和统一身份认证(CAS)密码即可,**无需打开浏览器、无需复制粘贴**。插件在后台以纯 HTTP 请求跑完整 SSO 链(OIDC → CAS → ticket 回跳),拿到 `jwt_token` 后自动复用默认 API Key(没有则以 `magpie` 为名自动创建一个)。
2. **直接粘贴 API Key**:已有 Key 时直接使用。

> 登录成功后实际保存的凭证是 API Key:它长期有效,聊天、模型列表、用量查询均可用;`jwt_token` 仅约 3 天有效,仅作为登录过程的一次性中间态。

## 实现要点

- SSO 链见 `src/sso.ts`:`/api/auth/cau/start` → OIDC authorize → `onecas.cau.edu.cn` CAS 登录表单(密码字段为 `strEnc(学号+密码+lt)` 三重 DES,直接复用平台原版 `des.js` 在 Bun `node:vm` 沙箱中执行)→ 302 回跳链 → `Set-Cookie: jwt_token`。
- CAS 密码仅在登录请求期间存在于内存,不落盘。

## 功能

- **聊天**:注册 OpenAI 兼容 provider(`https://maas.cau.edu.cn/v1`),自动注入 `Authorization: Bearer <api-key>`
- **用量显示**:查询 `/api/budget/me`,展示本月已用金额(或预算剩余)与账期结束时间
- **凭证验证**:登录时即调用 `/api/auth/login` 验证 Key 有效性,避免存下无效凭证

## 实现要点(续)

平台对 Magpie 暴露的接口全部可用 Bearer API Key 访问:

| 接口 | 用途 |
|---|---|
| `GET /v1/models` | 模型列表 |
| `POST /v1/chat/completions` | 聊天(OpenAI 兼容) |
| `GET /api/budget/me` | 本月用量 / 预算 |
| `GET /api/user/current` | 用户信息 |
| `POST /api/auth/login` `{api_key}` | Key 换会话 cookie(验证 Key) |

## 开发

```bash
bun install
bunx tsc --noEmit   # 类型检查
```

入口为 `src/index.ts`(Magpie 插件由 Bun 直接加载 TS,无需构建)。修改后重新 `magpie plugin add .` 生效。
