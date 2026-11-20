# 海阔视界 · 去广告规则订阅

从默影视（Silent1566/Mooyingshi）仓库提取的 HLS 广告分片规则，改写成海阔 `hosts + regex` 的清单文本匹配格式。

## 订阅地址

```
https://raw.githubusercontent.com/y249376764/yingshi-ads/main/ads.json
```

国内直连不稳时用加速：

```
https://gh-proxy.org/https://raw.githubusercontent.com/y249376764/yingshi-ads/main/ads.json
```

## 规则说明

| 名称 | 作用域 | 说明 |
|---|---|---|
| 通用·广告目录 /ad/ /ads/ /adv/ | 全站 | 广告目录下的 .ts 分片 |
| 通用·贴片关键词 preroll/midroll | 全站 | preroll / midroll / commercial |
| 通用·混淆广告分片名 1o/p1ayer/gg | 全站 | 1o.ts、p1ayer.ts、gg.ts、ad1.ts、guanggao.ts |
| 暴风 bfzy·约3秒贴片 | bfzy.tv / bfzycdn.com | 2.9–3.1 秒短分片 |
| 量子 lz·约6.43秒贴片 | vip.lz / hd.lz / cdnlz | 6.3–6.59 秒分片 |
| 非凡 ffzy·约6.67秒贴片 | vip.ffzy / hd.ffzy / super.ffzy | 6.55–6.79 秒分片 |

## 注意

- 只针对 `.ts` 分片；走 fMP4（`.m4s`）的站删不到。
- 第 4–6 条是默影视 `hls-rule-sources.md` 里的站点特征移植，时长按文档值推算，未对真实站点抓包核验。
- 全站生效的三条若遇到正片命名恰好是 `ad.ts` / `gg.ts`，会误删，把该条 `hosts` 收窄到具体站点即可。
