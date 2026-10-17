---
name: camscanner-full-backup
description: 全量备份扫描全能王(CamScanner)云端文档到本地磁盘并去重。含 search_cloud_doc 的 limit=50 分页限制与时间参数陷阱（只传 end_time 会按修改时间过滤，导致漏文档）、按年份分片枚举的正确做法、下载链接过期规避、PDF 三重完整性校验、MD5/像素级/感知哈希三层去重与质量取舍规则、Windows 重命名瞬时占用与两阶段重编号。触发场景：备份扫描全能王、下载 CamScanner 文档、扫描件导出、核对云端文档数量、扫描件去重。
---

# 扫描全能王云端文档全量备份

> **适用范围**：仅用于备份**使用者本人账号**下的云端文档。请遵守服务条款，不要用于他人账号或批量抓取。

## 核心陷阱（务必先读，否则一定会漏文档）

### 陷阱 1：`search_cloud_doc` 单次最多 50 条，且**没有 offset 参数**
`limit` 最大 50。返回 `total=50` 时**不代表只有 50 条**，只是被截断。

### 陷阱 2：只传 `end_time` 时，接口按 **modify_time** 过滤
这是最隐蔽的坑。实测：

- 传 `{"end_time": 1518678600}` → 过滤的是**修改时间**，会漏掉「创建于早年、但最近被改过」的文档
- 传 `{"start_time": X, "end_time": Y}`（**两个都传**）→ 才按 **create_time** 准确分片

实测案例：某账号有文档创建于 2016 年，但在近期被批量改动过。
用 `end_time` 单参数查询时这些文档**完全搜不到**，直到按年份分片才暴露出来。

### 陷阱 3：跨段会重复
同时传 start/end 时，过滤条件是 `create_time` **或** `modify_time` 落在区间内。
所以一份文档可能同时出现在 2018 段和 2019 段（创建于 2018、修改于 2019）。
**必须按 `cs_doc_id` 去重后再统计总数。**

### 陷阱 4：`doc_type` 过滤只在配合双时间参数时可信
`doc_type="word"` 返回 0 不代表没有 Word 文档——该字段可能未被索引。
不要用它来核实数量，只用作辅助。

## 正确流程

### 第 1 步：按年份分片枚举（唯一可靠的计数方式）

逐年调用，**start_time 和 end_time 都要传**，每段结果远小于 50 故不会截断：

```
start/end 用秒级 Unix 时间戳。常用边界：
2016-01-01 = 1451606400   2017-01-01 = 1483228800
2018-01-01 = 1514764800   2019-01-01 = 1546300800
2020-01-01 = 1577836800   2021-01-01 = 1609459200
2022-01-01 = 1640995200   2023-01-01 = 1672531200
2024-01-01 = 1704067200   2025-01-01 = 1735689600
```
- 最早年份段用 `start_time=0` 兜底，确认没有更早的文档
- 最新年份段 `end_time` 给到 1800000000 以上

把所有段的 `cs_doc_id` 汇总去重，得到真实总数。

### 第 2 步：取下载链接（流水式，不要一次攒齐）

`download_cloud_doc(doc_id, format="pdf")` 返回的是 `file_id` 和临时 `download_url`，**不是文件内容**。

- 图片类文档：`format="pdf"` 导出 PDF（默认行为）
- Office 类：`format="original"`；也有 `zip`（JPG 打包）
- **链接会过期**：`file_id` 形如 `file_<unix时间戳>_<hash>.pdf`
  → 采用「取 10~12 个链接 → 立即下载 → 再取下一批」，不要先攒齐全部
- 并行调用时每批控制在 10 个以内，一次发太多容易触发网络错误

### 第 3 步：用脚本落盘（不要用 download_file 取 raw）

`download_file` 会把二进制灌进上下文，几十份必爆。
改用 Python `urllib` 直接请求**第 2 步返回的 `download_url`** 写盘——原样使用，不要自己拼接口地址
（服务端改版就会失效，也没必要）：

```python
import urllib.request, shutil
req = urllib.request.Request(download_url, headers={"User-Agent": "Mozilla/5.0"})
with urllib.request.urlopen(req, timeout=60) as r, open(local_path, "wb") as f:
    shutil.copyfileobj(r, f)
```

- Windows 下**不要用 Git Bash 的 curl 写中文文件名**，容易出编码问题，用 Python 更稳
- 子目录要 `os.makedirs(os.path.dirname(path), exist_ok=True)`
- 已存在且 >1KB 的文件跳过，脚本可反复重跑

