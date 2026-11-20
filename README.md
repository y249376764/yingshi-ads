# 海阔视界 · M3U8 去广告规则订阅

从默影视（Silent1566/Mooyingshi）的 HLS 清洗逻辑提取，并结合开源 TVBox 配置（gaotianliuyun/gao、1771245847/TvBox）里的成熟规则改写而成。

## 订阅地址（固定，不要换）

```
https://raw.githubusercontent.com/y249376764/yingshi-ads/main/ads_blank_host.json
```

国内直连不稳时：

```
https://gh-proxy.org/https://raw.githubusercontent.com/y249376764/yingshi-ads/main/ads_blank_host.json
```

填法：海阔视界 → 设置 → 播放器相关设置 → 「M3U8广告…」输入框 → 粘贴地址 → 点【订阅】→ 点【保存】。

> 文件名里的 `blank_host` 是历史原因（第一版用空串 host 才通过校验），内容就是正式规则集，别被名字误导。

---

## 一、格式铁律（踩过的坑，别再犯）

这几条是实测出来的，改文件前先看：

### 1. 顶层必须是纯数组

```json
[ {"name":"…","hosts":["…"],"regex":["…"]} ]
```

不能是 `{"rules":[…]}` 这种对象包装，会直接报「订阅地址有误，返回格式不正确」。

### 2. `hosts` 不能是 `[]`，也不能是 `["*"]`

两种都会报格式错误。**全站生效必须写 `[""]`**——一个空字符串，靠"包含匹配"天然命中所有域名。

### 3. `hosts` 是包含匹配，不是精确匹配

写 `"bfzy"` 就能同时命中 `bfzy.tv`、`s5.bfzycdn.com`、`vip.bfzy.tv`。
所以站点规则只写域名主干，不要写完整域名。

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

### 6. 一条正则只管一个分片块

不要把 `#EXTINF` 和后面的 URL 分两条正则删，也不要写跨行的贪婪匹配，容易删多。

---

## 二、规则清单（11 条）

### 全站生效（`hosts: [""]`）

| 名称 | 删什么 |
|---|---|
| 通用·广告目录 /ad/ /ads/ /adv/ | 路径里带 `/ad/` `/ads/` `/adv/` `/advert/` 的 `.ts` |
| 通用·贴片关键词 preroll/midroll | 文件名含 `preroll` `midroll` `commercial` |
| 通用·混淆名 1o/p1ayer/gg/ad | 混淆命名的广告分片：`1o.ts` `p1ayer.ts` `gg.ts` `guanggao1.ts` `ad5.ts` |
| 通用·adjump 跳转广告 | 文件名含 `adjump`（暴风系常用） |
| 通用·SCTE35 CUE-OUT 广告段 | `#EXT-X-CUE-OUT` 到 `#EXT-X-CUE-IN` 之间的整段（标准广告插播标记） |

### 站点专用（`hosts` 为域名主干）

| 名称 | hosts | 特征 |
|---|---|---|
| 暴风 bfzy·断点3秒 | bfzy / bfbfvip / bfengbf | 两个 `#EXT-X-DISCONTINUITY` 之间夹着 3 秒分片 |
| 量子 lz·断点贴片 | vip.lz / hd.lz / cdnlz | 断点间 6.4x / 7.1x / 4.0x 秒分片 |
| 非凡 ffzy·断点贴片 | vip.ffzy / hd.ffzy / super.ffzy / ffzy | 断点间 6.4x / 6.6x 秒分片 |
| 索尼 suonizy·断点1秒 | suonizy | 断点间 1 秒分片，以及 `/video/original` 路径 |
| 快看 kuaikan·KEY=NONE 段 | kuaikan | `#EXT-X-KEY:METHOD=NONE` 后紧跟的单分片段 |
| 一起看 yqk88·CUE-IN 段 | yqk88 | 断点后到 `#EXT-X-CUE-IN` 之间的段 |

### 为什么站点规则都用「断点包裹」而不是纯时长

纯时长规则（比如"删所有 3 秒分片"）在短剧站会连正片一起删——正片本来就可能是 3 秒一个分片。
`#EXT-X-DISCONTINUITY … #EXT-X-DISCONTINUITY` 这种写法要求广告**被断点单独隔出来**，才动手，误删率低得多。

---

## 三、以后怎么加新规则

### 步骤

1. **拿到该站的真实 m3u8**

   浏览器 F12 → Network → 筛 `m3u8` → 复制媒体清单（不是 master 清单）的 URL，用 curl 拉下来：
   ```bash
   curl -s "https://xxx.com/xxx/index.m3u8" -o /tmp/x.m3u8
   ```

2. **对比广告段和正片段的差异**

   看广告分片有没有这些特征：
   - 路径里的目录名（`/ad/`、`/ads/`、`/gg/`）
   - 文件名（`1o.ts`、`p1ayer.ts`、`adjump.ts`、`ad1.ts`）
   - 前后有没有 `#EXT-X-DISCONTINUITY` 夹着
   - 时长是不是跟正片明显不同（正片 4~10 秒，广告常是 1/3/6.4/6.67/7.17 秒这种）
   - 有没有 `#EXT-X-CUE-OUT` / `#EXT-X-CUE-IN`

3. **写正则并在本地验证**

   装个 node 或直接用 python 模拟。验证两件事：**该删的删了**、**不该删的（正片、下载路径、含 ad 单词的正常文件名）没被删**。

   必须准备的误伤样本：
   ```
   /asset/download/1.ts   /video/adventure/1.ts   /video/ad_2024.ts
   /video/adsense/1.ts    /video/player/1.ts      /video/bad.ts
   /video/glad.ts         /video/1o_backup.ts     /video/s1o.ts
   ```

4. **追加到 `ads_blank_host.json`，推送**

   ```bash
   # 用 GitHub API 更新（需要 token）
   # PUT /repos/y249376764/yingshi-ads/contents/ads_blank_host.json
   ```

   只用追加，不要动已有条目的 `name`（方便定位是哪条出的问题）。

5. **告诉用户重新点【订阅】**

   地址不变，海阔每次订阅都会重新拉取。

---

## 四、已知边界

- **只处理 `.ts`**：走 fMP4 的站（`.m4s` / `.mp4`）删不到，要另写规则。
- **断点规则依赖清单真的带 `#EXT-X-DISCONTINUITY`**：有些站广告段不带断点，只能靠文件名特征。
- **短剧站慎用纯时长规则**：正片可能本身就是 2~3 秒一个分片。
- **CUE-OUT 规则是通用规则**：如果某个站用 CUE-OUT/CUE-IN 标记正片章节，会被误删，真遇到就把这条挪到站点专用。

## 五、来源

- 默影视（Silent1566/Mooyingshi）：`HlsManifestCleaner.java`、`HlsAdRule.java`、`docs/hls-rule-sources.md`
- gaotianliuyun/gao：`0821.json` 的 rules
- 1771245847/TvBox：`MoonYs.json` 的 rules
