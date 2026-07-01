# 付费墙 / 正式版本 / PDF 抓取笔记 v4.1

最后更新：2026-07-01

这是一份可复制到其它文献调研项目中的通用笔记，用于记录 Python 脚本下载论文 PDF、处理付费墙 venue、替换 arXiv/preprint 版本、校验 PDF 正确性的实测经验。

适用前提：

- 只下载你有权限访问的内容，或开放获取 / 作者公开版本。
- 付费墙 venue 的自动下载通常依赖机构网络、校园网 IP、浏览器级 User-Agent、cookie session 和合理速率限制。
- 自动下载不能替代人工核验；每个保存的 PDF 都必须检查文件头和首页题名。

## TL;DR

| 出版商 / 来源 | 实测状态 | 关键技巧 |
|---|---|---|
| IEEE Xplore | 通 | 先暖首页 session，再访问每篇 doc 页拿 CloudFront cookie，最后请求 iframe `stampPDF/getPDF.jsp`；不要把 DOI 后缀当 arnumber |
| ACM DL | 通 | 首页 session cookie + 每篇 DOI doc 页 + `/doi/pdf/<doi>`；带 Referer 和 iframe-like headers |
| Springer | 通 | `https://link.springer.com/content/pdf/<doi>.pdf`，带浏览器 UA |
| Wiley | 部分通 | 试 `pdfdirect` / `epdf` / `pdf`，带 UA 和 Referer；失败时保留人工下载 |
| Nature / PLOS / Frontiers / Hindawi / Science | 多数通 | 站点有稳定 PDF 模板或 landing page PDF link；仍需 `%PDF` 和首页题名校验 |
| TechRxiv / Research Square / bioRxiv / medRxiv / Preprints.org | 部分通 | 先用题名查正式 DOI；没有正式版时再下载预印本 PDF |
| MDPI | 自动脚本常 403 | PDF URL 明明存在也可能拒绝脚本；记录 403，优先浏览器/人工下载或 Playwright |
| Elsevier / ScienceDirect | 自动脚本常挡 | DOI 通常只到 HTML landing page，PDF 接口拒绝；优先查 arXiv、作者主页、机构版本 |
| IOP / De Gruyter / Taylor / Springer book chapter | 多数需人工 | 常返回 HTML 或无 PDF candidate；保留人工下载 |
| OpenReview v1 | 多数通 | `api.openreview.net/pdf?id=<id>` 或 `openreview.net/attachment?id=<id>&name=pdf`；注意限速 |
| OpenReview v2 / TMLR | 部分被 challenge | 论坛页可能跳 `/challenge`，API 返回 `ChallengeRequiredError`；用浏览器/Playwright/cookie session 或人工下载 |
| DBLP API | 通 | 元数据发现很好用，浏览器 UA 即可，单次可取大量结果 |

## 通用请求基础

```python
import base64
import http.cookiejar
import re
import urllib.request

UA = (
    "Mozilla/5.0 (Windows NT 10.0; Win64; x64) "
    "AppleWebKit/537.36 (KHTML, like Gecko) "
    "Chrome/124.0.0.0 Safari/537.36"
)

HEADERS_HTML = {
    "User-Agent": UA,
    "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8",
    "Accept-Language": "en-US,en;q=0.9",
    "Upgrade-Insecure-Requests": "1",
}

HEADERS_PDF = {
    "User-Agent": UA,
    "Accept": "application/pdf,*/*",
    "Accept-Language": "en-US,en;q=0.9",
}

cj = http.cookiejar.CookieJar()
op = urllib.request.build_opener(
    urllib.request.HTTPCookieProcessor(cj),
    urllib.request.HTTPRedirectHandler(),
)
op.addheaders = list(HEADERS_HTML.items())
```

## IEEE Xplore 三步走

实测唯一稳定拿到 IEEE PDF 字节流的方法。关键点：IEEE 用 CloudFront 签名 cookie 做授权，cookie 绑定到“当前 IP x 当前论文 arnumber”。必须先访问该论文 doc 页，服务器才会派发可访问该论文 PDF 的 CloudFront cookie。

### Step 1: 暖首页 session

```python
op.open("https://ieeexplore.ieee.org/", timeout=45).read(4096)
```

### Step 2: DOI 解析真实 arnumber

IEEE DOI 后缀不等于 arnumber。老论文尤其如此，期刊论文常见反例包括 TBME / TVCG / LRA 等。

