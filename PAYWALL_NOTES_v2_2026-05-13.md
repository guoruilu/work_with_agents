# 付费墙 venue 抓取笔记

最后更新：2026-05-16

记录用 Python 脚本下载 IEEE Xplore / ACM / Elsevier / Springer 论文的实测经验。所有结论来自 2026-05-04 在 WSL Linux + 校园网 IP 下的实际测试。

## TL;DR

| 出版商 | 状态 | 关键技巧 |
|---|---|---|
| **IEEE Xplore** | ✅ 通 | 必须先暖 cookie 走 doc 页，再请 iframe `getPDF.jsp` |
| **Springer** | ✅ 通 | `/content/pdf/<doi>.pdf` 直链，带浏览器 UA 即可 |
| **ACM DL** | ✅ 通（2026-05 复测推翻原结论）| 浏览器 UA + 暖首页 session cookie（`JSESSIONID`/`MAID`）+ 带 Referer 即可，**已无 Cloudflare 挑战** |
| **Elsevier (ScienceDirect)** | ❌ 挡 | 反爬 403，会返回真实 HTML 但 PDF 接口拒绝 |
| **DBLP API**（元数据） | ✅ 通 | 浏览器 UA 即可，单次最多 1000 hits |

ACM / Elsevier 走不通 ≠ 论文拿不到——绝大多数事件相机论文在 arXiv 上有版本，现有 `arxiv_fallback` 兜底覆盖率 >90%。

## IEEE Xplore 的三步走

实测唯一能拿到 PDF 字节流的方法。关键洞察：**IEEE 用 CloudFront 签名 cookie 做授权，cookie 绑定到「当前 IP × 这个 arnumber」**，必须先访问该论文的 doc 页让服务器派发 cookie，才能下到 PDF。

### Step 1: 暖 session（首次调用一次即可）

```python
import http.cookiejar, urllib.request
cj = http.cookiejar.CookieJar()
op = urllib.request.build_opener(urllib.request.HTTPCookieProcessor(cj))
op.addheaders = [
    ("User-Agent", "Mozilla/5.0 (Windows NT 10.0; Win64; x64) "
                   "AppleWebKit/537.36 (KHTML, like Gecko) "
                   "Chrome/124.0.0.0 Safari/537.36"),
    ("Accept", "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8"),
    ("Accept-Language", "en-US,en;q=0.9"),
    ("Upgrade-Insecure-Requests", "1"),
]
op.open("https://ieeexplore.ieee.org/", timeout=30).read(2048)
```

### Step 2: 访问 doc 页（每篇论文都要）

这是最容易被忽略的一步。访问 `/document/<arnumber>` 时，IEEE 检查请求 IP 是否在订阅范围内；若是，**响应里会塞三个 CloudFront 签名 cookie**：

- `CloudFront-Key-Pair-Id`
- `CloudFront-Policy` — base64 编码的 JSON，写明允许的资源路径 + IP + 过期时间
- `CloudFront-Signature`

这些 cookie 是访问 PDF 字节流的钥匙：

```python
op.open(f"https://ieeexplore.ieee.org/document/{arnumber}", timeout=45).read()
```

可以用 `cj` 看实际拿到的 cookie：

```python
for c in cj:
    if "CloudFront" in c.name:
        print(c.name, c.domain)
```

Policy 解码后类似：

```json
{"Statement":[{"Resource":"https://ieeexplore.ieee.org/mediastore/IEEE/content/media/10203037/10203050/10204173/*",
               "Condition":{"DateLessThan":{"AWS:EpochTime":1777893875},
                            "IpAddress":{"AWS:SourceIp":"145.118.99.8"}}}]}
```

注意：**Resource 包含 arnumber**——每篇论文都要单独暖 cookie，不能复用其他论文的 cookie 去拿这篇的 PDF。

### Step 3: 请 iframe 端点拿 PDF 字节

doc 页 HTML 里有个 iframe：

```html
<iframe src="https://ieeexplore.ieee.org/stampPDF/getPDF.jsp?tp=&arnumber=<id>&ref=<base64-of-doc-url>">
```

`ref` 是 doc URL 的 base64（不带 padding）。直接 GET 这个 iframe URL（带 Referer），返回 `application/pdf`：

