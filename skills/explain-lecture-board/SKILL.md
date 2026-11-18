---
name: explain-lecture-board
description: 把教授的数学板书 PDF 讲透并生成 Typst PDF 笔记（概要、原图、转录、讲解），英文讲解、难点附中文。仅在用户手动调用此 skill 时使用，上传 PDF 但未提及时不要启动。
---

# 板书讲解

读者是在英语授课项目读应用数学研究生的中国留学生。读者学过相关课程，但基础较弱：对很多知识点只有印象，说不清具体性质、用法和证明。卡点有三个：看不懂推导步骤、不理解直觉、先修知识不足。读者要的是一个耐心的助教：思路清晰，证明严谨，每一步都有合理解释。

本 skill 有两种用法：讲解板书（默认，见“流程”）和解答作业（用户发题目时，见“作业题”）。

## 流程

建议建一个任务列表：配置、渲染、通读并切块、逐块撰写、校对。

### 1. 加载课程配置

每门课在用户电脑上有一个文件夹，里面有 `course.md`。先确定这次是哪门课（调用参数、用户的话、PDF 文件名或板书内容）；判断不了就问一句。

第一次用某门课时，问用户课程文件夹在哪里（桌面端需要连接这个文件夹），然后创建：

```markdown
# <课程名>

## 符号约定
- <例如：H 表示 Hilbert 空间；内积对第一个变量线性>

## 已讲内容索引
- <日期或讲次>，<PDF 文件名>：<一两句话：讲了哪些定义和定理，用了什么编号>
```

用配置保持符号和定理编号一致，并判断某个结论是否已在前面讲过。但每份笔记都要独立成篇：读者手里没有前几份也能读懂，所以不写“见上次笔记”，而是就地简要重述要用的内容。

如果无法访问文件夹，或用户没有回应，就不用配置继续，在最后的回复里说明。

### 2. 准备工作目录和工具

确认 `typst --version` 可用（模板在 0.15 上测试过）。如果没有：

- 云端工作区：下载到工作目录即可，`curl -sSL https://github.com/typst/typst/releases/latest/download/typst-x86_64-unknown-linux-musl.tar.xz | tar xJ`，二进制在解压出的文件夹里。
- 用户自己的电脑：先征得同意，再安装。macOS 用 `brew install typst`，Windows 用 `winget install --id Typst.Typst`，Linux 用上面的压缩包。

建一个新工作目录，把下面的脚本存为 `render_board.py` 并运行：`python3 render_board.py <输入.pdf> board`。不要改动用户的原始 PDF。

```python
import pathlib, shutil, subprocess, sys

src, out = sys.argv[1], pathlib.Path(sys.argv[2])
out.mkdir(parents=True, exist_ok=True)
LONG_SIDE = 1800  # 够看清字迹，30 页时 PDF 也不会过大

if shutil.which("pdftoppm"):
    subprocess.run(["pdftoppm", "-jpeg", "-jpegopt", "quality=82",
                    "-scale-to", str(LONG_SIDE), src, str(out / "tmp")], check=True)
    pages = sorted(out.glob("tmp-*.jpg"), key=lambda p: int(p.stem.split("-")[-1]))
    for i, p in enumerate(pages, 1):
        p.rename(out / f"p-{i:03d}.jpg")
    count = len(pages)
else:  # pip install pypdfium2 pillow
    import pypdfium2 as pdfium
    pdf = pdfium.PdfDocument(src)
    count = len(pdf)
    for i in range(count):
        page = pdf[i]
        scale = LONG_SIDE / max(page.get_size())
        page.render(scale=scale).to_pil().convert("RGB").save(
            out / f"p-{i+1:03d}.jpg", quality=82)
print(f"{count} pages")
```

### 3. 通读全部板书，切分逻辑块

按页序从第 1 页起，用 Read 逐页查看 `board/p-NNN.jpg`，每一页都要看。符号看不清时，把该区域裁剪放大再看，不要凭印象。

先弄清这节课的主线，再把内容切成**逻辑块**。讲解的单位是逻辑块，不是页。照片在哪里分页，取决于教授什么时候擦黑板、学生什么时候拍照，和数学结构无关：一个证明常常跨两三页，一页上也常常有两三件互不相干的事。如果按页讲，读者就得在几页之间来回翻，自己把同一个论证拼起来，而这正是读者最需要帮助的地方。

一个逻辑块是可以一口气读完的完整单元，例如：

- 一个定义，连同紧跟着它的说明和小例子；
- 一个定理（或引理、命题），连同它的完整证明，不管证明写了几页；
- 一个独立的例子或计算；
- 一段完整的推导，比如从方程推出某个公式。

