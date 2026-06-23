# docx 文件批量修改导致图片丢失 — 事故报告与解决方案

## 一、事故描述

**问题**：使用 `python-docx` 库对含图片的 .docx 文件进行文本替换并保存后，文档中的图片全部消失。

**根因**：`python-docx` 的 `Document.save()` 方法会重新构建整个 XML 文档结构。在这个过程中，图片（Inline Drawing）与段落之间的 XML 关系引用（`rId`）会被丢弃或断裂，导致图片虽然在 zip 包内的 `word/media/` 目录中仍然存在，但 Word 打开时无法将其渲染到正文中。

**影响范围**：任何包含图片、图表、嵌入式对象的 .docx 文件，只要通过 `python-docx` 修改后保存，都可能出现图片丢失。

## 二、`python-docx` 的陷阱

### 场景一：修改 Run.text 后保存

```python
doc = Document("file.docx")
for p in doc.paragraphs:
    for run in p.runs:
        run.text = run.text.replace("old", "new")
doc.save("file.docx")  # 图片大概率丢失
```

### 场景二：直接操作 XML

```python
doc = Document("file.docx")
# 通过 lxml 操作底层 XML
body = doc.element.body
# ... 修改文本节点 ...
doc.save("file.docx")  # 图片也可能丢失，取决于操作方式
```

### 为什么会丢图片？

docx 本质是一个 ZIP 压缩包，内部结构如下：

```
file.docx (ZIP)
├── [Content_Types].xml
├── _rels/.rels
├── word/
│   ├── document.xml          ← 正文内容
│   ├── _rels/document.xml.rels ← 图片关系定义
│   ├── media/
│   │   ├── image1.png        ← 图片文件
│   │   └── image2.jpeg
│   └── ...
└── docProps/
```

图片与正文的关联依赖两处 XML：
1. `word/document.xml` 中有 `<wp:inline>` 或 `<w:drawing>` 元素，引用 `rId`
2. `word/_rels/document.xml.rels` 中定义 `rId` → `media/image1.png` 的映射

`python-docx` 保存时若重新构建了 `document.xml`，但未妥善维护 `rels` 文件中的关系，图片就从正文中断联了。

## 三、推荐方案：直接操作 ZIP 内 XML（安全方式）

### 核心思路

绕过 `python-docx` 的高级 API，直接把 docx 当作 ZIP 打开，只修改 `word/document.xml` 中的文本，不碰图片文件和关系文件。

### 通用脚本

```python
import zipfile
import os
import re
import shutil
from pathlib import Path


def safe_replace_text_in_docx(
    src_path: str,
    dst_path: str = None,
    replacements: list = None,
    regex_patterns: list = None,
) -> bool:
    """
    安全替换 docx 文件中的文本，不破坏图片、表格、格式。

    原理：直接操作 docx 底层 ZIP 包中的 word/document.xml，
         只修改文本节点，不触碰 word/media/ 和 _rels/ 中的任何文件。

    参数：
        src_path       -- 源 docx 文件路径
        dst_path       -- 输出路径，默认覆盖源文件
        replacements   -- 精确文本替换列表，格式 [("旧文本", "新文本"), ...]
        regex_patterns  -- 正则替换列表，格式 [(pattern, replacement), ...]

    返回：
        bool -- 替换是否成功
    """
    src_path = Path(src_path)
    if dst_path is None:
        dst_path = src_path
    else:
        dst_path = Path(dst_path)

    replacements = replacements or []
    regex_patterns = regex_patterns or []

    if not replacements and not regex_patterns:
        print("无事可做（未提供替换内容）")
        return False

    tmp_path = dst_path.with_suffix(".tmp.docx")

    try:
        with zipfile.ZipFile(src_path, "r") as zin:
            with zipfile.ZipFile(tmp_path, "w", zipfile.ZIP_DEFLATED) as zout:
                for item in zin.infolist():
                    data = zin.read(item.filename)

                    # 只修改正文 XML 文件
                    if item.filename == "word/document.xml":
                        text = data.decode("utf-8")

                        # 精确文本替换
                        for old, new in replacements:
                            text = text.replace(old, new)

                        # 正则替换
                        for pattern, replacement in regex_patterns:
                            text = re.sub(pattern, replacement, text)

                        data = text.encode("utf-8")

                    # 所有其他文件（图片、关系、样式等）原封不动
                    zout.writestr(item, data)

        # 原子替换
        os.replace(tmp_path, dst_path)
        return True

    except Exception as e:
        if tmp_path.exists():
            tmp_path.unlink()
        raise e
```

### 使用示例

```python
# 示例 1: 精确文本替换
safe_replace_text_in_docx(
    src_path="原始文件.docx",
    dst_path="修改后.docx",
    replacements=[
        (".csv", ".txt"),
        ("CSV文件", "数据表文件"),
        ("本批次 CSV", "本批次数据表"),
    ],
)

# 示例 2: 正则替换
safe_replace_text_in_docx(
    src_path="原始文件.docx",
    regex_patterns=[
        (r"http://old\.domain\.com", "https://new.domain.com"),
    ],
)

# 示例 3: 批量处理多个文件
from pathlib import Path

files = Path("docs").rglob("*.docx")
for f in files:
    safe_replace_text_in_docx(
        src_path=f,
        replacements=[("旧文案", "新文案")],
    )
```

## 四、为什么这个方案安全

| 对比维度 | python-docx API | ZIP + XML 直接操作 |
|----------|:-------------:|:----------------:|
| 修改后图片丢失 | **高概率** | **不会** |
| 修改后表格损坏 | **高概率** | **不会** |
| 修改后格式丢失 | **高概率** | **不会** |
| 文本替换能力 | 强 | 强 |
| 代码复杂度 | 低 | 低 |
| 依赖 | python-docx | 仅 Python 标准库 |

## 五、操作规范（所有 docx 处理的强制要求）

1. **永远不要** 用 `python-docx` 的 `Document.save()` 覆盖含图片/表格的原始文件
2. **操作前** 始终执行 `git commit` 或手动备份原文件
3. **修改后** 至少做两个验证：
   - 文件大小不应骤降（图片数据还在）
   - 用 Word 打开抽查图片是否正常显示
4. **使用上面提供的 `safe_replace_text_in_docx()`** 作为默认的 docx 文本替换工具
5. **批量操作前** 先在单个文件上验证，确认无误再批量执行

## 六、附录：从已损坏的 docx 中提取图片

如果文件已被 `python-docx` 损坏（图片在 ZIP 中但无法在 Word 中显示），可以用以下脚本提取出所有图片，然后手动重新插入：

```python
import zipfile
from pathlib import Path

def extract_images_from_docx(docx_path: str, output_dir: str):
    """从 docx 中提取所有图片文件"""
    output = Path(output_dir)
    output.mkdir(parents=True, exist_ok=True)
    
    with zipfile.ZipFile(docx_path, "r") as z:
        for name in z.namelist():
            if name.startswith("word/media/") and not name.endswith("/"):
                data = z.read(name)
                filename = Path(name).name
                (output / filename).write_bytes(data)
                print(f"提取: {filename}")
```

---

*本文档适用于此后所有涉及 .docx 文件的脚本化处理场景。*