```python
import base64
doc_url = f"https://ieeexplore.ieee.org/document/{arnumber}"
ref = base64.b64encode(doc_url.encode()).decode().rstrip("=")
pdf_url = f"https://ieeexplore.ieee.org/stampPDF/getPDF.jsp?tp=&arnumber={arnumber}&ref={ref}"
req = urllib.request.Request(pdf_url, headers={
    "User-Agent": "...Chrome/124...",
    "Accept": "application/pdf,*/*",
    "Referer": doc_url,
    "Sec-Fetch-Dest": "iframe",
    "Sec-Fetch-Mode": "navigate",
    "Sec-Fetch-Site": "same-origin",
})
data = op.open(req, timeout=120).read()
assert data[:4] == b"%PDF"
```

### DOI → arnumber

DBLP 给的 IEEE DOI（`10.1109/<JOURNAL>.<YEAR>.<XXXX>`）后缀 ≠ arnumber。老论文尤其如此：

| DOI | DOI 后缀 | 真实 arnumber |
|---|---|---|
| `10.1109/LRA.2018.2800793` | 2800793（404）| 8260838 |
| `10.1109/TVCG.2023.3320271` | 3320271（404）| 10269643 |

必须跟一次 doi.org 重定向才能拿到 arnumber：

```python
r = op.open(f"https://doi.org/{doi}", timeout=30)
m = re.search(r"/document/(\d+)", r.url)
arnumber = m.group(1) if m else ""
```

新论文（2020+）的 DOI 后缀通常就是 arnumber，但稳妥做法是统一走重定向。

## 踩过的坑（节省下次时间）

| 试过的失败做法 | 为何不行 |
|---|---|
| 默认 `Python-urllib/3.x` UA | IEEE WAF 直接 418 |
| 仅换 UA、不暖 cookie | 502 (CloudFront 拒签) |
| 直接 GET `/stamp/stamp.jsp?arnumber=X` | 这个 URL 返回的是 HTML 外壳（含 iframe），**不是 PDF**——容易被 `Content-Type` 骗以为是 |
| 直接 GET `/iel7/<volId>/<volId>/<arnumber>.pdf` | 是真实 PDF 路径，但没 cookie 拿不到 |
| 用 DOI 后缀当 arnumber | 老论文 DOI 后缀不是 arnumber，文档页 404 |
| 暖 doc 页 A，去下 paper B 的 PDF | CloudFront Policy 绑定 arnumber，串不了 |

## ACM / Elsevier 为什么挡

> **2026-05-13 更新**：ACM 这条结论已被实测推翻——见下方"ACM DL 三步走"小节。下面这段保留 2026-05-04 当时观察，仅作历史记录。

测试过的方法都失败（2026-05-04 当时）：

- 浏览器级 UA + 全套 `Sec-Fetch-*` 头
- 先访问首页暖 cookie
- 不同的 Accept / Accept-Encoding 组合
- 跟 doi.org 重定向

**ACM**（已过时）：当时返回 `<title>Just a moment...</title>` + Cloudflare 的 JS 挑战页。**2026-05-13 复测：无挑战，会话 cookie 暖好即 200**。

**Elsevier**：返回 HTTP 403 但 body 是真实的 ScienceDirect 页面（约 300 KB）——这种"软封锁"是反爬常见手段。即使解析出 `pdfft` 链接也会被拒。

**应对策略**：Elsevier 仍走 arXiv 兜底（`arxiv_find_by_title()`）；ACM 现已可直接抓（见下文）。

## ACM DL 三步走（2026-05-13 新增）

实测于 `ACM_Fetch/scripts/acm_fetch.py`，**7 个 DOI 中 6 个 ACM 全过**（第 7 个是 IJCAI 的 `10.24963` 前缀 DOI，本就不归 ACM 托管）。

### Step 1：暖首页 session

```python
op.open("https://dl.acm.org/", timeout=30).read(4096)
# 拿到 JSESSIONID / MAID / MACHINE_LAST_SEEN / I2KBRCK
```

跟 IEEE 的 Step 1 几乎一样，关键是 **`JSESSIONID` + `MAID`**——这俩是 ACM 的会话凭据，没有它们后续 doc 页会返回精简版（无授权访问 PDF）。