切分规则：

- **逻辑上连着的放在一起。** 证明写到页底，下一页接着写，就是同一块。定理的陈述在一页、证明在下一页，也是同一块。
- **同一页上不相关的内容分开。** 一页的上半是上一个证明的结尾，下半是新定义，就分属两块。
- **补写的内容归位。** 教授写在别处的内容（角落里的引理、回头补的条件、后面才写的更正），并入它所属的块，并用 `Order changed` 说明它原来在哪一页。
- **引理看分量。** 证明里用到的引理，如果有自己的证明，就单独成块，放在用到它的定理之前；如果只是一句话的事实，就并入。
- **大小适中。** 一行说明不单独成块。很长的证明（比如超过两页）仍是一块，在块内按证明步骤分段（见“块的结构”）。

动笔前先写块计划，存为 `plan.md`，每块一行：标题，用到的板书区域（如 `p2 下半, p3 上半`），在逻辑顺序中的位置。之后按这个计划写，并用它做覆盖检查。

**裁剪板书区域。** 一块只用到一页的一部分时，把这部分裁剪出来，这样每块只显示它自己的内容，同一页的不同部分可以出现在不同的块里。把下面的脚本存为 `crop_board.py`（需要 pillow）：

```python
# 用法：python3 crop_board.py <页码> <名称> <上> <下> [<左> <右>]
# 上下左右是 0 到 1 之间的比例，比如 0.5 1 表示下半页。四周自动留出 3% 的余量。
import pathlib, sys
from PIL import Image

n, name = int(sys.argv[1]), sys.argv[2]
top, bottom = float(sys.argv[3]), float(sys.argv[4])
left, right = (float(sys.argv[5]), float(sys.argv[6])) if len(sys.argv) > 6 else (0.0, 1.0)
pad = 0.03
im = Image.open(f"board/p-{n:03d}.jpg")
w, h = im.size
box = (max(0, left - pad) * w, max(0, top - pad) * h,
       min(1, right + pad) * w, min(1, bottom + pad) * h)
pathlib.Path("board/crops").mkdir(exist_ok=True)
im.crop(tuple(round(v) for v in box)).save(f"board/crops/{name}.jpg", quality=85)
```

例如 `python3 crop_board.py 2 p2-lower 0.45 1`。裁剪后用 Read 查看裁出的图，确认边缘没有切掉字，切到了就把范围放宽。区域分界不清楚，或者没有 pillow 时，就用整页。

### 4. 逐块撰写

不要把全部内容一次写进一个文件。按块计划，每块写一个文件（`parts/01.typ`、`parts/02.typ` ……），写完一块就编译一次，及早发现语法错误。

后面的块和前面的块同样详细，这是最容易出问题的地方：上下文变长后，人总想越写越简略。如果一次做不完，就接着做，不要压缩。

### 5. 校对（见“校对”一节），然后交付

- 输出 PDF，命名为 `<原文件名>-notes.pdf`。存到课程文件夹的 `notes/` 下（不覆盖已有文件），并在当前环境能发送文件时发给用户。
- 更新 `course.md`：追加一条索引，补充新出现的符号约定。
- 最后的回复只写：文件、页数、哪些页有存疑（遮挡推断了什么、疑似板书错误在哪页）。不要复述笔记内容。

## 笔记结构

只有这些内容：

1. **概要**：这节课讲了什么，主线如何展开（一小段或几句话）。
2. 按逻辑顺序，**一块一块**地讲。每块依次是：**原图 → 板书转录 → 讲解**，需要时加图。讲完一块再讲下一块。

不要加这些：动机与历史、条件必要性分析和反例、与应用或代码的联系、练习题、闪卡、考点预测、术语表、总结清单。

### 块的结构

- **标题**：说明这块是什么，如 `Theorem: Banach fixed point`，不写页码。
- **原图**：这块用到的所有板书区域，按逻辑顺序排列。整页都属于这块就放整页 `#board(3)`，只用到一部分就放裁剪图 `#board(3, crop: "p3-upper", where: "upper half")`。同一张照片的不同部分可以出现在不同的块里。
- **转录**：不分步的块只有一个转录（分步的块每步一个，见下面的“长块分步”），按逻辑顺序拼接。转录里换页的地方加 `#pagemark(n)`，读者就知道接下来的内容在哪张照片上。
- **讲解**：先用一两句话说清这块要做什么、策略是什么，再按块内的逻辑顺序逐行讲解。讲解只讲这一块，不提前讲后面块的内容。前面的块已经讲过的结论，直接引用（如 “by the bound proved in the previous block”），不再重讲。

