# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目性质

Loon（iOS 代理客户端）配置与脚本合集，包含两类产物：

- **插件**（`plugins/*.plugin`）：路径级改写、MITM、可选脚本，可被 Loon 远程加载。
- **主配置片段**（`conf/*.conf`）：策略组、规则、插件引用，需要用户手动合并进主配置。

仓库没有构建、打包、lint 或测试流程，文件本身就是发布产物。正确性依赖对 Loon 运行时与规则优先级的理解。

## 目录职责

- `plugins/*.plugin` — 插件。头部为 `#!name=` / `#!desc=` / `#!author=` / `#!date=` 等；正文可用 `[Rule]`、`[Rewrite]`、`[Script]`、`[Mitm]`、`[Argument]`。
- `scripts/*.js` — 插件引用的脚本，运行在 Loon 的 JavaScriptCore 中。
- `conf/*.conf` — 主配置片段，含 `[Proxy Group]`、`[Rule]`、`[Plugin]` 等段。
- `docs/` — 方案与调研记录。

`scripts/foo.js` 与 `plugins/foo.plugin` 同名成对**仅当插件包含脚本时**才要求；纯改写插件没有对应 `.js`。

## Loon 约束（写插件时必须遵守）

### 插件能力边界

- **插件不能声明 `[Proxy]` / `[Proxy Group]`**。插件内 `[Rule]` 的策略只能是 `DIRECT`、`REJECT` 系列、`PROXY`。
- 插件里的 `PROXY` 是占位符，由主配置 `[Plugin]` 行的 `policy=组名` 映射；`policy=` 是行级参数，一个插件只能映射一个策略组。需要多个独立开关就拆成多个插件。
- 插件不支持 `[Header Rewrite]`、`[Map Local]`，也不支持 Surge 的 `%APPEND%`。
- 插件 `[Mitm]` 只写 `hostname`，证书是全局配置。

### 规则优先级

**主配置规则通常先于插件规则匹配**（官方未明确说明合并顺序，社区普遍观察到插件规则排在主配置规则之后）。广告 REJECT 与分流规则放在 `conf/` 片段，并要求贴到主配置 `[Rule]` 最前面；放进插件可能被 `GEOIP`、`DOMAIN-SUFFIX`、`FINAL` 抢先匹配。

同一段内按「精确 `REJECT` → 分流」排序；不要用 `DOMAIN-SUFFIX` 覆盖同时承载广告与业务的大域（如 `y.qq.com`）。

### 改写与 MITM

- `[Rule]` 的域名 `REJECT` 不需要 MITM。
- `[Rewrite]` 对 HTTPS 路径生效需要 MITM，host 必须列入 `[Mitm] hostname`。
- 图片类广告用 `reject-img` 可能返回透明图并继续触发容器（腾讯动漫开屏即如此），这类场景要用 `reject`。

## 脚本运行时约束

- 无 `require` / `import` / `module.exports`，无 Node API，无 `fetch` / `XMLHttpRequest`；网络请求只能用 `$httpClient`。
- 注入对象：`$request`、`$response`、`$argument`、`$script`、`$environment`、`$httpClient`、`$persistentStore`、`$notification`、`$config`、`$done`。
- `$persistentStore.write(value, key)` 是**值在前、key 在后**，与 `read(key)` 不对称。
- 每次执行必须且只能调用一次 `$done(...)`。
- `[Script]` 的开关参数是 `enable`，不是 `enabled`；Loon 没有 `max-size`，也没有 `event` 类型（用 `generic` / `network-changed`）。

## 校验

没有测试框架：

```bash
node --check scripts/<name>.js   # 仅语法检查，$httpClient 等全局在 Node 中不存在
```

`.plugin` / `.conf` 只能导入 Loon 检查是否报错，并用连接日志确认规则、改写是否命中；改动后需在真机验证核心功能未被误杀。