```python
def resolve_ieee_arnumber(op, doi_or_id: str) -> str:
    value = doi_or_id.strip()
    if value.isdigit():
        return value
    doi_url = value if value.startswith("http") else f"https://doi.org/{value}"
    req = urllib.request.Request(doi_url, headers=HEADERS_HTML)
    with op.open(req, timeout=60) as resp:
        final_url = resp.geturl()
        body = resp.read(200_000).decode("utf-8", errors="ignore")
    m = re.search(r"/document/(\d+)", final_url) or re.search(r"/document/(\d+)", body)
    return m.group(1) if m else ""
```

已踩过的 DOI 后缀不等于 arnumber 反例：

| DOI | DOI 后缀 | 真实 arnumber |
|---|---|---|
| `10.1109/LRA.2018.2800793` | 2800793 | 8260838 |
| `10.1109/TVCG.2023.3320271` | 3320271 | 10269643 |
| `10.1109/TBME.2019.2911728` | 2911728 | 8693547 |

会议 DOI 有时后缀就是 arnumber，但统一走 doi.org 重定向最稳。

### Step 3: 暖每篇 doc 页

```python
arnumber = resolve_ieee_arnumber(op, doi)
doc_url = f"https://ieeexplore.ieee.org/document/{arnumber}"
op.open(urllib.request.Request(doc_url, headers=HEADERS_HTML), timeout=60).read()
```

访问 doc 页后，cookie jar 里通常会出现：

- `CloudFront-Key-Pair-Id`
- `CloudFront-Policy`
- `CloudFront-Signature`

这些 cookie 不能跨论文随便复用，因为 policy 绑定具体资源路径。

### Step 4: 请求 iframe PDF endpoint

doc 页 HTML 里实际加载的 PDF iframe 类似：

```html
<iframe src="https://ieeexplore.ieee.org/stampPDF/getPDF.jsp?tp=&arnumber=<id>&ref=<base64-doc-url>">
```

`ref` 是 doc URL 的 base64，不带 padding。

```python
ref = base64.b64encode(doc_url.encode("utf-8")).decode("ascii").rstrip("=")
pdf_url = f"https://ieeexplore.ieee.org/stampPDF/getPDF.jsp?tp=&arnumber={arnumber}&ref={ref}"

headers = dict(HEADERS_PDF)
headers.update({
    "Referer": doc_url,
    "Sec-Fetch-Dest": "iframe",
    "Sec-Fetch-Mode": "navigate",
    "Sec-Fetch-Site": "same-origin",
})

data = op.open(urllib.request.Request(pdf_url, headers=headers), timeout=120).read()
assert data[:4] == b"%PDF"
```

### IEEE 常见失败原因

| 失败做法 | 表现 | 原因 |
|---|---|---|
| 默认 `Python-urllib/3.x` UA | 418 | IEEE WAF 拒绝 |
| 只换 UA，不暖 cookie | 502 / 418 / HTML | 没有 CloudFront 签名 cookie |
| 只暖首页，不访问每篇 doc 页 | 502 / 403 / 418 | 没有该 arnumber 绑定的 policy |
| 直接 GET `/stamp/stamp.jsp?arnumber=X` | HTML | 这是外壳页，不是 PDF |
| 直接 GET `/iel7/.../<arnumber>.pdf` | 403 / 502 | 真 PDF 路径仍需要签名 cookie |
| 用 DOI 后缀当 arnumber | 404 / 错页 | DOI 后缀经常不是 arnumber |
| 暖 paper A 的 doc 页，下 paper B | 403 / 502 | CloudFront policy 绑定具体论文资源 |

`HTTP 418` 多数是下载流程不完整，不应直接判定为无权限。

## ACM DL 三步走

ACM 也需要 session，但比 IEEE 简单。ACM 的 DOI 本身就是路径标识，通常不需要 arnumber 解析。

### Step 1: 暖首页 session

```python
op.open("https://dl.acm.org/", timeout=45).read(4096)
```

常见 session cookie 包括 `JSESSIONID`、`MAID`、`MACHINE_LAST_SEEN`、`I2KBRCK`。

### Step 2: 暖 DOI doc 页

```python
doc_url = f"https://dl.acm.org/doi/{doi}"
headers = dict(HEADERS_HTML)
headers["Referer"] = "https://dl.acm.org/"
op.open(urllib.request.Request(doc_url, headers=headers), timeout=60).read()
```

