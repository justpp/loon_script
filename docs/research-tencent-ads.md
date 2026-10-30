# 调研：Loon 插件规范与 QQ音乐 / 腾讯动漫 广告接口

> 可靠度标注：【已验证】至少两套独立社区规则一致或有官方文档支持；【推断】接口名/域名强烈暗示用途但无该 App 实证；【未验证】单一来源或仅过滤器归类。
> 所有接口均无官方文档，App 升级后可能失效。

## 一、Loon 插件规范

### 1.1 头部字段

官方字段：`#!name`、`#!desc`、`#!author`、`#!icon`、`#!system`、`#!system_version`、`#!loon_version`、`#!homepage`、`#!date`、`#!tag`、`#!type`。

不存在：`#!version`、`#!category`、`#!arguments`。插件参数用正文的 `[Argument]` 段。不要使用 Surge 的 `%APPEND%`。

### 1.2 插件可包含的段

支持：`[Argument]`、`[General]`（仅白名单键）、`[Rule]`、`[Rewrite]`（旧名 `[URL Rewrite]`）、`[Script]`、`[Host]`、`[Mitm]`、`[IP Rewrite]`。

**不支持**：`[Proxy]`、`[Proxy Group]`、`[Header Rewrite]`、`[Map Local]`、`[Remote Script]`、`[Remote Rule]`。

### 1.3 插件内规则的策略

只能是 `DIRECT`、`REJECT` / `REJECT-IMG` / `REJECT-DICT` / `REJECT-ARRAY` / `REJECT-DROP`、`PROXY`。

`PROXY` 是占位符，由主配置 `[Plugin]` 行映射：

```ini
[Plugin]
https://example.com/a.plugin, policy=QQMusic, enabled=true
https://example.com/b.plugin, policy=TencentComic, enabled=true
```

`policy=` 是行级参数，所以不同插件可以映射到不同策略组。若插件写了 `PROXY` 却未配置 `policy=`，回退行为随版本/配置变化（旧文档：使用全局第一个节点），不可依赖。

### 1.4 主配置策略组

```ini
[Proxy Group]
PROXY = select, Auto, 香港节点, DIRECT
QQMusic = select, PROXY, DIRECT
TencentComic = select, PROXY, DIRECT
Auto = url-test, 香港节点, 日本节点, url = http://www.gstatic.com/generate_204, interval = 600, tolerance = 50
```

`select, PROXY, DIRECT` 合法，但前提是存在名为 `PROXY` 的组；官方示例就是先定义 `PROXY` 组，再 `Final = select,PROXY,DIRECT`。内置 PROXY 在配置含自定义策略组时可能被关闭，不要依赖。

### 1.5 脚本参数（本项目暂不使用）

`[Script]` 的开关参数是 `enable`，不是 `enabled`。Loon 没有 `max-size`，没有 `event` 类型（用 `generic` / `network-changed` 替代）。

### 1.6 规则与 MITM 的关系

- `[Rule] DOMAIN,ads.example.com,REJECT`：按域名拒绝，**不需要 MITM**。
- `[Rewrite]` 对 HTTPS 路径做 `reject` / `reject-200` / `reject-img`：**需要 MITM**，且 host 必须列入 `[Mitm] hostname`。
- 插件 `[Mitm] hostname` 会与主配置合并；插件里写全，不写 `%APPEND%`。

## 二、QQ音乐

### 2.1 广告域（域名级 REJECT，无需 MITM）【已验证】

```text
ad.tencentmusic.com          adcdn.tencentmusic.com       adcdn6.tencentmusic.com
adstats.tencentmusic.com     adexpo.tencentmusic.com      adclick.tencentmusic.com
tmead.y.qq.com               tmeadbak.y.qq.com            tmeadcomm.y.qq.com
tmeadquic.y.qq.com           us.l.qq.com                  p.l.qq.com
sdk.e.qq.com                 tangram.e.qq.com             gdt.qq.com
v.gdt.qq.com                 xs.gdt.qq.com                pgdt.gtimg.cn
adsmind.gdtimg.com           imtmp.net
```

### 2.2 路径级改写（需 MITM）

```text
^https?://us\.l\.qq\.com/exapp                                   reject   # 开屏请求，社区脚本确认是 GET + JSON
^https?://y\.gtimg\.cn/music/common/upload/t_splash_info/         reject   # 开屏素材/配置
^https?://.+?/music/common/upload/t_splash_info/                  reject   # 原始 IP 直连的同一路径
^https?://y\.gtimg\.cn/music/common/upload/targeted_ads           reject-img
^https?://y\.gtimg\.cn/music/common//upload/kg_ad/.*?\d{4}\.jpg   reject-img
^https?://(music|tmead|tmeadcomm)\.y\.qq\.com/maproxy/getPbCompressAd  reject  # 压缩广告接口【推断为 protobuf】
```

MITM hostname：`us.l.qq.com, y.gtimg.cn, music.y.qq.com, tmead.y.qq.com, tmeadcomm.y.qq.com`。

### 2.3 必须保留

`u.y.qq.com`（业务网关 `musicu.fcg`）、`c.y.qq.com`、`i.y.qq.com`、`y.qq.com`、`isure.stream.qqmusic.qq.com`、`ws.stream.qqmusic.qq.com`、`dl.stream.qqmusic.qq.com`、`aqqmusic.tc.qq.com`、`imgcache.qq.com`。