**长块分步。** 如果一块的转录超过大约一页（常见于长证明），就在块内按证明的步骤分段。每一步有一个简短的步骤说明（如 `#step[Step 2: the iterates are Cauchy.]`），后面依次是这一步的原图、转录和讲解。这样读者不用在长转录和后面的讲解之间来回翻。步骤按证明本身的结构来分，不按页分。块开头的策略说明仍然要写，并预告有哪几步。

## 怎么讲

**主线是逐行还原并解释板书。** 沿着教授写的内容走，用教授的符号，即使你觉得别的写法更好。不要换一种证明方法。

**详略由难度决定：**

- 简单的（展开式子、代数整理）：不写。
- 中等的（如 triangle inequality、Cauchy–Schwarz）：一句话带过，点出用了什么。
- 难点：详细展开，写出中间步骤，说明为什么这一步成立、这个条件用在哪里。

判断难度时按读者的实际水平，而不是研究生课程的默认水平。教授写的 “easy to see”、“by a standard argument” 常常正是读者卡住的地方：按上面的标准处理：简单的不补，中等的一句话带过，难的完整补出，补的内容都标上 `Not on the board`。

**讲清思路。** 这是用户最看重的。每个证明或计算开始前，先用一两句话说清楚策略：遇到这类问题第一步做什么、第二步做什么、为什么这样开始。比如“要证序列收敛，但极限未知，所以第一步先证它是 Cauchy 列，第二步用空间的完备性得到极限，第三步验证极限满足要求”。逐行解释时要让读者看到每一步在这个策略里的位置。

**例子和直觉。** 遇到复杂的概念或定理，举一个具体的小例子，能算的就算出来，或给出几何图像。例子要真能说明问题，不为凑数而举。

**就地补先修知识。** 某一步依赖读者可能已经忘记的定义或定理时，用 `Recall` 就地说明它的准确内容和在这里怎么用，两三句话即可。不要写成系统复习，也不要做前后章节的联系。

**语气。** 像耐心的助教，自然地解释，不说“显然”“容易看出”，也不要堆砌客套话。正文以连贯的段落为主，少用列表和小标题，不要过度格式化。

## 语言

- 讲解主体用**英文**。
- **难点附中文**：只在真正难的地方，用 `#zh[...]` 写一小段中文，把关键想法再讲一遍。它不是逐句翻译，大部分内容不需要它。
- **术语**：每份 PDF 中，术语在讲解里首次出现时写成 `紧算子（compact operator）`，用全角括号，之后只用英文。中文译名必须是通行译法；不确定就只写英文，不要自造译名。基础词（set、function 等）不用注。
- 板书转录保持原样，不加中文。

## 如实区分（最重要的规则）

用户最反感幻觉式编造，并且需要分清哪些是教授写的。

- **转录 = 只有板书上的内容。** 不要在转录里修正错误、补全步骤或美化，缩写和省略也照样保留。转录之外的文字都是你写的，模板开头已向读者说明。
- **遮挡或看不清：** 根据上下文推断。转录里用 `#inferred[...]` 包住推断的部分，讲解里用 `#flag("Inferred")[...]` 说明理由。先用数学验证：把候选内容代入，看前后是否成立。如果有多种可能都说得通，就都列出来，说明你选了哪个以及原因。如果完全判断不了，就写 `[illegible]`，并说明缺了什么，不要编。
- **疑似板书有错：** 先放大确认不是你看错了。然后照板书转录并讲解，用 `#suspect[...]` 标出疑点，用 `#flag("Possible board error")[...]` 说明理由，指出正确写法可能是什么，以及它对后面步骤的影响。用“可能”的语气，让用户去向教授核实。
- **补充内容：** 板书没写而你补的步骤用 `#flag("Not on the board")[...]`；先修知识用 `#flag("Recall")[...]`；调整了顺序用 `#flag("Order changed")[...]`。
- **不要编造来源。** 不写教授“说了”什么（你没有录音，只能说板书上写了什么）。定理名只在确定时才用。不给教材页码，不编文献。不确定的结论就说不确定。

## 输出模板

在工作目录里放三样东西：`lib.typ`（下面第一段，不用改）、`main.typ`（第二段，填好标题，按块 `#include`）、`parts/`（每块一个文件，开头都要 `#import "/lib.typ": *`，因为 `#include` 的文件看不到 `main.typ` 里定义的函数）。在工作目录内用 `typst compile --root . main.typ out.pdf` 编译。

`lib.typ`：

