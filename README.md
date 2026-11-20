# 海阔视界 · M3U8 去广告规则订阅

规则来源：默影视（Silent1566/Mooyingshi）的 HLS 清洗逻辑 + 开源 TVBox 配置（gaotianliuyun/gao、1771245847/TvBox）+ 自建 `yangmi2026/tvbox.json` 里实战用过的 m3u8 规则。

## 订阅地址（固定，不要换）

```
https://raw.githubusercontent.com/y249376764/yingshi-ads/main/ads_blank_host.json
```

国内直连不稳时：

```
https://gh-proxy.org/https://raw.githubusercontent.com/y249376764/yingshi-ads/main/ads_blank_host.json
```

填法：海阔视界 → 设置 → 播放器相关设置 → 「M3U8广告…」输入框 → 粘贴地址 → 点【订阅】→ 点【保存】。

> 文件名里的 `blank_host` 是历史原因（全站规则必须用空串 host 才通过校验），内容就是正式规则集，别被名字误导。

回退用（旧版 11 条，万一新版误删）：

```
https://raw.githubusercontent.com/y249376764/yingshi-ads/main/ads_v3_backup.json
```

---

## 一、格式铁律（实测，违反必失败）

### 1. 顶层必须是纯数组

```json
[ {"name":"…","hosts":["…"],"regex":["…"]} ]
```

不能是 `{"rules":[…]}` 这种对象包装，会直接报「订阅地址有误，返回格式不正确」。

### 2. `hosts` 不能是 `[]`，也不能是 `["*"]`

两种都会报格式错误。**全站生效必须写 `[""]`**——一个空字符串，靠"包含匹配"天然命中所有域名。

### 3. `hosts` 是包含匹配，不是精确匹配

写 `"bfzy"` 就能同时命中 `bfzy.tv`、`s5.bfzycdn.com`、`vip.bfzy.tv`。
所以站点规则只写域名主干，不要写完整域名，更不要写 `*.ffzy*` 这种通配（要清洗成 `ffzy`）。

**host 长度别少于 4**：写 `"c1"`、`"lz"` 会命中大量无关域名。

### 4. 正则严格按 JS 语法，`.ts` 后面必须吃掉换行

海阔是对 m3u8 原文做 `replaceAll(regex, "")`。删分片要连换行一起删，否则留下空行会让清单解析错位。

正确结尾：

```
…\.ts.*?\s+
```

`.ts` 后接 `.*?\s+`：先吃掉可能的查询串（`x.ts?t=9`），再把换行吃掉。

### 5. 保守写法优先

- 用 `(...)` 分组，不用 `(?:...)`
- 跨行匹配用 `[\s\S]*?`，不用 `[^\n]`
- `.` 在 JS 里不匹配换行，所以 `.*?` 天然只在行内匹配，安全

### 6. 绝不能把「纯数字」当正则

TVBox 配置里有大量 `"5.480000"`、`"18.4"` 这种写法，那是 TVBox 内部的按分钟匹配约定。
搬到海阔会直接删掉清单里所有这个数字，把 `#EXTINF:5.480000,` 变成 `#EXTINF:,`，整份清单报废。

---

## 二、规则清单（38 条 / 72 条正则）

### 全站生效（`hosts: [""]`，6 条）

| 名称 | 删什么 |
|---|---|
| 通用·广告目录 /ad/ /ads/ /adv/ | 路径里带 `/ad/` `/ads/` `/adv/` `/advert/` 的 `.ts` |
| 通用·贴片关键词 preroll/midroll | 文件名含 `preroll` `midroll` `commercial` |
| 通用·混淆名 1o/p1ayer/gg/ad | `1o.ts` `p1ayer.ts` `gg.ts` `guanggao1.ts` `ad5.ts` |
| 通用·adjump 跳转广告 | 文件名含 `adjump`（暴风系常用） |
| 通用·SCTE35 CUE-OUT 广告段 | `#EXT-X-CUE-OUT` 到 `#EXT-X-CUE-IN` 之间的整段 |
| 通用-广告片段过滤 | 0~5 秒短分片且文件名含 ad/promo/preroll/midroll/postroll |

### 站点专用（32 条）

覆盖域名：暴风系（bfzy / bfbfvip / bfengbf / s5.bfzycdn）、量子系（vip.lz / hd.lz / cdnlz）、非凡系（vip.ffzy / hd.ffzy / super.ffzy / svipsvip.ffzy）、索尼 suonizy、快看 kuaikan、U酷 ukzy、ikun（bfikuncdn / jkunbf / bfaskcdn）、卧龙 cdn.wl、星星 aws.ulivetv.net、黑木耳 hmrvideo、cdn.ryplay、qihubf、海外看 haiwaikan、360 lyhuicheng。