### Step 3: 下载 PDF

```python
pdf_url = f"https://dl.acm.org/doi/pdf/{doi}"
headers = dict(HEADERS_PDF)
headers.update({
    "Referer": doc_url,
    "Sec-Fetch-Dest": "iframe",
    "Sec-Fetch-Mode": "navigate",
    "Sec-Fetch-Site": "same-origin",
})
data = op.open(urllib.request.Request(pdf_url, headers=headers), timeout=120).read()
assert data[:4] == b"%PDF"
```

注意：

- 不要把非 ACM DOI 送进 ACM PDF 模板。`10.1145/*` 最稳，其它 DOI 要先确认是否由 ACM 托管。
- 不要省略首页 session。直接进 `/doi/<doi>` 有时拿不到完整授权状态。

## Springer

Springer 期刊 / 会议论文通常最简单：

```python
url = f"https://link.springer.com/content/pdf/{doi}.pdf"
req = urllib.request.Request(url, headers=HEADERS_PDF)
data = urllib.request.urlopen(req, timeout=120).read()
assert data[:4] == b"%PDF"
```

注意：

- `10.1007/<book>_<chapter>` 这种 book chapter 有时返回 HTML，不是 PDF。
- Springer Nature 不等于所有 `10.1038/*` 或 book chapter 都能用同一模板。

## Wiley

Wiley 可以按顺序尝试：

```python
urls = [
    f"https://onlinelibrary.wiley.com/doi/pdfdirect/{doi}",
    f"https://onlinelibrary.wiley.com/doi/epdf/{doi}",
    f"https://onlinelibrary.wiley.com/doi/pdf/{doi}",
]
```

带浏览器 UA 和从 DOI/doc 页来的 Referer。部分 Wiley 期刊可以直接返回 PDF，部分仍会返回 HTML 或需要人工浏览器下载。

## Nature / PLOS / Frontiers / Hindawi / Science

常见模板：

```python
# Nature / Scientific Reports
url = f"https://www.nature.com/articles/{doi.split('/')[-1]}.pdf"

# PLOS
url = f"https://journals.plos.org/plosone/article/file?id={doi}&type=printable"

# Frontiers
url = f"https://www.frontiersin.org/articles/{doi}/pdf"

# Hindawi, older articles vary by journal/year/article id
url = f"https://downloads.hindawi.com/journals/<journal>/<year>/<article_id>.pdf"

# Science / Science Advances
url = f"https://www.science.org/doi/pdf/{doi}?download=true"
```

这些模板需要配合 landing page 解析，不能完全硬编码。拿到 PDF 后仍然必须首页题名校验。

## MDPI

MDPI 常见 PDF URL 类似：

```text
https://www.mdpi.com/<issn>/<volume>/<issue>/<article>/pdf
https://www.mdpi.com/<issn>/<volume>/<issue>/<article>/pdf?download=1
```

但脚本请求经常返回 403，即使浏览器可以打开。处理建议：

- 记录 DOI、PDF URL、HTTP 403，不要写成“无 PDF”。
- 之后用真实浏览器 cookie、Playwright、或人工下载。
- 如果人工下载后自动归档，必须做首页题名校验，避免通用关键词误匹配。

## Elsevier / ScienceDirect

常见表现：

- DOI 重定向到 `linkinghub.elsevier.com` 或 ScienceDirect landing page；
- 返回 `text/html`，不是 PDF；
- 即使页面里解析到 PDF 链接，PDF endpoint 也可能 403。

处理策略：

1. 不要反复硬打 PDF endpoint，容易浪费请求。
2. 优先查 arXiv、PubMed Central、作者主页、机构 repository、ResearchGate 或预印本。
3. 没有公开版本就保留为人工 / 机构下载。

注意：arXiv 覆盖率强烈依赖领域。CV/ML 相关方向通常较高，医学 / 生理信号 / BCI / 传统工程期刊明显更低。

## IOP / De Gruyter / Taylor / 书章节

这些来源经常返回 HTML，而不是 PDF 字节：

```text
https://iopscience.iop.org/article/<doi>/pdf
https://www.degruyter.com/document/doi/<doi>/pdf
https://link.springer.com/content/pdf/<book-chapter-doi>.pdf
```

处理策略：