### Step 2：暖 doc 页（每篇都要）

```python
url = f"https://dl.acm.org/doi/{doi}"
req = urllib.request.Request(url, headers={"Referer": "https://dl.acm.org/"})
op.open(req, timeout=45).read()
```

ACM 的订阅授权是基于 **IP × 当前 session** 在 doc 页阶段就完成判定；不像 IEEE 还要派发额外的 CloudFront 签名 cookie，**这里 cookie jar 不变也能继续**——但请求 **必须发**，否则 PDF 端点会拒。

### Step 3：拿 PDF 字节

```python
pdf_url = f"https://dl.acm.org/doi/pdf/{doi}"
req = urllib.request.Request(pdf_url, headers={
    "User-Agent": "...Chrome/124...",
    "Accept": "application/pdf,*/*",
    "Referer": f"https://dl.acm.org/doi/{doi}",
    "Sec-Fetch-Dest": "iframe",
    "Sec-Fetch-Mode": "navigate",
    "Sec-Fetch-Site": "same-origin",
})
data = op.open(req, timeout=120).read()
assert data[:4] == b"%PDF"
```

返回头：`Content-Type: application/pdf;charset=UTF-8`，体积 0.6–6 MB 都见过。

### 与 IEEE 的差异

| 维度 | IEEE | ACM |
|---|---|---|
| 暖 doc 页是否换 cookie | 是（CloudFront 签名 3 件套）| 否（仍是 `JSESSIONID`/`MAID`）|
| PDF URL 模板 | `/stampPDF/getPDF.jsp?arnumber=X&ref=<base64>` | `/doi/pdf/<doi>` 简洁直观 |
| DOI 后缀可否当资源标识 | 老论文不行（需重定向取 arnumber）| 可（DOI 本身就是路径）|
| 跨论文 cookie 复用 | 不行（Policy 绑 arnumber）| 行（同一 session 多篇都可下）|

### 别犯的错

- **不要送非 ACM 域名的 DOI 进来**：IJCAI（`10.24963/*`）、Elsevier（`10.1016/*`）等不在 `dl.acm.org` 上，`/doi/pdf/<doi>` 对它们要么 302 到外站要么返回乱码。脚本里加个 `assert doi.startswith("10.1145/")` 比较保险（虽然有些 IJCAI/JCDL 也会 cross-listed，要看具体情况）。
- **不要省略首页 Step 1**：直接进 `/doi/<doi>` 会拿不到 `JSESSIONID`，后续 PDF 端点 403。

## Springer：最简单

不需要 cookie，不需要 Referer：

```python
url = f"https://link.springer.com/content/pdf/{doi}.pdf"
req = urllib.request.Request(url, headers={
    "User-Agent": "...Chrome/124...",
    "Accept": "application/pdf,*/*",
})
data = urllib.request.urlopen(req, timeout=120).read()
```

带浏览器 UA 直接拿。覆盖 IJCV / Machine Learning / MVA / NCA / NPL。

## DBLP：发现层

不要为每个 venue 单独写 fetcher。DBLP 一个 search API 搞定全部 venue：

```
https://dblp.org/search/publ/api?q=event+camera&format=json&h=1000
```

返回的每条 hit 含 `key`（如 `conf/icra/Author23` / `journals/pami/Author24`），通过 key prefix 映射到 venue。多关键词查询合并去重（按 key）即可。

完整流程：

1. 用 6 个关键词查询（`event camera` / `event-based vision` / `neuromorphic` / `spiking neural network` / `dynamic vision sensor` / `spike camera`）每次取 1000 hits
2. 按 `key` 去重
3. `_dblp_venue_for(key)` 查表得到 `(venue_label, publisher)` 二元组
4. 用 `is_event(title)` 做二次过滤（防止 NLP 类 false positive）
5. 按 publisher 派发到对应 resolver

实测 7 个关键词 × 1000 hits 在 30s 内完成，得到 ~500 篇付费墙 venue 元数据。

## 防御性技巧