识别特征分四类：

1. **断点 + 精确时长**：`#EXT-X-DISCONTINUITY\r*\n*#EXTINF:6.400000,[\s\S]*?#EXT-X-DISCONTINUITY`
2. **断点 + 哈希文件名**：`[a-z0-9]{18,}\.ts`（量子系混淆分片名，只在相对路径清单里生效）
3. **`#EXT-X-KEY:METHOD=NONE` 段**：快看、qihubf、U酷、ikun 这类把广告单独换 key
4. **断点 + 指定行数**：非凡系 `#EXT-X-DISCONTINUITY(?:\n.*?){8}\n#EXT-X-DISCONTINUITY`

### 为什么站点规则都用「断点包裹」

纯时长规则（"删所有 3 秒分片"）在短剧站会连正片一起删——正片本来就可能 3 秒一片。
`#EXT-X-DISCONTINUITY … #EXT-X-DISCONTINUITY` 要求广告被断点单独隔出来才动手，误删率低得多。

---

## 三、以后怎么加新规则

### 步骤

1. **拿真实 m3u8**

   F12 → Network → 筛 `m3u8` → 复制**媒体清单**（不是 master）URL：
   ```bash
   curl -s "https://xxx.com/index.m3u8" -o /tmp/x.m3u8
   ```

2. **找广告段特征**

   - 目录名：`/ad/`、`/ads/`、`/gg/`
   - 文件名：`1o.ts`、`p1ayer.ts`、`adjump.ts`、18 位以上哈希名
   - 前后有 `#EXT-X-DISCONTINUITY` 夹着
   - 时长异常：1 / 3 / 6.4 / 6.67 / 7.17 秒
   - `#EXT-X-KEY:METHOD=NONE` 前后切换
   - `#EXT-X-CUE-OUT` / `#EXT-X-CUE-IN`

3. **写正则并本地验证**

   必测 9 个误伤样本：
   ```
   /asset/download/1.ts   /video/adventure/1.ts   /video/ad_2024.ts
   /video/adsense/1.ts    /video/player/1.ts      /video/bad.ts
   /video/glad.ts         /video/1o_backup.ts     /video/s1o.ts
   ```

4. **追加到 `ads_blank_host.json` 并推送**

   只追加，不改已有条目的 `name`（方便定位问题）。

5. **让用户重新点【订阅】**

   地址不变，每次都重新拉。

### 从别人的 TVBox 配置里淘规则

看 `rules` 数组，只留 `regex` 里含 `#EXTINF` 或 `#EXT-X` 的条目，然后：

- `hosts` 里的 `*` 全去掉（`*.ffzy*` → `ffzy`）
- 纯数字正则全部丢弃
- 不以 `#` 开头的正则丢弃（会删掉 URL 却留下 `#EXTINF`，清单报废）
- host 短于 4 个字符的丢弃（`c1`、`lz` 这种会误伤）

> 注意：TVBox 的 `rules` 里还有一类「嗅探过滤」和「自动点击」，比如 `video/tos/cn`、`item_id=`、`document.querySelector(...).click()`，那是给网页播放用的，跟 m3u8 分片广告无关，不要搬。

---

## 四、已知边界

- **只处理 `.ts`**：fMP4 的站（`.m4s` / `.mp4`）删不到。
- **断点规则依赖清单真的带 `#EXT-X-DISCONTINUITY`**：不带的站只能靠文件名特征。
- **短剧站慎用纯时长规则**：正片可能本身就是 2~3 秒一片。
- **CUE-OUT 是通用规则**：若某站用 CUE 标记正片章节会误删，真遇到就挪成站点专用。
- **行数型规则脆弱**：`(?:\n.*?){8}` 依赖分片数量，站方改结构就失效（失效只是不删，不会删错）。

## 五、来源

- 默影视（Silent1566/Mooyingshi）：`HlsManifestCleaner.java`、`HlsAdRule.java`、`docs/hls-rule-sources.md`
- gaotianliuyun/gao：`0821.json`
- 1771245847/TvBox：`MoonYs.json`
- 自建：`y249376764/yangmi2026` 的 `tvbox.json`（114 条 rules 里的 34 条 m3u8 规则）