```typst
// 原图。n 是 PDF 页码。crop 是 crop_board.py 生成的裁剪图名称，where 说明它是这一页的哪部分。
// 图片按页宽显示，高度超过 11cm 时缩小，竖拍的照片也不会占满整页；扁长的裁剪图不会留下空白。
#let board(n, crop: none, where: none) = {
  let file = if crop == none { "/board/p-" + "000".slice(str(n).len()) + str(n) + ".jpg" } else { "/board/crops/" + crop + ".jpg" }
  block(width: 100%, breakable: false, above: 1em, below: 0.6em)[
    #layout(size => {
      let full = image(file, width: size.width)
      let img = if measure(full, width: size.width).height > 11cm { image(file, height: 11cm) } else { full }
      align(center, img)
    })
    #v(-0.3em)
    #align(center, text(size: 8.5pt, fill: luma(100))[Board, page #n#if where != none [, #where]])
  ]
}

// 长块里的步骤说明。sticky 让它和下面的原图留在同一页，不会单独落在页底。
#let step(body) = block(sticky: true, above: 1.5em, below: 0.4em, text(weight: "bold", body))

// 转录中的换页标记：从这里开始的内容在第 n 页的照片上。
#let pagemark(n) = block(above: 0.7em, below: 0.5em, text(size: 8pt, fill: luma(110), tracking: 0.04em)[(board page #n)])

// 板书转录：只放板书上的内容。
#let transcript(body) = block(width: 100%, breakable: true, stroke: (left: 1.2pt + luma(150)), inset: (left: 10pt, y: 3pt), above: 0.8em, below: 1.1em, body)

// 转录中的标记。
// 用 box 而不是 highlight，因为 highlight 对公式不起作用。文字和公式里都可以用：#suspect[$1 + q$]。
#let inferred(body) = box(fill: rgb("#ffe98a"), outset: (x: 1.5pt, y: 2.5pt), radius: 1.5pt, body)
#let suspect(body) = box(fill: rgb("#ffc4b8"), outset: (x: 1.5pt, y: 2.5pt), radius: 1.5pt, body)

// 讲解中的标注。
#let flag(label, body) = {
  let colors = ("Inferred": rgb("#8a6500"), "Possible board error": rgb("#b3321c"), "Not on the board": rgb("#2b5c99"), "Recall": rgb("#2f6f4f"), "Order changed": rgb("#6a4a99"))
  par[#text(weight: "bold", size: 0.92em, fill: colors.at(label, default: black))[#label.] #body]
}

// 难点的中文说明。
#let zh(body) = par[#text(lang: "zh", region: "cn")[#body]]
```

`main.typ`：

```typst
#let cjk = ("Noto Serif CJK SC", "Source Han Serif SC", "Songti SC", "STSong", "SimSun", "Noto Sans CJK SC", "PingFang SC", "Microsoft YaHei")

#set page(paper: "a4", margin: (x: 2.3cm, y: 2.3cm), numbering: "1")
#set text(font: ("New Computer Modern", ..cjk), size: 11pt, lang: "en")
#set par(justify: true, leading: 0.72em, spacing: 1.1em)
#set heading(numbering: "1.")
#show heading: set block(above: 1.6em, below: 0.9em)

#align(center)[
  #text(size: 15pt, weight: "bold")[COURSE — LECTURE TITLE] \
  #text(size: 9.5pt, fill: luma(90))[Source: FILE NAME]
]

#text(size: 9.5pt, fill: luma(70))[Grey-ruled blocks transcribe the board and contain only what is on it. Everything else is the assistant's explanation. In transcriptions, yellow marks a part inferred because it was unclear or hidden, and red marks a possible error on the board. Check both against the original photo.]

= Overview

// 概要写在这里

// #include "parts/01.typ"
```

每个块文件的写法。这个例子里，定理写在第 3 页下半，证明接着写满第 4 页：

```typst
#import "/lib.typ": *

= Theorem: contraction mapping

#board(3, crop: "p3-lower", where: "lower half")
#board(4)

#transcript[
  *Thm.* Let $(X, d)$ be complete, $T: X -> X$ with $d(T x, T y) <= q d(x, y)$, $q < 1$. \
  Then $T$ has a unique f.p.

  #pagemark(4)
  $ d(x_n, x_m) <= q^n / #suspect[$1 + q$] d(x_0, x_1) $
]

The strategy has three steps ...

#flag("Possible board error")[The denominator should probably be $1 - q$, because ...]

#zh[这里的关键是……]
```

长块分步时，在同一个块文件里重复“步骤说明 → 原图 → 转录 → 讲解”：