- 如果返回 HTML，先查 landing page 是否有 `citation_pdf_url` 或显式 PDF link。
- 如果仍没有 PDF 字节，保留为人工下载，不要伪造下载状态。

## TechRxiv / Research Square / bioRxiv / medRxiv / Preprints.org

预印本 DOI 经常在之后对应正式发表 DOI。下载前先做正式 DOI 反查。

需要反查的 DOI 前缀：

- `10.36227/*` TechRxiv
- `10.21203/*` Research Square
- `10.1101/*` bioRxiv / medRxiv
- `10.20944/*` Preprints.org

Crossref 题名反查示例：

```python
import json
import urllib.parse
import urllib.request

def crossref_by_title(title: str):
    params = {
        "query.title": title,
        "rows": "5",
        "select": "DOI,title,container-title,published-print,published-online,issued,publisher,URL",
    }
    url = "https://api.crossref.org/works?" + urllib.parse.urlencode(params)
    req = urllib.request.Request(url, headers=HEADERS_HTML)
    data = json.loads(urllib.request.urlopen(req, timeout=60).read())
    return data["message"]["items"]
```

只接受题名高度匹配的结果；不要因为 DOI 前缀看起来像 publisher 就直接替换。

## 官方 proceedings 页面

很多正式版不需要付费墙，但需要从 metadata 页面解析真正 PDF URL。DBLP / Crossref 给出的 `ee` 往往是 abstract / landing page，不一定是 PDF。

### NeurIPS proceedings

DBLP 常给出类似：

```text
https://proceedings.neurips.cc/paper_files/paper/<year>/hash/<hash>-Abstract-Conference.html
https://papers.nips.cc/paper_files/paper/<year>/hash/<hash>-Abstract.html
```

不要手写猜 PDF 文件名。先打开 abstract page，解析 `citation_pdf_url`：

```python
import re
import urllib.request

def neurips_pdf_from_abstract(abstract_url: str) -> str:
    req = urllib.request.Request(abstract_url, headers=HEADERS_HTML)
    html = urllib.request.urlopen(req, timeout=60).read().decode("utf-8", errors="ignore")
    m = re.search(r'<meta name="citation_pdf_url" content="([^"]+)"', html)
    return m.group(1) if m else ""
```

常见 PDF 后缀包括：

```text
<hash>-Paper.pdf
<hash>-Paper-Conference.pdf
```

具体以后端页面为准，不要固定模板。

### CVF / CVPR / ICCV / ECCV open access

CVF openaccess 页面通常提供 `citation_pdf_url`，并且可用来区分主会和 workshop。

建议流程：

1. 先用 DBLP 或 CVF 站内搜索确认 title / year / venue；
2. 打开 CVF paper HTML page；
3. 解析 `citation_pdf_url`；
4. 首页题名校验；
5. 若 venue 是 workshop，文件名和 venue abbreviation 要显式写成如 `CVPRW`，不要误标主会。

不要只依赖 arXiv comment 判断 CVPR / CVPRW。arXiv comment 可能缺失、滞后或不写 workshop 信息。

## OpenReview

OpenReview 不能只按一个固定 URL 模板处理。实测差异主要来自 OpenReview v1 / v2、venue 配置、以及 challenge gate。

### 元数据发现

先用题名查 forum / note ID：

```python
import json
import urllib.parse
import urllib.request

def openreview_search(title: str, api_base: str = "https://api2.openreview.net"):
    url = (
        f"{api_base}/notes/search?term="
        + urllib.parse.quote(title)
        + "&source=all&limit=10"
    )
    req = urllib.request.Request(url, headers=HEADERS_HTML)
    return json.loads(urllib.request.urlopen(req, timeout=45).read())
```

注意：

- `api.openreview.net` 和 `api2.openreview.net` 覆盖的记录不完全相同；
- ICLR 老论文常在 v1 API 上更稳定；
- ICLR / TMLR 新记录常在 v2 API 上更完整；
- 搜索结果可能包含 workshop、preprint、CoRR、评论或 withdrawn 记录，必须用 venue / title / authors 核验。

### OpenReview venue/status 判定

OpenReview 上“能搜到 forum”不等于正式发表。必须读取 `venue` / `venueid` / decision / invitation 信息，并保守分类。

常见状态判断：

