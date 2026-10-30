# 使用说明

自用的 [Loon](https://apps.apple.com/app/loon/id1373567447) 配置与脚本合集。仓库内容即发布产物，无构建、打包步骤。

## 目录

| 目录 | 用途 |
| --- | --- |
| `plugins/` | Loon 插件（`.plugin`），承载路径级 `[Rewrite]`、`[Mitm]`，需要时也放 `[Script]` |
| `scripts/` | 插件内引用的 `.js` 脚本，运行在 Loon 的 JavaScriptCore 中 |
| `conf/` | 主配置片段（`[Proxy Group]`、`[Rule]`、`[Plugin]` 等），供手动合并 |
| `docs/` | 方案与调研文档 |

## 现有插件

### QQ音乐 / 腾讯动漫去广告 + 双代理开关

| 文件 | 作用 |
| --- | --- |
| `plugins/qqmusic-ads.plugin` | QQ音乐开屏/插屏路径改写 + MITM |
| `plugins/tencent-comic-ads.plugin` | 腾讯动漫开屏/插屏路径改写 + MITM |
| `conf/loon-routing.conf` | 广告域 REJECT、分流规则、两个 select 策略组、插件引用 |

### Raw 链接

QQ音乐：

```
https://raw.githubusercontent.com/justpp/loon_script/main/plugins/qqmusic-ads.plugin
```

腾讯动漫：

```
https://raw.githubusercontent.com/justpp/loon_script/main/plugins/tencent-comic-ads.plugin
```

功能：

- 去除两个 App 的开屏与插屏广告（保守范围，不动 banner/信息流）
- 「策略」页新增两个独立下拉：`QQMusic`、`TencentComic`，各自在 `PROXY` / `DIRECT` 间切换
- 广告过滤与开关解耦：选 `DIRECT` 时广告同样被过滤

### 安装

1. Loon →「配置」→「插件」，导入两个 `.plugin`（本地文件或上面的 raw 链接）。
2. 打开 `conf/loon-routing.conf`，按文件顶部注释合并三段内容：
   - `[Proxy Group]` 追加到主配置，把 `<自动选择组名或节点名>` 换成你已有的代理组；
   - `[Rule]` **整段粘贴到主配置 `[Rule]` 的最前面**，必须在 `GEOIP`、`DOMAIN-SUFFIX`、`FINAL` 之前；
   - `[Plugin]` 追加两条引用，把 URL 换成插件实际地址。
3. 确认 Loon CA 证书已安装，并在 iOS「设置 → 通用 → 关于本机 → 证书信任设置」中完全信任。
4. 回到「策略」页，即可看到并切换 `QQMusic` 与 `TencentComic`。

> 规则必须放在主配置靠前位置：Loon 规则自上而下匹配，插件规则通常排在主配置规则之后。若主配置已有 `GEOIP,CN`、`DOMAIN-SUFFIX` 或 `FINAL`，插件内的规则可能匹配不到。

### 开关说明

| 选择 | 效果 |
| --- | --- |
| `PROXY` | 该 App 的专属域名走 `PROXY` 组当前选中的节点/自动组 |
| `DIRECT` | 该 App 的专属域名直连 |

两个开关互不影响。切换只改变出口，不改变去广告行为。

### 已知限制

- iOS 无法按 App 进程分流，只能按域名；`y.gtimg.cn`、`imgcache.qq.com` 等与微信/QQ 共用的域名不纳入分流。
- 代理出口在海外时，QQ音乐/腾讯动漫可能有地域或版权限制，`PROXY` 组建议指向国内节点；若播放异常，可取消 `conf/loon-routing.conf` 中音频 CDN 直连的注释。
- 广告已部分走 QUIC（`tmeadquic.y.qq.com`），路径改写依赖 MITM，QUIC 请求可能无法命中。
- 若 MITM 触发 App 证书锁定报错，移除对应 `[Mitm] hostname`，退化为域名级 `REJECT`。
- App 升级可能改动接口。插件头部保留 `#!date`，失效时按 `docs/research-tencent-ads.md` 的抓包步骤重新确认。

更多细节见 `docs/loon-plugin-plan.md` 与 `docs/research-tencent-ads.md`。

## 新增插件 / 脚本

1. 需要改写广告或接口时，在 `plugins/` 新建 `<name>.plugin`，填写 `#!name` / `#!desc` / `#!author` / `#!date` 等字段。
2. 需要脚本时，在 `scripts/` 新建同名 `<name>.js`，并在插件 `[Script]` 段引用。
3. 需要策略组或分流规则时，写到 `conf/` 片段，不要指望插件内 `[Rule]` 一定优先匹配。

## 校验

```bash
node --check scripts/<name>.js   # 仅语法检查，不模拟 Loon 运行时
```

`.plugin` 与 `.conf` 没有本地校验工具，导入 Loon 后看是否报错，并用连接日志确认规则命中。

## 许可

[MIT](../LICENSE)