### 第 4 步：三重校验

1. **结构**：头 5 字节 `%PDF-` + 尾部含 `%%EOF`
2. **字节数对账**：本地 `os.path.getsize()` 与接口返回的 `file_size` **逐个相等**（判断传输截断最硬的证据）
3. **深度解析**（推荐）：`pypdf` 以 strict 模式打开，**逐页 `page.get_contents().get_data()`**，能发现"头尾齐全但中间损坏"

```python
from pypdf import PdfReader
r = PdfReader(p, strict=True)
for page in r.pages:
    c = page.get_contents()
    if c is not None: c.get_data()   # 真正解压，损坏会抛异常
```

## 与 App 显示数量对不上的排查顺序

接口枚举数 < App 显示数是常见现象（实测某账号接口 60 份 / App 显示 62 份）。按此顺序排查：

1. 先确认没有更早年份的文档（用 `start_time=0` 兜底查）
2. 确认已按年份去重统计（不是被 limit=50 截断）
3. 剩余差异通常是：
   - **回收站/最近删除**的文档（接口不返回）
   - 本地未上传/同步中的草稿
   - App 端统计口径不同（示例文档、合并历史等）
4. 如实告知用户差异，请其在 App 内核对回收站

## 命名建议

- `序号_标题.pdf`，序号按 create_time 倒序，便于与云端顺序对应
- Windows 非法字符 `\ / : * ? " < > |` 需替换；`《》（）` 可用
- 超长标题人工缩短（避免路径过长）
- 云端文件夹（如"我的证件"）建成子目录保留结构

### 第 5 步：去重（比较质量保留一份）

CamScanner 云端常存在重复备份（同一份扫描被重复上传/同步）。

**重要前提**：CamScanner 导出的 PDF **通常没有 OCR 文本层**（`extract_text()` 返回空），
所以不能靠文本比质量，只能用：图像分辨率 / 文件体积 / **像素级比对**。

三层检测，逐层加严：

```python
# 1) MD5 完全相同 -> 字节级重复
# 2) 页数 + 最大图像像素 + 体积比 >0.98 -> 高度疑似
# 3) 页级 aHash(16x16) 感知哈希：逐页汉明距离 <=6、覆盖率 >=80% -> 内容重复
from PIL import Image
def ahash(img, n=16):
    g = img.convert("L").resize((n, n), Image.Resampling.LANCZOS)
    px = list(g.getdata()); avg = sum(px)/len(px)
    b = 0
    for v in px: b = (b << 1) | (1 if v > avg else 0)
    return b
```

用 pypdf 的 `page.images` 取内嵌图做哈希，比渲染 PDF 快得多。

**像素级确认**（判定"是否真的一样"的终局手段）：

```python
da, db = a.convert("RGB").tobytes(), b.convert("RGB").tobytes()
diff = sum(1 for x, y in zip(da[::997], db[::997]) if x != y)   # 抽样步长加速
```

**取舍规则**（当两份质量完全相同时）：
1. 保留文件名**无副本后缀**的那份（`xxx.pdf` > `xxx(1).pdf` / `xxx(2).pdf`）
2. 元数据里 `/Author = Android_CSx.y.z` 版本**较新**的那份
3. 体积略大的通常元数据更全，但图像一致时无实质差别

**不要硬删**：移入 `_重复文件隔离/` 子目录，等用户确认后再清理。

## 已知问题：文件被占用无法改名

重排编号时若 `os.rename` 抛 `PermissionError: [WinError 32]`：

- **不要一次失败就判定被锁**。实测过：用 `open(p,'r+b')` 扫全目录显示"锁定 0 个"，
  但重命名照样失败 —— 是杀软/Windows 索引器在扫新创建的文件，属**瞬时占用**，几秒后自动释放。
- 正确做法：`attempts=12, wait=2.5` 的重试循环，基本都能过。仍失败才跳过并在最终报告里说明。
- 判定"是否真被锁"用 `open(p, 'r+b')` 试探，比反复 rename 更准。

## 已知问题：重编号撞名

按新序号重命名时，`024 -> 022` 这类"新号正被旧文件占着"会直接失败。
**必须两阶段**：

```python
# 阶段A：全部改成临时名，避开所有冲突
os.rename(src, f"__rstmp{i:04d}__.pdf")
# 阶段B：再改成正式名
os.rename(tmp, dst)
```

编号留空档会误导用户数错（曾出现"明明 60 份却数出 59"的误会），
去重后建议**重排为连续编号**并同步重建索引 CSV。