| OpenReview venue 字样 | 建议分类 |
|---|---|
| `ICLR <year> poster/oral/spotlight/notable` | 正式 ICLR |
| `Published as a conference paper at ICLR <year>` 出现在 PDF 首页 | 正式 ICLR 的强证据 |
| `Transactions on Machine Learning Research` 或 PDF 首页 `Published in TMLR` | TMLR |
| `Submitted to ICLR <year>` | 投稿 / under review，不是正式发表 |
| `ICLR <year> Workshop`、`R2-FM Workshop` 等 | workshop，不按主会分类 |
| `CoRR` | 预印本 |
| `Withdrawn` / `Desk Rejected` / `Rejected` | 不应归为正式发表 |

同一题名可能同时存在 workshop 版、CoRR 版和主会版。优先使用正式主会 / 期刊记录；如果只有 workshop 版，应在 venue abbreviation 中显式标 workshop。

### OpenReview v1 PDF

OpenReview v1 记录常可直接下载：

```python
def download_openreview_v1_pdf(note_id: str) -> bytes:
    candidates = [
        f"https://api.openreview.net/pdf?id={note_id}",
        f"https://openreview.net/attachment?id={note_id}&name=pdf",
        f"https://openreview.net/pdf?id={note_id}",
    ]
    last_error = None
    for url in candidates:
        try:
            headers = dict(HEADERS_PDF)
            headers["Referer"] = f"https://openreview.net/forum?id={note_id}"
            data = urllib.request.urlopen(
                urllib.request.Request(url, headers=headers),
                timeout=90,
            ).read()
            if data[:4] == b"%PDF":
                return data
        except Exception as exc:
            last_error = exc
    raise RuntimeError(f"OpenReview v1 PDF failed: {last_error!r}")
```

保存后必须做首页题名和 venue 标记校验。OpenReview PDF 首页常包含：

```text
Published as a conference paper at ICLR <year>
Published in Transactions on Machine Learning Research
```

这些字符串是很有用的正式版本证据。

### OpenReview v2 / TMLR challenge

OpenReview v2 / TMLR 有时会对脚本触发 challenge。典型表现：

| 请求 | 表现 |
|---|---|
| `https://openreview.net/forum?id=<id>` | HTTP 200，但最终 URL 变成 `/challenge?redirect=...` |
| `https://openreview.net/pdf?id=<id>` | HTTP 403，返回 HTML challenge page |
| `https://api2.openreview.net/pdf?id=<id>` | HTTP 403，JSON: `ChallengeRequiredError` |
| `https://api2.openreview.net/attachment?id=<id>&name=pdf` | HTTP 403，JSON: `ChallengeRequiredError` |
| 反复重试 | 可能转成 HTTP 429 / `RateLimitError` |

诊断代码：

```python
def probe_openreview_pdf(note_id: str):
    urls = [
        f"https://openreview.net/forum?id={note_id}",
        f"https://openreview.net/pdf?id={note_id}",
        f"https://api2.openreview.net/pdf?id={note_id}",
        f"https://openreview.net/attachment?id={note_id}&name=pdf",
        f"https://api2.openreview.net/attachment?id={note_id}&name=pdf",
    ]
    for url in urls:
        try:
            req = urllib.request.Request(
                url,
                headers={**HEADERS_PDF, "Referer": f"https://openreview.net/forum?id={note_id}"},
            )
            with urllib.request.urlopen(req, timeout=30) as resp:
                head = resp.read(256)
                print(resp.status, resp.geturl(), resp.headers.get("content-type"), head[:40])
        except urllib.error.HTTPError as exc:
            sample = exc.read(200).decode("utf-8", errors="ignore")
            print(exc.code, exc.headers.get("content-type"), sample[:160])
```

如果明确出现 `ChallengeRequiredError`，单纯增加 User-Agent、Referer、首页 warmup、cookie jar 通常无效。不要无限重试。

### 可行处理策略

按优先级处理：

1. 先试 v1 API：`api.openreview.net/pdf?id=<id>`。
2. 再试 attachment 端点：`openreview.net/attachment?id=<id>&name=pdf`。
3. 如果 v2 API 返回 `ChallengeRequiredError`，改用真实浏览器、Playwright interactive session、或导出的已验证浏览器 cookie。
4. 如果浏览器能下载，人工下载后用首页题名、authors、venue 标记核验，再归档。
5. 如果仍然无法下载正式 PDF，保留 arXiv / author copy fallback，并记录为 `formal-version-found-download-failed`。

