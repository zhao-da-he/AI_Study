# UnstructuredLoader 完全学习指南（新手小白版）

> 本文档面向 **AI 工程师新手**，从零开始系统讲解 UnstructuredLoader。
> 配套本项目 [test_load/](../test_load/) 和 [documents/](../documents/) 的实战代码。

---

## 📚 目录

- [一、UnstructuredLoader 是什么？](#一unstructuredloader-是什么)
- [二、为什么要用 UnstructuredLoader？](#二为什么要用-unstructuredloader)
- [三、UnstructuredLoader 能解析哪些文件？](#三unstructuredloader-能解析哪些文件)
- [四、本项目用的 3 个 Loader 对比](#四本项目用的-3-个-loader-对比)
- [五、[test_load/demo1.py](../test_load/demo1.py) —— PyPDFLoader 详解](#五test_loaddemo1py--pypdfloader-详解)
- [六、[test_load/demo2.py](../test_load/demo2.py) —— UnstructuredLoader PDF 详解](#六test_loaddemo2py--unstructuredloader-pdf-详解)
- [七、[test_load/demo4.py](../test_load/demo4.py) —— UnstructuredMarkdownLoader 详解](#七test_loaddemo4py--unstructuredmarkdownloader-详解)
- [八、[test_load/dome3.py](../test_load/dome3.py) —— JSON 反向读取](#八test_loaddome3py--json-反向读取)
- [九、[documents/markdown_parser.py](../documents/markdown_parser.py) —— Unstructured 在生产中怎么用](#九documentsmarkdown_parserpy--unstructured-在生产中怎么用)
- [十、参数详解（strategy / mode / coordinates / api_key）](#十参数详解strategy--mode--coordinates--api_key)
- [十一、加载后的 Document 对象详解](#十一加载后的-document-对象详解)
- [十二、常见问题 FAQ](#十二常见问题-faq)
- [十三、学习路线图](#十三学习路线图)

---

## 一、UnstructuredLoader 是什么？

### 1.1 一句话定义

> **UnstructuredLoader = 一台"高级扫描仪"**
> 把 PDF / Word / Markdown 等各种文档，**按元素拆开**（标题/段落/列表/表格……），每个元素变成一个 LangChain `Document` 对象

### 1.2 通俗类比

| 类比 | 解释 |
| --- | --- |
| 📑 **高级扫描仪** | 把任何文档"扫描"成结构化数据 |
| 🔪 **拆机师傅** | 把一篇文章拆成"零件" |
| 🏷️ **贴标签的人** | 给每个零件贴上"这是标题/段落/表格"的标签 |
| 🍳 **菜谱分解** | 把"番茄炒蛋"分解成"番茄 100g + 鸡蛋 3 个 + 盐 5g" |

### 1.3 它跟普通"读文件"有什么区别？

#### 普通读 PDF（`open()` 读出来是乱码）

```python
with open('xxx.pdf', 'rb') as f:
    content = f.read()
# → b'%PDF-1.4\n%\xe2\xe3\xcf\xd3...'  # 二进制乱码
```

#### UnstructuredLoader 读 PDF（结构化输出）

```python
from langchain_unstructured import UnstructuredLoader

loader = UnstructuredLoader(file_path='xxx.pdf', strategy='hi_res')
docs = loader.load()

# docs[0] = Document(
#     page_content='EU V光刻机是...',
#     metadata={'page_number': 1, 'category': 'NarrativeText', ...}
# )
```

> 类比：普通 `open()` 是"直接打开快递包裹看到东西"，UnstructuredLoader 是"拆包 + 把每件物品分类 + 贴标签"。

---

## 二、为什么要用 UnstructuredLoader？

### 2.1 不用的话有什么问题？

#### 问题 1：不同格式需要不同代码

```python
# PDF 用 PyPDFLoader
from langchain_community.document_loaders import PyPDFLoader
pdf_loader = PyPDFLoader(file_path="xxx.pdf")

# Word 用 Docx2txtLoader
from langchain_community.document_loaders import Docx2txtLoader
word_loader = Docx2txtLoader(file_path="xxx.docx")

# Excel 用 UnstructuredExcelLoader
from langchain_community.document_loaders import UnstructuredExcelLoader
excel_loader = UnstructuredExcelLoader(file_path="xxx.xlsx")

# 每种格式都要学一个 loader 😩
```

#### 问题 2：只能拿到"原始文本"

```python
# PyPDFLoader 只能拿到"一页一段"
docs = PyPDFLoader(file_path='xxx.pdf').load()
# docs[0] = Document(page_content='第 1 页全文...')
# 不知道哪里是标题，哪里是表格
```

#### 问题 3：扫描件 PDF（图片）读不出来

```python
# 扫描版 PDF 本质是图片
# PyPDFLoader 提取 → 空字符串
# UnstructuredLoader 用 OCR → 能读出文字 ✅
```

### 2.2 用 UnstructuredLoader 的好处

| 优点 | 说明 |
| --- | --- |
| **统一接口** | PDF/Word/MD/Excel/Html……都一个套路 |
| **元素级拆分** | 按"标题/段落/列表/表格"分类 |
| **元数据丰富** | 每页号、坐标、分类都记录 |
| **支持 OCR** | 扫描件 PDF 也能读 |
| **保留版面** | 知道元素在页面上的位置 |
| **开源免费** | 完全免费使用 |

### 2.3 终极类比

> 没有 UnstructuredLoader：你需要给每种文档买不同的"扫描仪"
>
> 有了 UnstructuredLoader：**一台"万能扫描仪"搞定所有文档**

---

## 三、UnstructuredLoader 能解析哪些文件？

### 3.1 支持的文档类型

| 文档类型 | 文件后缀 | 解析难度 |
| --- | --- | --- |
| **PDF** | `.pdf` | ⭐⭐ |
| **Word** | `.docx`, `.doc` | ⭐ |
| **Markdown** | `.md` | ⭐ |
| **HTML** | `.html`, `.htm` | ⭐ |
| **Excel** | `.xlsx`, `.xls` | ⭐⭐ |
| **PowerPoint** | `.pptx`, `.ppt` | ⭐⭐ |
| **纯文本** | `.txt` | ⭐ |
| **EPUB** | `.epub` | ⭐ |
| **图片**（OCR） | `.png`, `.jpg` | ⭐⭐⭐ |

### 3.2 解析后的"元素类型"

Unstructured 把任何文档拆成这些"零件"：

| category | 中文 | 例子 |
| --- | --- | --- |
| `Title` | 标题 | `# 一级`、`## 二级` |
| `NarrativeText` | 正文段落 | 普通的文字段落 |
| `ListItem` | 列表项 | `- 第一条` |
| `Table` | 表格 | Markdown 表格 |
| `CodeSnippet` | 代码块 | ` ```python ` |
| `Image` | 图片 | `![](...)` |
| `Header` / `Footer` | 页眉/页脚 | PDF 页眉页脚 |
| `Formula` | 公式 | 数学公式 |
| `Address` / `Email` | 地址/邮箱 | 联系方式 |

---

## 四、本项目用的 3 个 Loader 对比

[test_load/](../test_load/) 目录演示了 3 种 Loader：

| Loader | 文件 | 输入 | 输出 | 粒度 |
| --- | --- | --- | --- | --- |
| **PyPDFLoader** | [test_load/demo1.py](../test_load/demo1.py) | PDF | 每页 1 Doc | 粗 |
| **UnstructuredLoader** | [test_load/demo2.py](../test_load/demo2.py) | PDF | 每元素 1 Doc | **细** |
| **UnstructuredMarkdownLoader** | [test_load/demo4.py](../test_load/demo4.py) | Markdown | 每元素 1 Doc | **细** |

### 4.1 选哪个？

| 场景 | 推荐 |
| --- | --- |
| 简单 PDF，快速读取 | PyPDFLoader |
| PDF 含表格/图片/复杂版面 | UnstructuredLoader (hi_res) |
| Markdown 文档 | UnstructuredMarkdownLoader |
| Word / Excel | UnstructuredLoader |
| RAG 项目（精细拆分） | UnstructuredLoader / UnstructuredMarkdownLoader |

### 4.2 一个例子对比

输入：一份 PDF 文档，包含标题 + 表格 + 段落

```
PyPDFLoader 输出：
  [Doc: "第 1 页全部内容"]          ← 一页混在一起

UnstructuredLoader 输出：
  [Doc: Title "半导体技术"]           ← 单独一个标题
  [Doc: NarrativeText "本文介绍..."]  ← 单独一段
  [Doc: Table "型号 | 波长"]           ← 单独一张表
  [Doc: NarrativeText "EUV 是..."]    ← 单独一段
```

---

## 五、[test_load/demo1.py](../test_load/demo1.py) —— PyPDFLoader 详解

### 5.1 完整代码

```python
from langchain_community.document_loaders import PyPDFLoader

pdf_file = r'E:\my_project\RAG_PROJECT\datas\layout-parser-paper.pdf'

loader = PyPDFLoader(file_path=pdf_file)
docs = loader.load()

print(f'doc的数量是: {len(docs)}')
print(docs[0].metadata)
print(docs[0].page_content)
```

### 5.2 逐行讲解

```python
from langchain_community.document_loaders import PyPDFLoader
# ↑ 导入 PyPDFLoader（LangChain 内置的 PDF 读取器）

pdf_file = r'E:\...\layout-parser-paper.pdf'
# ↑ PDF 文件路径（前面加 r 表示原始字符串，反斜杠不转义）

loader = PyPDFLoader(file_path=pdf_file)
# ↑ 创建加载器（参数 file_path 是 PDF 路径）

docs = loader.load()
# ↑ 一行加载！把整个 PDF 一次性读到内存
# ↑ 每个 Doc 对应 PDF 的一页
```

### 5.3 输出示例

```
doc的数量是: 16          ← 16 页的 PDF
docs[0].metadata: {'page': 0, 'source': '...layout-parser-paper.pdf'}
docs[0].page_content: 'Layout Parser: A Unified Toolkit...'
```

### 5.4 PyPDFLoader 的局限

| 局限 | 说明 |
| --- | --- |
| 扫描件 PDF | 提取不出文字（要 OCR） |
| 表格 | 不能识别表格结构 |
| 双栏排版 | 内容顺序会乱 |
| 元数据 | 只有 page 和 source |

> 💡 如果 PDF 比较复杂，用 UnstructuredLoader。

---

## 六、[test_load/demo2.py](../test_load/demo2.py) —— UnstructuredLoader PDF 详解

### 6.1 完整代码

```python
import json
from IPython.core.display import HTML
from IPython.core.display_functions import display
from langchain_unstructured import UnstructuredLoader

pdf_file = r'E:\...\layout-parser-paper.pdf'

def write_json(data, file_name):
    with open('E:\\...\\output\\' + file_name, 'w', encoding='utf-8') as f:
        json.dump(data, f, ensure_ascii=False, indent=4)

loader = UnstructuredLoader(
    file_path=pdf_file,
    strategy='hi_res',
    partition_via_api=False,
    coordinates=True,
    api_key='IhWKAZRBmZ14c8tmCsOLabqwIKLJ2e'
)

docs = []
counter = 0
for doc in loader.lazy_load():
    docs.append(doc)
    json_file_name = str(doc.metadata.get('page_number')) + '_' + str(counter) + '.json'
    counter += 1
    write_json(doc.model_dump(), json_file_name)

print(f'doc的数量是: {len(docs)}')
print(docs[0].metadata)
print(docs[0].page_content)
```

### 6.2 创建加载器（4 个关键参数）

```python
loader = UnstructuredLoader(
    file_path=pdf_file,                  # 文件路径
    strategy='hi_res',                    # 解析策略
    partition_via_api=False,              # 是否走云端 API
    coordinates=True,                     # 是否保留坐标
    api_key='IhWKAZRBmZ14c8tmCsOLabqwIKLJ2e'   # API Key（即便走本地也要传）
)
```

| 参数 | 含义 | 必填 |
| --- | --- | --- |
| `file_path` | 文档路径 | ✅ |
| `strategy` | `fast` / `hi_res` / `ocr_only` | ❌ |
| `partition_via_api` | `True` = 走云端 API，`False` = 本地 | ❌ |
| `coordinates` | `True` = 保留元素坐标 | ❌ |
| `api_key` | 云端 API Key（即便本地也要占位） | ❌ |

### 6.3 懒加载 + 保存 JSON

```python
for doc in loader.lazy_load():
    docs.append(doc)
    # 把每个元素存成 JSON
    json_file_name = str(doc.metadata.get('page_number')) + '_' + str(counter) + '.json'
    counter += 1
    write_json(doc.model_dump(), json_file_name)
```

| 操作 | 含义 |
| --- | --- |
| `loader.lazy_load()` | 逐个返回元素（**懒加载**，省内存） |
| `doc.model_dump()` | Pydantic 对象 → 字典（才能存 JSON） |
| `json.dump(data, f)` | 字典 → JSON 文件 |
| `loader.load()` | 一次性全部加载（vs `lazy_load`） |

### 6.4 提取表格 HTML

```python
segments = [
    doc.metadata
    for doc in docs
    if doc.metadata.get("page_number") == 5 and doc.metadata.get("category") == "Table"
]
print(f'表格数据为:')
print(segments)

# 在 Jupyter Notebook 里显示 HTML
# display(HTML(segments[0]["text_as_html"]))
```

> 类比：**找到第 5 页的表格**，把它单独挑出来展示。

### 6.5 demo2 生成的产物

```
datas/output/
├── 1_0.json    ← 第 1 页元素 0
├── 1_1.json    ← 第 1 页元素 1
├── 2_5.json
└── ...
```

每个 JSON 文件长这样：

```json
{
  "page_content": "EUV 光刻机是...",
  "metadata": {
    "page_number": 5,
    "category": "Table",
    "coordinates": {"x1": 100, "y1": 200, "x2": 500, "y2": 400},
    "filename": "layout-parser-paper.pdf"
  }
}
```

---

## 七、[test_load/demo4.py](../test_load/demo4.py) —— UnstructuredMarkdownLoader 详解

### 7.1 完整代码

```python
from langchain_community.document_loaders import UnstructuredMarkdownLoader

loader = UnstructuredMarkdownLoader(
    file_path=r'E:\...\operational_faq.md',
    mode='elements',
    strategy='fast'
)

docs = loader.load()
print(f'doc的数量是: {len(docs)}')

for i in range(10):
    print(docs[i].metadata)
    print(docs[i].page_content)
    print('--' * 50)
```

### 7.2 两个关键参数

```python
UnstructuredMarkdownLoader(
    file_path='xxx.md',
    mode='elements',         # ← 按元素拆
    strategy='fast'          # ← 快速模式
)
```

| 参数 | 取值 | 含义 |
| --- | --- | --- |
| `mode` | `'single'` | 整个文件当 1 个 Doc |
| `mode` | `'elements'` | **按元素拆（推荐）** |
| `mode` | `'paged'` | 按页拆 |
| `strategy` | `'fast'` | 快速模式（默认） |
| `strategy` | `'hi_res'` | 高精度（慢） |

### 7.3 输出示例

```
doc的数量是: 30                              ← 30 个元素

docs[0].metadata:
  {'category': 'Title', 'filename': 'operational_faq.md'}
docs[0].page_content:
  '# 半导体常见问题 FAQ'

docs[1].metadata:
  {'category': 'NarrativeText', 'filename': 'operational_faq.md'}
docs[1].page_content:
  '本文档收集了...'
```

### 7.4 PyPDFLoader vs UnstructuredMarkdownLoader

| | PyPDFLoader | UnstructuredMarkdownLoader |
| --- | --- | --- |
| 文件 | PDF | Markdown |
| 粒度 | 每页 1 Doc | **每元素 1 Doc** |
| 元素类型 | ❌ 无 | ✅ 有 |
| 坐标 | ❌ 无 | ❌ 无 |
| 速度 | 快 | 中 |

---

## 八、[test_load/dome3.py](../test_load/dome3.py) —— JSON 反向读取

### 8.1 完整代码

```python
import json
from langchain_core.documents import Document

def load_doc_from_json(json_file):
    with open(json_file, 'r', encoding='utf-8') as f:
        data = json.load(f)
        return Document(page_content=data['page_content'], metadata=data['metadata'])

if __name__ == '__main__':
    doc = load_doc_from_json('E:\\...\\output\\1_3.json')
    print(doc)
```

### 8.2 作用

> **JSON → Document 反序列化**
> 把 [test_load/demo2.py](../test_load/demo2.py) 生成的 JSON 文件，重新读回 Document 对象

### 8.3 为什么要反向？

| 场景 | 作用 |
| --- | --- |
| **保存中间结果** | PDF 解析很慢，存 JSON 后下次直接用 |
| **调试** | 看清 Unstructured 到底识别出哪些元素 |
| **数据交换** | JSON 是通用格式，跟其他系统对接方便 |
| **重新处理** | 改变 LangChain 处理逻辑，不用再解析 PDF |

### 8.4 跟 demo2 的关系

```
demo2:  PDF → 解析 → 存 JSON → Document 列表
                              ↓
                            json 文件
                              ↓
dome3:  JSON → 反序列化 → Document 对象
```

> demo2 和 dome3 是**一对**操作：前者序列化，后者反序列化。

---

## 九、[documents/markdown_parser.py](../documents/markdown_parser.py) —— Unstructured 在生产中怎么用

### 9.1 生产中的完整流水线

```python
parser = MarkdownParser()
docs = parser.parse_markdown_to_documents('xxx.md')
# → 自动完成：解析 → 父子合并 → 太长的切分
```

### 9.2 内部的 4 个方法

[documents/markdown_parser.py](../documents/markdown_parser.py) 用了 Unstructured 的 4 个核心调用：

```python
from langchain_community.document_loaders import UnstructuredMarkdownLoader

# 1. 创建加载器
loader = UnstructuredMarkdownLoader(
    file_path=md_file,
    mode='elements',
    strategy='fast'
)

# 2. 懒加载读取
for doc in loader.lazy_load():
    docs.append(doc)

# 3. 父子合并
merged = self.merge_title_content(docs)

# 4. 太长的再切分
if len(d.page_content) > 5000:
    new_docs.extend(self.text_splitter.split_documents([d]))
```

### 9.3 与 demo 的区别

| | demo | 生产代码 |
| --- | --- | --- |
| 目的 | 演示功能 | 实际入库 |
| 处理 | 单个文件 | 批量入库 |
| 后续 | 打印看看 | 转向量 + 入 Milvus |

> 生产代码不是直接调 demo，而是在 demo 的基础上**加了业务逻辑**。

---

## 十、参数详解（strategy / mode / coordinates / api_key）

### 10.1 strategy 详解

| strategy | 速度 | 精度 | OCR | 适合 |
| --- | --- | --- | --- | --- |
| `fast` | ⚡⚡⚡ 快 | 一般 | ❌ | 普通文档 |
| `hi_res` | 🐢 慢 | 高 | ✅ | 扫描件、复杂版面 |
| `ocr_only` | ⚡ 中 | 中 | ✅ | 纯图片 PDF |
| `auto` | 自动 | 自动 | 自动 | 默认 |

#### 通俗解释

| strategy | 类比 |
| --- | --- |
| `fast` | 🏃 普通扫描仪（快速但粗） |
| `hi_res` | 📷 高清扫描仪（慢但精，能 OCR） |
| `ocr_only` | 👁 只用 OCR 识别图片 |

#### 怎么选？

```python
# 普通 PDF（文字型）
loader = UnstructuredLoader(strategy='fast')

# 扫描件 PDF / 复杂版面 / 有表格图片
loader = UnstructuredLoader(strategy='hi_res')

# 实在不知道
loader = UnstructuredLoader(strategy='auto')
```

### 10.2 mode 详解（MarkdownLoader 专属）

| mode | 输出 | 例子 |
| --- | --- | --- |
| `'single'` | 整个文件 = 1 个 Doc | `[Doc(...整篇...)]` |
| `'elements'` | **按元素拆（推荐）** | `[Title, Paragraph, ListItem, ...]` |
| `'paged'` | 按页拆 | 多页 PDF 适用 |

### 10.3 coordinates 详解

```python
loader = UnstructuredLoader(coordinates=True)
```

| coordinates | 含义 | 用途 |
| --- | --- | --- |
| `True` | 保留元素在页面上的坐标 | 重新画高亮、版面分析 |
| `False` | 不保留（默认） | 普通检索够用 |

#### 坐标长什么样？

```json
{
  "coordinates": {
    "points": [[100, 200], [500, 200], [500, 400], [100, 400]],
    "system": "PixelSpace",
    "width": 612,
    "height": 792
  }
}
```

> 4 个点定义一个矩形（左上→右上→右下→左下）。

### 10.4 api_key 详解

```python
loader = UnstructuredLoader(
    partition_via_api=True,                  # 走云端
    api_key='YOUR_UNSTRUCTURED_API_KEY'      # 需要注册 Unstructured 账号
)
```

| 模式 | 速度 | 费用 | 限制 |
| --- | --- | --- | --- |
| 本地（`partition_via_api=False`） | 慢（自己跑） | 免费 | 受本地硬件限制 |
| 云端（`partition_via_api=True`） | 快 | 收费 | 有免费额度 |

> 本项目用本地（`partition_via_api=False`），所以 api_key 是个**假值占位**。

---

## 十一、加载后的 Document 对象详解

### 11.1 Document 结构

```python
Document(
    page_content="EU V光刻机是...",         # 必填：正文
    metadata={                              # 可选：元数据
        "page_number": 5,
        "category": "NarrativeText",
        "filename": "xxx.md",
        "coordinates": {...}
    }
)
```

### 11.2 metadata 字段详解

不同 Loader 给出的 metadata 字段不同：

#### PyPDFLoader 的 metadata

```python
{
    "source": "xxx.pdf",     # 文件路径
    "page": 0               # 页码（从 0 开始）
}
```

#### UnstructuredLoader 的 metadata

```python
{
    "filename": "xxx.pdf",       # 文件名
    "page_number": 5,            # 页码（从 1 开始）
    "category": "NarrativeText", # 元素类型
    "coordinates": {...},        # 坐标（如果 coordinates=True）
    "languages": ["zh"],         # 检测到的语言
    "filetype": "application/pdf"
}
```

#### UnstructuredMarkdownLoader 的 metadata

```python
{
    "filename": "xxx.md",
    "category": "Title"          # Title / NarrativeText / ListItem 等
}
```

### 11.3 常用元数据字段对照

| 字段 | 含义 | PyPDFLoader | UnstructuredLoader | UnstructuredMarkdownLoader |
| --- | --- | --- | --- | --- |
| `source` | 文件路径 | ✅ | ❌ | ❌ |
| `page` / `page_number` | 页码 | ✅ | ✅ | ❌ |
| `filename` | 文件名 | ❌ | ✅ | ✅ |
| `category` | 元素类型 | ❌ | ✅ | ✅ |
| `coordinates` | 坐标 | ❌ | ✅ | ❌ |
| `languages` | 语言 | ❌ | ✅ | ❌ |

---

## 十二、常见问题 FAQ

### Q1：扫描件 PDF 怎么处理？

```python
loader = UnstructuredLoader(
    file_path="扫描件.pdf",
    strategy="hi_res"       # 必须用 hi_res
)
```

> `hi_res` 会先渲染 PDF 为图片，再 OCR 识别。

### Q2：中文文档解析乱码怎么办？

```python
from langchain_community.document_loaders import UnstructuredMarkdownLoader

loader = UnstructuredMarkdownLoader(
    file_path='xxx.md',
    mode='elements',
    strategy='fast'
)

# 确认源文件是 UTF-8 编码
```

> 如果源文件是 GBK，要先转 UTF-8。

### Q3：解析太慢怎么办？

| 原因 | 解决 |
| --- | --- |
| `strategy='hi_res'` 太慢 | 改 `strategy='fast'` |
| 一次解析太多文件 | 用 `lazy_load()` 懒加载 |
| 文档很大 | 拆成小文档 |

### Q4：怎么跳过某些页？

```python
loader = UnstructuredLoader(
    file_path="xxx.pdf",
    strategy='hi_res'
)

# 自定义过滤
for doc in loader.lazy_load():
    if doc.metadata.get('page_number') in [1, 2]:  # 跳过第 1、2 页
        continue
    docs.append(doc)
```

### Q5：解析后内存爆了？

```python
# 用 lazy_load 替代 load
for doc in loader.lazy_load():    # 逐个返回
    process(doc)                  # 处理完一个丢一个
```

### Q6：可以同时处理多个文件吗？

```python
import os
from langchain_community.document_loaders import UnstructuredFileLoader

docs = []
for file in os.listdir('docs/'):
    loader = UnstructuredFileLoader(file_path=f'docs/{file}')
    docs.extend(loader.load())
```

### Q7：表格提取不出来？

```python
# 用 hi_res 策略
loader = UnstructuredLoader(strategy='hi_res')

# 或 partition_via_api=True（云端效果更好）
loader = UnstructuredLoader(partition_via_api=True, api_key='YOUR_KEY')
```

### Q8：怎么把 markdown 转 word？

**不行**。Unstructured 是**单向**的：
- ✅ Word → 元素
- ❌ 元素 → Word

> 想要双向用 `python-docx`。

### Q9：UnstructuredLoader 跟 PDFPlumber、PyMuPDF 比？

| Loader | 擅长 | 速度 |
| --- | --- | --- |
| **PyPDFLoader** | 快速读取纯文本 PDF | ⚡⚡⚡ |
| **UnstructuredLoader** | 复杂版面、表格、图片 | ⚡（fast）/ 🐢（hi_res） |
| **PDFPlumber** | 精确提取表格 | ⚡⚡ |
| **PyMuPDF** | 高级版面分析 | ⚡⚡ |

> 本项目用 UnstructuredLoader 是因为它跟 LangChain 集成最好。

### Q10：生产环境怎么部署？

```python
# 1. 安装依赖
pip install unstructured langchain-unstructured

# 2. 处理扫描件要装 OCR 引擎
pip install "unstructured[all-docs]"

# 3. 中文 OCR 要装 tesseract
# Windows: 下载 https://github.com/UB-Mannheim/tesseract/wiki
# Linux: sudo apt install tesseract-ocr tesseract-ocr-chi-sim
```

---

## 十三、学习路线图

### 阶段 1：理解 Loader 的作用（半天）

```
□ 读懂本文档第一、二、三章
□ 知道"为什么需要把文档拆成元素"
□ 知道 3 种元素类型（Title/NarrativeText/Table）
```

### 阶段 2：跑通本项目 4 个 demo（1 小时）

```
□ 读 [test_load/demo1.py](../test_load/demo1.py)（PyPDFLoader）
□ 读 [test_load/demo2.py](../test_load/demo2.py)（UnstructuredLoader）
□ 读 [test_load/demo4.py](../test_load/demo4.py)（UnstructuredMarkdownLoader）
□ 读 [test_load/dome3.py](../test_load/dome3.py)（JSON 反向）
□ 自己运行一遍
```

### 阶段 3：理解生产代码（半天）

```
□ 读 [documents/markdown_parser.py](../documents/markdown_parser.py)
□ 理解 4 步流水线（拆→合→切→入库）
□ 看实际项目怎么把 demo 升级成生产代码
```

### 阶段 4：参数调优（1 小时）

```
□ strategy 选择（fast vs hi_res）
□ mode 选择（single vs elements）
□ coordinates 开关
□ 性能 vs 精度的权衡
```

### 阶段 5：进阶（按需）

```
□ 学 OCR 引擎（tesseract / paddleocr）
□ 学版面分析（layout-parser）
□ 学云端 API（Unstructured SaaS）
□ 学其他 Loader（PDFPlumber / PyMuPDF）
```

### 阶段 6：实战（1 周）

```
□ 处理自己公司的文档
□ 处理扫描件 PDF
□ 提取复杂表格
□ 集成到 RAG pipeline
```

---

## 终极类比

把 UnstructuredLoader 比作"**文档界的翻译官**"：

| UnstructuredLoader | 翻译官 |
| --- | --- |
| 接收各种语言（PDF/Word/MD） | 接收各种方言 |
| 输出统一格式（Document） | 输出普通话 |
| 识别元素类型（Title/NarrativeText） | 区分礼貌语/俚语 |
| 保留元数据（页码/坐标） | 保留出处 |

> 🎯 **一句话总结**：
>
> **UnstructuredLoader = 文档解析的"瑞士军刀"**，把任何文档变成 LangChain Document 对象。
> 本项目演示了 PDF/Markdown 两种用法，生产代码在 [documents/markdown_parser.py](../documents/markdown_parser.py) 里集成。
> **新手必会**：PyPDFLoader / UnstructuredLoader / UnstructuredMarkdownLoader 三个 Loader。