`y.gtimg.cn` 既放专辑封面也放开屏广告，**只能按路径拦，不能整域封**。

### 2.4 未覆盖范围

首页 banner、信息流、播放页、MV、活动弹窗没有公开可靠接口。社区靠拦截 TME 广告域一并失效，信息流中的原生广告卡可能需要改 `u.y.qq.com` 业务响应，风险高，本项目不做。

## 三、腾讯动漫

### 3.1 开屏接口【已验证，多套规则一致】

```text
^https?://iphone\.ac\.qq\.com/.*/Support/bootScreen     reject
^https?://iphone\.ac\.qq\.com/.*/Support/getSystemConf  reject-200   # 默认注释，先只拦 bootScreen
```

MITM：`iphone.ac.qq.com`。

### 3.2 开屏素材【已验证】

```text
^https://manhua\.acimg\.cn/operation/0/[^/?]+\.jpg/0\?tp=sharp(?:&.*)?$  reject
```

必须用 `reject` 而不是 `reject-img`：返回透明图时 App 仍会创建开屏容器。只匹配 `/operation/0/` + `tp=sharp`，避免误杀漫画图、封面、头像。

MITM：`manhua.acimg.cn`。

### 3.3 广告域候选【部分推断】

```text
gdt.qq.com  v.gdt.qq.com  pgdt.gtimg.cn  adsmind.gdtimg.com
p.l.qq.com  sdkreport.e.qq.com  rdelivery.qq.com
```

`a.ac.qq.com` 仅被 AdGuard 中文过滤器归类，未在 App 内验证，不纳入。

### 3.4 必须保留

`ac.qq.com`、`m.ac.qq.com`、`iphone.ac.qq.com`（除 Support 两个路径）、`comic.qq.com`、`ac.tc.qq.com`、`manhua.acimg.cn`（除 operation 路径）、`static.ac.qq.com`、`manhua.qpic.cn`、`ugc.tc.qq.com`、`tpns.tencent.com`（推送，不能拦）。

## 四、风险

1. **误杀 CDN**：`y.gtimg.cn`、`manhua.acimg.cn`、`ac.tc.qq.com` 承载大量正常图片，整域封会直接破坏使用。
2. **SSL Pinning**：腾讯系 App 有多层证书校验。若 MITM 后报网络错误，缩小 MITM host 或退化为域名 REJECT。
3. **QUIC**：`tmeadquic.y.qq.com` 表明广告已走 HTTP/3。域名 REJECT 依赖 SNI 通常有效，路径 Rewrite 可能看不到 QUIC 请求。
4. **响应体加密**：腾讯动漫 Android 端 `CryptUtils` 使用 3DES/ECB + Base64，改响应体不可行；iOS 未验证，但仍不采用。
5. **规则优先级**：主配置规则通常先于插件规则（官方未明确合并顺序），广告 REJECT 与分流规则应放在主配置 `[Rule]` 最前面。
6. **共用域名**：`gdt.qq.com` 等广点通域名被大量腾讯系 App 使用，拦截会影响其他 App 的广告（不影响核心功能）。

## 五、抓包方法

1. Loon 安装并信任 CA，MITM hostname 先按需添加，不要一上来 `*`。
2. 冷启动 App 复现开屏（开屏只在冷启动出现），再测试插屏场景。
3. 在 Loon 日志中按 host 过滤，记录完整 URL、Method、响应类型。
4. 候选 URL 先加临时 `reject`，冷启动复测；确认广告消失且业务正常后再收窄正则。
5. 若看不到请求，检查是否走 QUIC：阻断 UDP 443 或改用 TCP 后重试。
6. 用 mitmproxy 可看完整 body：`mitmdump -w flow.mitm` 保存后离线分析。

## 六、来源

- [Loon Plugins 文档](https://nsloon.app/docs/Plugin/)
- [Loon Script 文档](https://nsloon.app/docs/Script/)
- [Loon Policy Groups](https://nsloon.app/docs/Policy/policygroup/)
- [Loon Rewrite](https://nsloon.app/docs/Rewrite/)
- [Loon MitM](https://nsloon.app/docs/MitM/)
- [Loon 官方示例配置 example2.lcf](https://github.com/Loon0x00/LoonExampleConfig/blob/master/example2.lcf)
- [Zz1xuan/ONE QQMusic.plugin](https://github.com/Zz1xuan/ONE/blob/main/Loon/plugin/QQMusic.plugin)
- [Zz1xuan/ONE TencentComic.plugin](https://github.com/Zz1xuan/ONE/blob/main/Loon/plugin/TencentComic.plugin)
- [zqzess/rule_for_quantumultX QMusicAd.plugin](https://github.com/zqzess/rule_for_quantumultX/blob/master/Loon/Plugin/QMusicAd.plugin)
- [fmz200/wool_scripts QQMusic.lpx](https://github.com/fmz200/wool_scripts/blob/main/Loon/plugin/split/partQ/QQMusic.lpx)
- [blackmatrix7/ios_rule_script Advertising.plugin](https://github.com/blackmatrix7/ios_rule_script/blob/master/rewrite/Loon/Advertising/Advertising.plugin)
- [junjiego2/Tool Remove_Tencent_Comic_ads.lpx](https://github.com/junjiego2/Tool/blob/main/Loon/Plugin/Remove_Tencent_Comic_ads.lpx)
- [腾讯动漫返回数据加密分析](https://www.jb51.net/article/236944.htm)