建议限速：

- OpenReview 搜索 / note API：至少 2-3 s 间隔；
- OpenReview PDF / attachment：至少 5-10 s 间隔；
- 出现 403 challenge 后不要立即对同一 host 批量重试；
- 出现 429 后停止该 host 的自动请求，等待冷却或改用浏览器。

### 人工下载文件名

浏览器保存的 OpenReview PDF 文件名可能是 `<submission-number>_<truncated-title>.pdf`，不一定含 forum ID。不要依赖文件名做唯一匹配。正确做法仍然是：

1. `%PDF` 文件头检查；
2. 首页题名、作者、venue 标记检查；
3. 与已确认的 forum ID / DOI / venue 元数据绑定；
4. 通过校验后再重命名为标准文件名。

## arXiv fallback

不要凭记忆猜 arXiv ID。LLM 对 arXiv ID 的“记忆”不可靠，猜错后下载到的仍然是合法 PDF，只是错论文。

正确做法：

```python
import xml.etree.ElementTree as ET

def exact_arxiv_pdf_by_title(title: str) -> str:
    params = {
        "search_query": f'ti:"{title}"',
        "start": "0",
        "max_results": "5",
    }
    url = "https://export.arxiv.org/api/query?" + urllib.parse.urlencode(params)
    req = urllib.request.Request(url, headers=HEADERS_HTML)
    root = ET.fromstring(urllib.request.urlopen(req, timeout=45).read())
    ns = "{http://www.w3.org/2005/Atom}"
    expected = normalize_title(title)
    for entry in root.findall(ns + "entry"):
        found = normalize_title(entry.findtext(ns + "title") or "")
        if found == expected:
            for link in entry.findall(ns + "link"):
                if link.attrib.get("title") == "pdf" or link.attrib.get("type") == "application/pdf":
                    return link.attrib.get("href", "")
    return ""
```

保存 arXiv PDF 后仍需首页题名校验。

## DBLP 元数据发现

DBLP 适合快速发现会议/期刊元数据：

```text
https://dblp.org/search/publ/api?q=<query>&format=json&h=1000
```

建议流程：

1. 多关键词查询，每次取 1000 hits；
2. 按 DBLP `key` 去重；
3. 用 `key` prefix 映射 venue / publisher；
4. 用题名关键词做二次过滤，去掉同名领域 false positive；
5. 按 publisher 派发到 IEEE / ACM / Springer / Elsevier / etc. resolver。

### 定期复查 arXiv fallback

之前没找到正式版，不代表之后仍没有正式版。DBLP 对新会议 proceedings、JACM/ACM 期刊、NeurIPS/CVF 等记录可能晚于预印本出现。建议对本地 arXiv/preprint fallback 做定期复查：

```python
def dblp_search_title(title: str, h: int = 10):
    url = (
        "https://dblp.org/search/publ/api?q="
        + urllib.parse.quote(title)
        + f"&format=json&h={h}"
    )
    req = urllib.request.Request(url, headers=HEADERS_HTML)
    return json.loads(urllib.request.urlopen(req, timeout=45).read())
```

升级判断：

- 题名要高度匹配，不要只按系统名或缩写匹配；
- DBLP 结果里如果同时有 `CoRR` 和正式会议/期刊，优先正式版本；
- 对 `J. ACM` / `PACM` / `IEEE` / `ACM` 记录，用 DOI 或 official `ee` 进入 publisher flow；
- 对 `NeurIPS`，从 abstract page 解析 `citation_pdf_url`；
- 对 `CVPR/ICCV`，从 CVF paper page 解析 `citation_pdf_url`；
- 更新失败清单时保留旧失败原因，并把状态改成 `formal_version_resolved`，不要让历史原因继续误导后续审查。

复查频率可以按项目节奏定：初次大规模下载后、投稿前、camera-ready 前各跑一次最稳。

## PDF 校验规则

最小校验：

```python
assert data[:4] == b"%PDF"
assert len(data) > 40_000
```

不要相信 `Content-Type`。反爬页面、登录页、错误页有时会伪装或误标。

推荐首页题名校验：

```bash
pdftotext -layout -l 1 paper.pdf -
```

自动化校验示例：