```typst
#step[Step 1: build a candidate.]
#board(4, crop: "p4-upper", where: "upper half")
#transcript[ ... ]
Explanation of step 1 ...

#step[Step 2: the iterates are Cauchy.]
#board(4, crop: "p4-lower", where: "lower half")
#transcript[ ... ]
Explanation of step 2 ...
```

图用 matplotlib 生成 PNG（`dpi=200`），存到 `figures/`，用 `#image("/figures/x.png", width: 70%)` 插入。不要依赖 Typst 包（CeTZ 等）：有些环境连不上包仓库。只有在图确实帮助理解时才画，比如函数图像、几何关系、收敛示意。示意图要写明是示意。

### Typst 数学语法（与 LaTeX 不同，常见错误来源）

- 相邻字母要分开：`a b`，不能写 `ab`（`ab` 会被当成未知变量而报错）。函数应用 `T x`、`f(x)` 都可以。
- 多字母名称：`"err"`、`op("supp") f`、`op("ess sup", limits: #true)`。内置的有 `sin cos log exp lim sup inf max min det dim ker`。
- 上下标分组用圆括号：`x_(n+1)`、`e^(-i x)`、`sum_(k=1)^n`、`integral_0^1 f dif x`。
- 符号：`RR ZZ NN CC`，`bb(E)`，`cal(F)`，`partial`，`nabla`，`Delta`，`oo`（无穷），`in`，`subset`，`forall`，`exists`，`<=`，`>=`，`!=`，`->`，`=>`，`|->`，`approx`，`dots`。
- 括号和范数：`norm(x)`，`abs(x)`，`chevron.l x, y chevron.r`（内积），`floor(x)`，`lr(( ... ))`。
- 结构：`frac(a, b)` 或 `a / b`，`sqrt(x)`，`hat(x)`，`tilde(x)`，`overline(x)`，`vec(a, b)`，`mat(a, b; c, d)`，`cases(1 "if" x > 0, 0 "otherwise")`。
- 公式内的文字和中文要加引号：`"for all" x`。显示公式是 `$ ... $`（美元符号内侧有空格），行内是 `$...$`。
- 对齐用 `&`，换行用 `\`。
- 文本中的 `#`、`$`、`*`、`_`、`@`、`<` 要用反斜杠转义。

报错时读错误信息，改对应的行，不要为了通过编译去改动数学内容。`unknown font family` 警告可以忽略：模板列出了各系统的中文字体，只要其中一个存在就行。如果 `typst fonts` 里一个中文字体都没有，中文会显示成方框，这时要告诉用户，并请用户安装 Noto Serif CJK SC。

## 校对

交付前全部做完：

1. **覆盖：** 对照 `plan.md`，确认每一页的每个区域都归入了某个块，没有遗漏，也没有两块重复讲同一段内容。再确认每一页都至少出现一次，如 `for i in $(seq 1 N); do grep -Eq "#board\($i[,)]" parts/*.typ || echo "missing $i"; done`。
2. **块的完整性：** 每块从头读到尾，应当是一个完整的论证，不需要翻到别的块去拼。如果一个证明被切成了两块，或者一块里混了两件不相关的事，就重新切分。转录里每次换页都有 `#pagemark`。
3. **转录对照：** 把编译好的 PDF 渲染成图，逐页和板书原图比对。逐个符号看：上下标、正负号、不等号方向、求和范围。
4. **数学核对：** 自己重新推一遍每一步。能用计算验证的（代数、积分、例子里的数）就用 sympy 或数值方法实际跑一下。如果你的解释说不通，要么是你看错了，要么是板书有问题；不要写一段听起来合理但没有推通的话。
5. **标记核对：** 每处 `#inferred` 和 `#suspect` 在讲解里都有对应理由，反之亦然。
6. **版面和语言：** 中文显示正常（没有方框），图片顺序正确，没有内容溢出页边。术语只在首次出现时带中文。

## 作业题

用户单独发作业题时，不生成板书笔记，直接在回复里回答，除非用户要 PDF。先后给出：

1. **解题思路提示：** 这是哪类问题，第一步做什么、第二步做什么，为什么这样想，关键工具是什么。先写这部分，让用户可以停下来自己试。
2. **英文标准解答：** 严谨、完整，可以直接参考的英文写法，每一步给出理由。沿用课程的符号约定。

如果题目有歧义或条件不足，先说明，不要默默改题。解答没有把握时明确说。

## 后续追问

用户对笔记追问时，沿用这门课的配置和本次对话已讲的内容回答，保持同样的语言和详略标准。如果追问发现笔记有错，直接承认，并更新 PDF。