1. **速率限制**：脚本里全局 `PAGE_SLEEP=3.0`（HTML 请求间隔）+ `PDF_SLEEP=2.0`（PDF 下载间隔）。IEEE 没明示限流但宁可慢。
2. **PDF 完整性校验**：永远检查 `data[:4] == b"%PDF"`，反爬常返回 HTML 错误页但 `Content-Type` 仍标 PDF。
3. **续跑友好**：把发现和下载分两阶段持久化到 `.crawl_index.json`，下载阶段读 `local_path` 字段判定哪些要重试，避免中断后从头来。
4. **域名级开关**：脚本里 `PAYWALL_DOMAINS` 元组用来手动短路下载（无校园网时跳过 IEEE 等域名直接走 arXiv）。

---

## 2026-05-13 补充：SHAP-AAD 相关工作抓取经验

第二次实战（`RelatedWorks_Fetch/`，15 篇成功 0 失败）后追加的几条教训。原方案完全验证通过，无需修改，只补充周边问题。

### 教训 1：永远不要用 Claude 的 `WebFetch` 工具抓付费墙 venue

`WebFetch` 是无状态单次请求、**没有 cookie 容器**，永远跑不完三步走。要在本机执行下载就必须走 `Bash` + Python `http.cookiejar`，**即使两种工具都从同一台机器出口**。

上一次会话失败的根因不是 IP、不是校园网、不是 UA，而是工具选错。再遇到 IEEE/ACM/Springer 这种需要 cookie 链的，**先看 PAYWALL_NOTES，再写 Bash 脚本**，不要图省事一发 WebFetch。

### 教训 2：DOI 后缀 ≠ arnumber 的反例又添一例

PAYWALL_NOTES 表格基础上再加：

| DOI | DOI 后缀 | 真实 arnumber |
|---|---|---|
| `10.1109/TBME.2019.2911728` | 2911728（404）| **8693547** |

规律重申：**TBME / TVCG / LRA 等期刊的 DOI 后缀几乎从来不是 arnumber**，必须跟 doi.org 重定向。`EMBC.2018.8512205` 这种会议则后缀就是 arnumber——但即便如此，**统一走 `doi.org` 重定向**最稳。

### 教训 3：arXiv ID 不要靠 LLM 凭直觉猜

实际踩到的坑：`arxiv.org/pdf/2502.14025` 猜成 Fan 2025 mamba AAD，结果是引力波论文；`2407.07738` 猜成 Lan 2025 spiking transformer，结果是 Minkowski 几何论文。**LLM 的 arXiv ID "记忆" 是不可靠的**。

正确做法二选一：
1. **arXiv API 检索**：`http://export.arxiv.org/api/query?search_query=ti:"<title>"&max_results=5`，再用 `title` 字段精确匹配
2. **下载后强校验**：`pdftotext -layout -l 1 <pdf>` 读首页，正则匹配预期标题/作者，不匹配立即删除

光看 PDF 字节流以 `%PDF` 开头还不够——它确实是合法 PDF，只是是错的 PDF。

### 教训 4：Elsevier 的 arXiv 兜底覆盖率因领域而异

PAYWALL_NOTES 说事件相机领域 arXiv 覆盖 >90%。**AAD（脑机接口/听觉注意）领域明显更低**：
- [5] Fan 2025 (Information Fusion, mamba AAD) — 无 arXiv 版本
- [7] Lan 2024 (Neural Networks, spiking transformer AAD) — 无 arXiv 版本

医学/生理信号领域作者上 arXiv 的习惯弱于 CV/ML 主流。遇到这两类期刊（Information Fusion、Neural Networks、NeuroImage 等）应先用 WebSearch 确认有无 arXiv 版本，没有就直接跳过，不要在 ScienceDirect 上空耗请求。可行的备选：
- 作者 GitHub Pages 个人主页（如 `drliuqi.github.io` 这次就拿到了 [6] Cai 2022）
- ResearchGate（也会挡，但偶尔放行 preprint 版本）
- 让用户在浏览器手动下载（最快）

### 教训 5：脚本结构上的小改进

这次实战中改进了几个细节，建议沿用：