```python
import re
import subprocess
from difflib import SequenceMatcher

def normalize_title(text: str) -> str:
    text = (text or "").lower()
    text = text.replace("‐", "-").replace("‑", "-").replace("–", "-")
    text = re.sub(r"[^a-z0-9]+", " ", text)
    return re.sub(r"\s+", " ", text).strip()

def first_page_title_match(pdf_path: str, title: str) -> bool:
    proc = subprocess.run(
        ["pdftotext", "-layout", "-l", "1", pdf_path, "-"],
        check=False,
        stdout=subprocess.PIPE,
        stderr=subprocess.DEVNULL,
        text=True,
        timeout=25,
    )
    page = normalize_title(proc.stdout)
    expected = normalize_title(title)
    words = [w for w in expected.split() if len(w) > 3]
    if not page or not words:
        return False
    word_hit_ratio = sum(1 for w in words if w in page) / len(words)
    sequence_similarity = SequenceMatcher(None, expected, page[:max(len(expected) * 3, 240)]).ratio()
    return word_hit_ratio >= 0.72 and sequence_similarity >= 0.30
```

短标题需要人工复核。比如单词式系统名、缩写式标题，关键词覆盖率没有区分度。

## Manual PDF 归档规则

人工下载的 PDF 不能只按文件名或关键词归档。推荐流程：

1. 检查 `%PDF` 文件头；
2. `pdftotext -layout -l 1` 抽首页；
3. 首页必须包含题名、作者、venue 或 DOI 的强证据；
4. 题名匹配失败立即放入人工复核队列；
5. 保留原 DOI、下载 URL、旧文件名、归档文件名和判断备注。

不要只用通用词匹配，例如 `ECG`、`PPG`、`deep learning`、`sensor`、`hardware`。这些词会造成高风险误配。

## 正式版本替换规则

如果本地已有 arXiv / preprint PDF，但元数据有正式 DOI，按以下顺序处理：

1. 用题名查 Crossref / OpenAlex，确认正式 DOI 和 venue；
2. 用 publisher resolver 下载正式 PDF；
3. 首页题名匹配通过后替换本地 PDF；
4. 元数据保留原始 preprint URL 和正式 DOI；
5. 若正式版下载失败，保留 arXiv/preprint 版本，并在文件名或备注里标明 fallback。

建议文件名：

```text
{year}_{venue-abbr}_{short-title-or-system-abbr}.pdf
{year}_{venue-abbr}_{short-title}_via-arXiv.pdf
```

## 速率限制和续跑

建议：

- HTML/page 请求间隔：2-3 s；
- PDF 请求间隔：2 s 以上；
- 失败重试不要无限循环；
- 每次下载后立刻写入状态，保证中断后可续跑；
- 发现阶段和下载阶段分开；
- `pdfs/`、`metadata/`、`scripts/`、`LOG.md` 分开。

续跑判断：

```python
if path.exists() and path.stat().st_size > 50_000 and path.read_bytes()[:4] == b"%PDF":
    skip_download = True
```

## 工具选择

需要 cookie 链的 publisher，必须用本机脚本或真实浏览器 session。无状态网页抓取工具通常跑不完：

- IEEE：需要首页 session + doc 页 CloudFront cookie + iframe PDF；
- ACM：需要首页 session + doc 页；
- MDPI：可能需要真实浏览器 cookie；
- Elsevier：自动脚本常被 PDF endpoint 拒绝。
- OpenReview v2 / TMLR：可能需要完成 challenge 的浏览器 session；脚本看到 `ChallengeRequiredError` 时应切换策略。

如果脚本一直失败，而浏览器可打开 PDF，最省时间的方案通常是人工下载，然后用严格 PDF 校验和归档脚本整理。

## 常见分类

下载失败时不要只写 `failed`。建议至少区分：

- `paywalled-or-manual`: 需要机构、浏览器或人工下载；
- `publisher-blocked`: 站点存在 PDF，但脚本被 403/418/Cloudflare 拒绝；
- `challenge-required`: 站点明确要求浏览器 challenge，例如 OpenReview `ChallengeRequiredError`；
- `no-pdf-candidate`: 没找到 PDF URL；
- `not-pdf`: 返回 HTML / 登录页 / landing page；
- `title-mismatch`: 下载到 PDF，但首页不是目标论文；
- `preprint-only`: 暂无正式版，只能保留 arXiv/bioRxiv/TechRxiv 等版本；
- `formal-version-found-download-failed`: 找到正式 DOI，但正式 PDF 未能自动下载。
- `formal-version-resolved`: 之前失败或 fallback，后来复查找到并归档了正式版本；
- `metadata-found-download-blocked`: 找到元数据，但 PDF 被 challenge / 权限 / 登录阻挡；
- `submitted-not-accepted`: OpenReview 等平台上只有投稿记录，不能按正式发表分类。

