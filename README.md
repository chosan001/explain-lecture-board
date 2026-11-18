# Explain Lecture Board · 板书讲解

[English](#english) | [简体中文](#简体中文)

## English

A Claude skill that turns a PDF of a professor's blackboard photos into lecture notes you can actually follow. It reads every page, transcribes the board faithfully, and explains the lecture line by line in a single PDF typeset with [Typst](https://typst.app).

It was written for Chinese students in English-taught graduate math programs, so by default the explanation is in English with short Chinese notes on the hard parts. It works for any math course, and you can [switch it to English only](#switching-to-english-only).

See [`examples/sample-notes.pdf`](examples/sample-notes.pdf), generated from [`examples/sample-board.pdf`](examples/sample-board.pdf). The sample board was typeset, not handwritten, and contains a deliberately hidden symbol and a deliberate error, so you can see how both are marked.

### What you get

One PDF per lecture, containing:

1. **Overview:** what the lecture covers and how the argument unfolds.
2. The lecture explained **block by block**. A block is one logical unit: a definition, a theorem with its whole proof, an example, a derivation. Blocks follow the logic of the lecture, not the photo pages: a proof that runs over three pages is one block, and a page holding the end of one proof and the start of a new definition is split between two blocks. Each block contains:
   - **The board photos it uses**, cropped to the relevant part of each page, so you can always check against the board.
   - **A transcription** of exactly what is on the board, nothing added or corrected, with markers where it moves to the next photo page.
   - **An explanation** in the order the professor wrote it, in the professor's notation.

   A long proof is split into its steps, and each step shows its own photo, transcription, and explanation, so you never have to flip back and forth.

How the explanation works:

- **The strategy first.** Before each proof or computation, it says what to do first and what to do second, and why the argument starts there.
- **Depth follows difficulty.** Routine algebra is skipped. Standard steps such as the triangle inequality get one sentence. Hard steps are worked out in full, including steps the board dismisses as "easy to see".
- **Examples and intuition** for difficult concepts, computed where possible, with figures when a picture helps.
- **Prerequisites recalled in place.** When a step relies on a definition or theorem you may have forgotten, its exact statement is given right there.
- **Language.** The main text is in English. By default, a technical term appears once with its Chinese name, as in 紧算子（compact operator）, and only in English after that, and hard passages get a short paragraph in Chinese.

### Reading the notes

| Mark | Meaning |
| :- | :- |
| Grey rule on the left | Transcription. Only what is on the board. |
| Yellow highlight | Part of the board that was hidden or unclear, inferred from context. |
| Red highlight | Possible error on the board. It is transcribed as written, explained, and flagged for you to check with the professor. |
| **Inferred.** | Why the yellow part was read that way. |
| **Possible board error.** | Why the red part looks wrong, what it probably should be, and whether later steps are affected. |
| **Not on the board.** | A step the board skipped, filled in by the assistant. |
| **Recall.** | A prerequisite definition or theorem. |
| **Order changed.** | Content moved because the board was written out of order. |

Everything outside the transcription is written by the assistant, not the professor.

### Requirements

The assistant sets these up as needed and asks before installing anything on your computer.

- **Typst** 0.15 or later, to compile the notes.
- **Python 3**, plus either `pdftoppm` (from Poppler) or the Python packages `pypdfium2` and `pillow`, to turn the PDF into page images. Cropping the photos needs `pillow`; without it, whole pages are shown.
- **A Chinese font**, such as Noto Serif CJK SC, for the Chinese notes. macOS and Windows already include one. Not needed in English-only mode.

### Installation

#### Claude app (web, desktop, mobile)

1. Download `explain-lecture-board.zip` from the [Releases](https://github.com/chosan001/explain-lecture-board/releases) page. Or download this repository and zip the `skills/explain-lecture-board` folder yourself, so the ZIP contains that folder at its top level.
2. In Claude, open **Customize > Skills**, upload the ZIP, and turn the skill on.

Skills enabled in your Claude account also load in Claude Code when you sign in with the same account.

#### Claude Code, as a plugin

```
/plugin marketplace add chosan001/explain-lecture-board
/plugin install explain-lecture-board@lecture-board
```

#### Claude Code, by copying the folder

```bash
git clone https://github.com/chosan001/explain-lecture-board.git
mkdir -p ~/.claude/skills
cp -r explain-lecture-board/skills/explain-lecture-board ~/.claude/skills/
```

Install it one way only. Two copies of the same skill show up as two commands.

### Usage

The skill starts only when you call it. Uploading a PDF alone does not trigger it.

```
/explain-lecture-board PDE, today's lecture
```

Attach the PDF, or give its path in Claude Code. If you installed it as a plugin, the command is `/explain-lecture-board:explain-lecture-board`.

**Course folders.** The first time you use the skill for a course, it asks where that course's folder is. In the desktop app, connect the folder from the **+** menu beside the message box. The skill keeps a `course.md` there with the course's notation and a short index of past lectures, and saves notes to `notes/`. Each set of notes still stands on its own. You can edit `course.md` yourself. Without a folder, the notes are still produced but nothing carries over to the next lecture.

**Homework.** Call the skill and send a problem. You get a hint first (what kind of problem it is, the first and second steps, the key tool), then a complete model solution in English.

**Follow-up questions.** Ask in the same conversation, for example "Why does the second inequality on page 5 hold?" If a question reveals a mistake in the notes, the PDF is corrected.

### Switching to English only

By default, the notes add Chinese names to technical terms and a short Chinese paragraph on hard passages. There are two ways to turn that off.

**For one lecture.** Say so when you call the skill. This lasts for that conversation only.

```
/explain-lecture-board PDE, today's lecture. Write the notes entirely in English, with no Chinese.
```

**Permanently.** Edit the skill file, `SKILL.md`, which is written in Chinese.

1. Find the section headed `## 语言` (Language). It has four bullet points.
2. Replace the four bullet points with:

   ```markdown
   - Write the whole explanation in English.
   - Do not add Chinese paragraphs, and do not use `#zh[...]`.
   - Do not add Chinese names to technical terms.
   - Keep the transcription exactly as it is on the board.
   ```

3. In the section headed `## 校对` (Checks), the last item ends with the sentence `术语只在首次出现时带中文。`, which means "terms carry Chinese only at their first appearance". Delete that sentence so the checklist does not ask for Chinese.
4. Optional: the first paragraph under `# 板书讲解` describes the reader as a Chinese international student. Rewrite it to describe yourself. The assistant uses it to judge how much to explain.

Where the file is depends on how you installed the skill:

- **Claude app:** unzip `explain-lecture-board.zip`, edit `explain-lecture-board/SKILL.md`, zip the folder again so it is at the top level, and upload it in place of the old version.
- **Claude Code, copied folder:** edit `~/.claude/skills/explain-lecture-board/SKILL.md`.
- **Claude Code, plugin:** don't edit the installed copy, because updates overwrite it. Fork this repository, edit `skills/explain-lecture-board/SKILL.md` in your fork, and install from your fork with `/plugin marketplace add YOUR-GITHUB-USERNAME/explain-lecture-board`.

### Customizing

All behavior is in [`skills/explain-lecture-board/SKILL.md`](skills/explain-lecture-board/SKILL.md). Besides `## 语言` (Language), the section most worth changing is `## 输出模板` (Output template), which controls the note layout.

### Limitations

- The skill has been tested on typeset sample boards, not yet on real handwriting. Compare the transcription with the photo, especially for subscripts, signs, and inequality directions.
- Hidden or unclear content is inferred and marked. An inference can still be wrong.
- A long lecture of 20 to 30 pages takes a while to process.

### A note on use

Lecture materials may be covered by your university's copyright or course rules. Keep notes made from them for your own study unless you are allowed to share them. For homework, follow your course's academic integrity policy.

### License

[MIT](LICENSE)

---

## 简体中文

这是一个 Claude skill。你提供教授板书照片组成的 PDF，它逐页读完，忠实转录板书，再沿着板书逐行讲解，最后用 [Typst](https://typst.app) 排版成一份 PDF 笔记。

它是为在英语授课的数学研究生项目里读书的中国留学生写的，所以讲解主体是英文，难点附一小段中文。任何数学课都能用。

示例见 [`examples/sample-notes.pdf`](examples/sample-notes.pdf)，由 [`examples/sample-board.pdf`](examples/sample-board.pdf) 生成。示例板书是电脑排版的，不是手写，里面故意遮住了一个符号、写错了一个式子，可以看看这两种情况分别怎么标注。

### 你会得到什么

每节课一份 PDF，内容如下：

1. **概要：** 这节课讲了什么，论证如何展开。
2. **一块一块**地讲。一块是一个逻辑单元：一个定义、一个定理连同它的完整证明、一个例子、一段推导。块按讲课的逻辑划分，不按照片分页：跨三页的证明是一块；一页上半是上个证明的结尾、下半是新定义，就分属两块。每块依次是：
   - **原图：** 这块用到的板书，裁剪到每页的相关部分，随时可以对照。
   - **板书转录：** 只有板书上的内容，不增补，不修正；换到下一张照片的地方有标记。
   - **讲解：** 按教授的书写顺序，用教授的符号。

   长证明按步骤拆开，每一步有自己的原图、转录和讲解，不用来回翻。

讲解的方式：

- **先讲思路。** 每个证明或计算开始前，先说清楚第一步做什么、第二步做什么，以及为什么从这里入手。
- **详略看难度。** 简单的代数整理不写；triangle inequality 这类常规步骤一句话带过；难点完整展开，包括板书上写着 “easy to see” 的地方。
- **例子和直觉：** 复杂的概念或定理配一个具体例子，能算的就算出来，需要时配图。
- **就地补先修知识：** 某一步依赖你可能忘了的定义或定理时，当场给出准确表述。
- **语言：** 正文用英文。术语首次出现时附中文，写成「紧算子（compact operator）」，之后只用英文。难点附一小段中文说明。

### 怎么看笔记

| 标记 | 含义 |
| :- | :- |
| 左侧灰色竖线 | 板书转录，只有板书上的内容。 |
| 黄色底色 | 被遮挡或看不清，根据上下文推断的部分。 |
| 红色底色 | 疑似板书有错。照原样转录并讲解，同时标出来，请你向教授核实。 |
| **Inferred.** | 黄色部分为什么这样推断。 |
| **Possible board error.** | 红色部分为什么可疑，正确写法可能是什么，对后面的步骤有没有影响。 |
| **Not on the board.** | 板书省略、由 AI 补上的步骤。 |
| **Recall.** | 补充的先修定义或定理。 |
| **Order changed.** | 板书不是按顺序写的，内容调整了位置。 |

转录以外的文字都是 AI 写的，不是教授写的。

### 运行环境

需要的工具由 AI 按需配置；在你的电脑上安装任何东西之前，它都会先问你。

- **Typst** 0.15 或更高版本，用来编译笔记。
- **Python 3**，以及 `pdftoppm`（Poppler 自带）或 Python 包 `pypdfium2` 和 `pillow` 二选一，用来把 PDF 转成图片。裁剪照片需要 `pillow`，没有时显示整页。
- **中文字体**，如 Noto Serif CJK SC。macOS 和 Windows 自带中文字体。

### 安装

#### Claude 应用（网页版、桌面端、手机）

1. 在 [Releases](https://github.com/chosan001/explain-lecture-board/releases) 页面下载 `explain-lecture-board.zip`。也可以下载整个仓库，自己把 `skills/explain-lecture-board` 文件夹压缩成 ZIP，ZIP 打开后第一层必须就是这个文件夹。
2. 在 Claude 中打开 **Customize > Skills**，上传 ZIP，并启用这个 skill。

在 Claude 账号里启用的 skill，用同一账号登录 Claude Code 时也会自动加载。

#### Claude Code：作为插件安装

```
/plugin marketplace add chosan001/explain-lecture-board
/plugin install explain-lecture-board@lecture-board
```

#### Claude Code：直接复制文件夹

```bash
git clone https://github.com/chosan001/explain-lecture-board.git
mkdir -p ~/.claude/skills
cp -r explain-lecture-board/skills/explain-lecture-board ~/.claude/skills/
```

以上方式只选一种。装了两份，命令列表里就会出现两个同样的 skill。

### 使用

这个 skill 只在你手动调用时启动，只上传 PDF 不会触发它。

```
/explain-lecture-board PDE 这是今天的板书
```

把 PDF 作为附件上传；在 Claude Code 里也可以直接写文件路径。如果是作为插件安装的，命令是 `/explain-lecture-board:explain-lecture-board`。

**课程文件夹：** 第一次为某门课使用时，它会问这门课的文件夹在哪里。用桌面端时，在输入框旁的 **+** 菜单里连接这个文件夹。它会在文件夹里维护一个 `course.md`，记录这门课的符号约定和已讲内容的简短索引，笔记存到 `notes/` 下。每份笔记仍然独立成篇。`course.md` 你也可以自己改。不连接文件夹也能生成笔记，只是下一节课用不上这次的记录。

**作业题：** 调用 skill 后发题目。它先给提示（这是哪类问题、第一步和第二步做什么、关键工具是什么），再给完整的英文标准解答。

**追问：** 在同一个对话里直接问，比如「第 5 页第二个不等式为什么成立？」如果追问发现笔记有错，它会改正 PDF。

### 自定义

所有行为都写在 [`skills/explain-lecture-board/SKILL.md`](skills/explain-lecture-board/SKILL.md) 里。想换讲解语言，改「语言」一节；想改笔记版式，改「输出模板」一节。

### 局限

- 目前只用电脑排版的示例板书测试过，还没测过真实的手写板书。请对照原图检查转录，尤其是上下标、正负号和不等号方向。
- 遮挡或看不清的内容是推断出来的，都有标注，但推断仍可能出错。
- 20 到 30 页的长课需要处理一段时间。

### 使用须知

课堂材料可能受学校版权或课程规定约束。除非允许分享，用它们做的笔记请仅供自己学习。做作业时请遵守课程的学术诚信规定。

### 许可证

[MIT](LICENSE)