- **DOI 与 arnumber 都接受**：`resolve_to_arnumber(s)` 用 `s.isdigit()` 自动分流，调用方传哪种都行。
- **下载即跳过**：`os.path.exists(out) and os.path.getsize(out) > 50000` 双判断，避免反复抓同一篇；> 50 KB 阈值能挡住"返回了 HTML 错误页但 Content-Type 标 PDF"的 1-2 KB 垃圾。
- **`pdfs/` 与 `scripts/` 分离**：避免脚本和 PDF 混在一起，方便用 `ls pdfs/` 一眼看完成度。
- **每次任务起一个独立子目录**（如 `RelatedWorks_Fetch/`）+ 子目录里维护 `LOG.md`：日志靠近代码而非靠近根目录，多任务并行时不互相干扰。

---

## 2026-05-16 补充：PPT / Poster 交付复审经验

这部分来自 `SHAP_AAD_Intro/` 的 A4 PPT 制作与 reviewer-executor 循环，虽然不是付费墙抓取经验，但能迁移到 poster 与论文展示材料制作。

### 1. 交付物要用截图复审，不要只相信脚本坐标

`python-pptx` 里的 inches 坐标只是几何估计，PowerPoint / WPS 最终渲染会受字体、行距、符号宽度影响。实际出现过 LOG 写“无溢出”，但截图中 `Symmetry` 卡片文字被 take-away 条遮挡。

可迁移规则：

- 每次 rebuild 后必须导出 PNG；
- 复审对象以 PNG 为准，而不是以脚本坐标为准；
- poster 也一样，尤其要检查大字号正文、caption、底栏、参考文献是否越界。

### 2. reviewer-executor 要循环到双方明确收敛

单轮“审查 -> 修改”不够。更稳的闭环是：

1. reviewer 只审查，不改文件；
2. executor 只按 reviewer 清单改，不额外发散；
3. executor rebuild + screenshot；
4. reviewer 再审新截图；
5. 只有 reviewer 给 `CONVERGED` 且 executor 确认 `AGREES_CONVERGED` 时停止。

这套流程适用于 PPT、poster、camera-ready 图表和 rebuttal figure。

### 3. 事实表述必须区分论文原文和讲者推断

展示材料经常为了讲清楚而加入解释，但必须标清来源。例如本次 PPT 中：

- 论文明确支持：TCN 直接处理 raw EEG 是为了保留 temporal dynamics，并避免 reduced-channel topomap interpolation artifacts；
- 讲者推断：直接对 TCN 做 SHAP 可能把 channel attribution 和 temporal filters 纠缠在一起。

poster 上空间更紧，更容易把推断压缩成强断言。写 poster 时应使用 `Paper claim:` / `Interpretation:` 这种显式标签，避免把讲者理解包装成论文结论。

### 4. 百分比和百分点要写死基线

结果图表中最容易犯的错是把 percentage 与 percentage point 混用。本次修正后的写法：

- `64 -> 48: -1.79 pp`
- `48 -> 32: additional -0.06 pp`
- `64 -> 32 total: -1.85 pp`

poster 中任何 “drops only X%” 都要追问两个问题：

1. 这是 relative percent 还是 percentage points？
2. 对照基线是 64、48，还是上一档配置？

### 5. 机制图的箭头必须表达真实计算流

DeepSHAP 图不能画成 `f(x)-E[f(x')]` 直接生成 `SHAP values`，否则观众会误解为标量差值直接变成 attribution。图中必须看得出：

- forward pass 经过模型；
- backward attribution 也经过模型；
- 最终归因落到 input features。

poster 上机制图通常比文字更先被读到，箭头方向和路径必须比 caption 更可靠。

### 6. 图表之间要互相解释

如果一张图展示 60-channel bar，而旁边表格没有 60 行，需要写明 “Fig. 6 includes 60-channel; table shows selected rows”。poster 上图表更密集，这类“不矛盾但会让人疑惑”的细节要提前消掉。

同理，如果标题强调 pixel map，pixel map 就不能被 ranking plot 在视觉上压住；视觉权重应该跟叙事权重一致。

---

## 修订记录

- 2026-05-16：补充 PPT / Poster 制作与复审经验；新增“最后更新”日期；记录 reviewer-executor 循环、截图复审、事实/推断区分、百分比/百分点、机制图箭头和图表一致性等可迁移规则。