失败清单不要只保留 still-failed 条目。已经解决的旧失败可以保留一行 resolved 记录，写清楚：

- 旧失败原因；
- 新发现的正式 DOI / venue / URL；
- 最终归档文件名；
- 校验方式，例如首页题名和 venue marker。

## 防御性清单

- 不用 DOI 后缀猜 IEEE arnumber。
- 不凭记忆猜 arXiv ID。
- 不把 HTML landing page 保存成 PDF。
- 不因为 `Content-Type` 像 PDF 就跳过 `%PDF` 检查。
- 不用只含通用词的匹配规则归档人工 PDF。
- 不把 MDPI 403 写成无 PDF。
- 不对 Elsevier PDF endpoint 反复硬试。
- 不对 OpenReview v2 challenge 反复硬试；确认 `ChallengeRequiredError` 后切换到浏览器/Playwright/cookie 或人工下载。
- 不把 OpenReview 的 `Submitted to ...` 当作正式发表。
- 不因为 DBLP 过去没结果就永久保留 arXiv；投稿前应复查重要 fallback。
- 不直接删除 resolved failure；保留解决路径方便审计。
- 不让失败静默丢失；写入 paywall/failure 清单。
- 不做破坏性清空；默认增量更新。

## 交付物复审经验

这部分不是下载策略，但对文献调研后的报告、PPT、poster、图表制作有通用价值。

### 截图复审优先

脚本坐标或排版参数不等于最终渲染效果。每次生成 PPT / poster / 图表后，导出 PNG 或 PDF 页面截图，以实际渲染为准检查：

- 文字是否溢出；
- caption 是否遮挡；
- 底栏、参考文献、legend 是否越界；
- 大字号标题是否压住图表；
- 图和表是否互相矛盾。

### reviewer-executor 闭环

稳妥流程：

1. reviewer 只审查，不改文件；
2. executor 按 reviewer 清单修改；
3. executor rebuild 并导出截图；
4. reviewer 复审新截图；
5. 双方都确认收敛后停止。

### 区分论文原文和自己的解释

文献总结里要显式区分：

- `Paper claim:` 论文明确说了什么；
- `Interpretation:` 你的理解、迁移、推断；
- `Open question:` 还没确认的点。

不要把自己的推断写成论文结论。

### 百分比和百分点写清楚

结果对比中避免混用 percent 和 percentage point：

- `64 -> 48: -1.79 pp`
- `48 -> 32: additional -0.06 pp`
- `64 -> 32 total: -1.85 pp`

每个“下降 X%”都要说明是相对百分比还是百分点，基线是哪一档。

### 机制图箭头必须表达真实计算流

图比 caption 更先被读到。机制图里箭头方向、输入输出、forward/backward 路径必须真实表达计算过程，不要为了美观简化到误导。

### 图表之间要互相解释

如果一张图包含全量结果，旁边表格只显示 selected rows，要明确说明。视觉权重应与叙事重点一致。

## 修订记录

- 2026-05-13：确认 IEEE / ACM / Springer 通用下载路径；记录 Elsevier 自动脚本受阻、DBLP 元数据发现、PDF 校验和续跑经验。
- 2026-05-16：补充 arXiv ID 不可猜、领域 arXiv 覆盖率差异、脚本结构、PPT / poster 复审等通用经验。
- 2026-07-01 v3：补充 IEEE 418 的 cookie/ref 根因、正式 DOI 反查与 preprint 替换、MDPI 自动 403、人工 PDF 误匹配防护、正式版替换流程。
- 2026-07-01 v4：补充 OpenReview v1/v2/TMLR 的 PDF 下载分流、`ChallengeRequiredError` / `/challenge` / 429 的诊断方式、以及浏览器 session 或人工下载后的严格归档流程。
- 2026-07-01 v4.1：补充 DBLP 定期复查 arXiv fallback、OpenReview submission/workshop/formal venue 判定、NeurIPS/CVF `citation_pdf_url` 解析、以及 resolved failure 的记录规范。
