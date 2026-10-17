<div align="right">

[English](README.md) · **[简体中文](README.zh-CN.md)**

</div>

# dsh-multi-router-intl

在 DeepSeek Harness 里，把模型请求分发到多个客户端——WorkBuddy（国内版与国际版）和 Trae——并单独打通国际版的对话链路。

<sub>TypeScript · 运行不需要构建 · 带依赖锁与变更记录</sub>

---

## 为什么做这个

WorkBuddy 国际版走不通。同一个产品、同一个模型，但有三处和国内版不一样：

| | 国内版 | 国际版 |
|---|---|---|
| 登录 | 一次跳转 | 独立的跳转 + 回调 |
| 端点 | 一个域名 | 另一个域名 |
| 会话续期 | 容错 | 过期快，需要提前续 |

第一次的修法很直接，但是错的：两套并行实现并排放着。每改一处，要改两遍。

真正让结构清晰起来的，是意识到**这件事从来不是关于 WorkBuddy 的**。把产品名剥掉，它永远是同一个操作：

```
请求进来 → 判断该由哪个客户端处理 → 转发 → 把响应流接回来
```

于是「客户端」从代码变成了数据。

---

## 怎么组织的

```
src/
├── clients.mjs          每个客户端是一份声明式条目
├── router.mjs           分发——不认识任何具体产品
├── accounts.mjs         账号级状态
├── workbuddy-login.mjs  唯一没能共享的流程
├── stream.mjs           统一的流式处理
├── config.mjs           配置
└── manager.mjs          编排
```

**一份客户端条目声明的是真正在变的东西**：端点、鉴权方式、登录是否需要单独实现，以及能力差异（可用模型、上下文长度）。

其余部分——路由、流式、重试、会话记账——全部共享。

---

## 接入 Trae

Trae 是检验这套抽象是否成立的第一个机会。

如果要加一个客户端就得改 `router.mjs`，说明抽象没做对——应该回头改抽象，而不是在路由里再加一个分支。

接 Trae 实际只做了一件事：在 `clients.mjs` 里加一份条目。`router.mjs`、`stream.mjs`、`config.mjs` 都没有动。

这成了我现在用的判断标准——**新增一个实现的成本，是检验抽象唯一诚实的办法。**

---

## 没有抽出来的部分

登录。

国际版有自己的跳转与回调，Trae 又是完全另一套。如果强行把它们收敛成同一个形状，产出的配置格式会比它替代的代码更复杂。

所以它留在 `workbuddy-login.mjs` 里单独维护。**在业务确实不同的地方做特化，比强行统一更好。**

---

## 目录

```
src/          19 个模块——路由、客户端、账号、流式、鉴权
scripts/      构建、上游拉取、主机探测、在线检查、预览，
              外加两个校验脚本（客户端发现、热重载）
tests/
docs/         参考实现对照、截图、
              evolution-workbuddy-to-trae.md
upstream/     锁定的上游副本
upstream-lock.json   依赖锁
```

### 校验脚本

| 脚本 | 检查什么 |
|---|---|
| `check-client-discovery.mjs` | 每个客户端条目都能被路由识别 |
| `check-live-reload.mjs` | 热重载后状态保持一致 |

---

## 运行

```bash
npm install
node scripts/build.mjs
node scripts/host-probe.mjs     # 确认主机有响应
node scripts/live-check.mjs     # 用真实账号跑端到端
```

`scripts/preview.mjs` 在本地预览路由配置。

---

## 延伸阅读

- [docs/evolution-workbuddy-to-trae.md](docs/evolution-workbuddy-to-trae.md) —— 迁移过程详解
- [docs/reference-comparison.md](docs/reference-comparison.md) —— 与参考实现的行为对照
- [CHANGELOG.md](CHANGELOG.md) —— 变更记录
- [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md) —— 上游署名

---

## 适用范围

用于在 DeepSeek Harness 内连接模型服务的个人工具。与 WorkBuddy、Trae、Qoder 官方无关。账号与凭据保留在本地。

## 许可证

MIT —— 见 [LICENSE](LICENSE)。
