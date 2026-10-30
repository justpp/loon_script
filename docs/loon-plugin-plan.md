# QQ音乐 / 腾讯动漫 去广告 Loon 插件方案

## 1. 背景与目标

为两个 iOS App 去除广告，并在 Loon 内提供**两个互相独立**的代理开关：

| App | 去广告 | 开关 |
| --- | --- | --- |
| QQ音乐 | 开屏、插屏 | `QQMusic` 策略组：`PROXY` / `DIRECT` |
| 腾讯动漫 | 开屏、插屏 | `TencentComic` 策略组：`PROXY` / `DIRECT` |

约束：广告过滤必须与开关解耦——**无论走代理还是直连，广告都照样过滤**。

## 2. 架构结论

### 2.1 为什么拆成两个插件

Loon 的插件（`.plugin`）**不能声明 `[Proxy]` 或 `[Proxy Group]`**，插件内 `[Rule]` 的策略只能是 `DIRECT`、`REJECT` 系列、`PROXY`。其中 `PROXY` 是占位符，由主配置 `[Plugin]` 行的 `policy=组名` 映射，而 `policy=` 是行级参数。

一个插件只能映射一个策略组，所以两个独立开关必须对应两个插件。

### 2.2 为什么规则放在主配置

Loon 的规则匹配是自上而下的，插件规则通常排在主配置规则之后（官方文档未明确说明合并顺序，这是社区一致观察到的行为）。若把 `DOMAIN-SUFFIX,qq.com,DIRECT`、`GEOIP,CN,DIRECT`、`FINAL` 之类规则放在前面，插件里的 `REJECT` / `PROXY` 可能永远匹配不到。

因此本方案把**所有规则放进主配置片段** `conf/loon-routing.conf`（要求贴到 `[Rule]` 最前面），插件只保留**路径级 `[Rewrite]` 与 `[Mitm]`**。这样规则顺序可控、调试简单；插件单独导入也能完成开屏素材拦截。

## 3. 文件清单

| 文件 | 作用 |
| --- | --- |
| `plugins/qqmusic-ads.plugin` | QQ音乐开屏/插屏路径改写 + MITM hostname |
| `plugins/tencent-comic-ads.plugin` | 腾讯动漫开屏/插屏路径改写 + MITM hostname |
| `conf/loon-routing.conf` | 主配置片段：策略组、广告 REJECT、分流规则、`[Plugin]` 引用 |
| `docs/research-tencent-ads.md` | 调研报告：Loon 插件规范、广告接口、风险与抓包方法 |
| `docs/usage.md` | 安装步骤、开关用法、已知限制 |

无 `.js` 脚本：保守范围下 `[Rule]` + `[Rewrite]` 已足够；腾讯动漫响应体有 3DES 加密，改写 body 不可行。

## 4. 主配置片段

见 `conf/loon-routing.conf`，包含三段：

1. `[Proxy Group]`
   ```ini
   PROXY = select, <自动选择组或节点>, DIRECT
   QQMusic = select, PROXY, DIRECT
   TencentComic = select, PROXY, DIRECT
   ```
   `PROXY` 组是「代理侧总选择」，两个业务组只需在 `PROXY` / `DIRECT` 之间切换。若主配置已存在同名 `PROXY` 组，删除这一行即可。

2. `[Rule]`，顺序固定：
   - 广告域 `REJECT`（精确 `DOMAIN`，先于一切分流）
   - QQ音乐专属域 → `QQMusic`
   - 腾讯动漫专属域 → `TencentComic`
   - 注释掉的「音频/图片 CDN 直连」可选规则

3. `[Plugin]`：两条插件引用，`enabled=true`。插件内没有 `PROXY` 规则，因此**不需要 `policy=`**。

> `[Rule]` 段必须粘贴到主配置 `[Rule]` 的最前面，至少在所有 `DOMAIN-SUFFIX`、`GEOIP`、`FINAL` 之前。

## 5. 插件内容

### `plugins/qqmusic-ads.plugin`

```ini
[Rewrite]
^https?://us\.l\.qq\.com/exapp reject
^https?://y\.gtimg\.cn/music/common/upload/t_splash_info/ reject
^https?://.+?/music/common/upload/t_splash_info/ reject
^https?://y\.gtimg\.cn/music/common/upload/targeted_ads reject-img
^https?://y\.gtimg\.cn/music/common//upload/kg_ad/.*?\d{4}\.jpg reject-img
^https?://(music|tmead|tmeadcomm)\.y\.qq\.com/maproxy/getPbCompressAd reject

[Mitm]
hostname = us.l.qq.com, y.gtimg.cn, music.y.qq.com, tmead.y.qq.com, tmeadcomm.y.qq.com
```

### `plugins/tencent-comic-ads.plugin`

```ini
[Rewrite]
^https?://iphone\.ac\.qq\.com/.*/Support/bootScreen reject
^https://manhua\.acimg\.cn/operation/0/[^/?]+\.jpg/0\?tp=sharp(?:&.*)?$ reject
# 若开屏仍在，取消下面一行的注释后复测
# ^https?://iphone\.ac\.qq\.com/.*/Support/getSystemConf reject-200

[Mitm]
hostname = iphone.ac.qq.com, manhua.acimg.cn
```

开屏素材必须用 `reject`，不能用 `reject-img`：返回透明图后 App 仍会创建开屏容器。

## 6. 使用方式

1. Loon →「配置」→「插件」导入两个 `.plugin`（本地文件或 raw URL）。
2. 把 `conf/loon-routing.conf` 的三段内容合并进主配置：策略组、`[Rule]` 置顶、`[Plugin]` 引用。
3. 确认 Loon CA 证书已安装，并在 iOS「设置 → 通用 → 关于本机 → 证书信任设置」中完全信任。
4. 「策略」页出现 `QQMusic` 与 `TencentComic` 两个下拉：
   - 选 `PROXY`：该 App 的专属域名走 `PROXY` 组当前节点；
   - 选 `DIRECT`：直连。
5. 切换开关不影响广告过滤。

## 7. 验证清单

| 检查项 | 通过标准 |
| --- | --- |
| 插件加载 | 无红色报错；Rewrite 与 MITM 生效 |
| 策略组 | 策略页出现两个下拉，可在 `PROXY`/`DIRECT` 间切换 |
| 分流生效 | Loon 连接日志中 `u.y.qq.com`、`iphone.ac.qq.com` 的策略随开关变化 |
| 广告解耦 | `DIRECT` 模式下冷启动 3 次，开屏/插屏均无广告 |
| QQ音乐回归 | 普通/会员歌曲播放、切音质、下载、MV、专辑封面正常 |
| 腾讯动漫回归 | 首页、搜索、阅读 3 本漫画、登录、评论正常 |
| 交叉影响 | `QQMusic` 切到 `PROXY` 时，微信/QQ 图片与消息不受影响 |
| 证书锁定 | 若报网络错误，移除对应 MITM host，退化为域名 REJECT 并记录 |

## 8. 已知限制

- iOS 无法按 App 进程分流，只能按域名；`y.gtimg.cn`、`imgcache.qq.com` 等共用 CDN 不纳入分流。
- 代理出口在海外时，QQ音乐/腾讯动漫可能有地域或版权限制，`PROXY` 组应指向国内节点。
- `tmeadquic.y.qq.com` 说明广告已走 QUIC，路径 Rewrite 依赖 MITM，QUIC 流量可能无法命中。
- 腾讯动漫 `getSystemConf` 默认不拦截，避免影响首页配置。
- App 升级可能改变接口，插件头部保留 `#!date`，改动后需按 `docs/research-tencent-ads.md` 重新抓包。
