---
name: course-handout-generator
description: |
  根据教材、PPT、考卷等资料生成系统化 LaTeX 课程讲义 PDF。当用户提到"复习讲义""整理知识点""做复习资料""考前总结""出复习PDF"等，或提供教材/PPT/考卷要求整理知识点时使用。支持自定义考试范围（考研数一/数三、期末考试等），产出格式统一的中文数学/专业课课程讲义。
---

# 课程讲义生成器

根据用户提供的素材（教材 PDF、PPT、往年考卷、习题册等），按照系统化方法论产出 LaTeX 课程讲义 PDF。

## 环境检测

本 skill 依赖以下工具链，**首次部署或在新机器上使用前必须逐项验证**。缺少任何一项都会导致后续阶段无法正常运行。

### 必需依赖

| 依赖 | 用途 | 必要性 | 检测命令 | 预期输出 |
|------|------|--------|----------|----------|
| `mineru` skill | PDF 结构化解析（OCR、LaTeX 公式提取、表格识别） | **必需** | 调用 mineru skill 对任意 PDF 解析 | 返回结构化 Markdown |
| `vision-support` skill | 图片/示意图/复杂图表的语义理解 | **按需** | 传入一张含公式或图表的图片让视觉模型描述 | 返回自然语言描述 |
| LaTeX 编译器 | 编译 .tex 为 PDF | **必需** | `xelatex --version` 或 `pdflatex --version` | 版本号 ≥ 3.x |
| `ctex` 宏包 | 中文 LaTeX 支持 | **必需** | `kpsewhich ctexart.cls` | 返回文件路径 |
| 中文字体 | PDF 中正确渲染中文 | **必需** | 编译测试文档并用 `pdftotext` 提取文字验证 | 中文正常显示 |

### 检测流程

开始任务前，按顺序执行：

```
1. xelatex --version          → 确认 LaTeX 可用
2. kpsewhich ctexart.cls      → 确认中文支持已安装
3. 编译测试文档并验证中文字体  → 确认 PDF 能正确渲染中文（见下方详细步骤）
4. 调用 mineru skill          → 确认 PDF 解析能力可用
5. 调用 vision-support skill  → 确认视觉模型可用（可选，仅需按需调用时）
```

**中文字体渲染测试**（第3步，关键！）：

```bash
# 创建最小测试文档
echo '\documentclass[UTF8,10pt]{ctexart}\begin{document}中文测试\end{document}' > /tmp/font_test.tex
# 编译
xelatex -interaction=nonstopmode -output-directory=/tmp /tmp/font_test.tex
# 提取文字验证中文是否渲染
pdftotext /tmp/font_test.pdf - | grep "中文测试"
```

如果 `pdftotext` 输出包含"中文测试"，说明字体正常。如果是空白或乱码，则中文字体有问题，需修复后再继续。

**常见中文字体问题处理**：
- ctex 默认不指定 `fontset` 时会自动检测操作系统。如果自动检测失败：
  - 尝试 `fontset=fandol`（TeX Live 自带中文开源字体，跨平台可用）
  - 或 `fontset=ubuntu`（Ubuntu/Debian 系统）
  - 安装缺失字体：`sudo apt install texlive-lang-chinese`

**常见缺失处理**：
- LaTeX 缺失 → `sudo apt install texlive-xetex texlive-latex-recommended texlive-latex-extra`
- ctex 缺失 → `sudo apt install texlive-lang-chinese`
- mineru 未配置 → 参考 mineru skill 文档完成 API 配置
- vision-support 未配置 → 参考 vision-support skill 文档完成模型接入

> **注意**：如果 mineru 或 vision-support 不可用，本 skill 仍可运行但会退化为基础 PDF 文本提取，公式和图表质量会显著下降。**建议先配置好再使用。**

## 前置确认

开始前向用户确认以下信息（用户可能已在对话中提供部分）：

1. **考试范围** — 如数三、数一、某校期末考试等，这决定内容边界
2. **参考教材** — 以哪本教材/PPT 为主要依据
3. **习题来源** — 例题和练习题从哪选取（660题、课后题、考卷等）
4. **输出章数** — 预计分几章，每章覆盖什么主题
5. **特殊模块** — 如数三的经济应用、差分方程等专属内容
6. **视觉调用上限** — 每份 PDF 最多用视觉模型描述几张图片（默认 10 张，额度紧张可下调）

**重要原则**：
- 教材没提到的知识点一律不加
- 考试不考的内容一律不加
- 例题只从用户提供的习题/考卷中选取或改编
- 不使用具体机构名/教师名，统称"高等数学教材"等通用说法

## 工作流程

### 阶段〇：素材预处理

在分析知识点之前，必须先将原始 PDF/PPT/图片转化为结构化文本。**本 skill 的核心工具链是 mineru + vision-support（按需）。**

1. **mineru 批量解析 PDF**
   - 使用 `mineru-open-api extract` 支持批量传入：`mineru-open-api extract *.pdf -o ./parsed/`
   - 输出结构化 Markdown，自动识别并保留 LaTeX 数学公式（行内 + 块级）、表格结构、阅读顺序
   - 每章产物命名为 `chXX_parsed.md`

2. **视觉模型按需补位** ⚠️ 额度有限，仅必要时调用
   - 通读 mineru 每章输出的 `chXX_parsed.md`，对其中引用的图片逐一判断：图中信息能否从上下文推断？
   - **需要调用**的场景（满足任一）：
     - 图包含关键几何关系/函数曲线，上下文未用文字说明
     - 流程图/思维导图，不看图无法理解逻辑结构
     - mineru 输出中明显有公式/表格区域乱码或缺失
   - **跳过**的场景：
     - 上下文已将图的信息用文字完整表述
     - 纯装饰性图片、封面、页眉页脚
     - 图中内容简单可推断（如「如图1-3所示的坐标系」且上下文已描述坐标轴）
   - 确认需要后，按优先级排序（关键图表 > 辅助图表），**每份 PDF 调用上限为前置确认中约定的数量（默认 10 张）**，超出则截断，只取最重要的
   - 将对应图片传入 `vision-support` skill（`node vision.mjs <图片路径> "描述这张图"`），获取自然语言描述
   - 将描述文字回填到 `chXX_parsed.md` 对应位置

3. **合并产物**
   - 每章最终得到一个 `chXX_parsed.md`，包含：mineru 结构化文本 + （按需）视觉模型图片描述
   - 这个文件是后续阶段一（知识点提取）和阶段二（讲义编写）的**唯一素材源**

> **关键**：mineru 是主力，负责把公式变回 LaTeX。vision-support 是后备，只在 mineru 输出不足以理解图片时才调用。**不要无差别把所有图都喂给视觉模型。**

### 阶段一：素材分析与规划

1. **读取素材** — 读取阶段〇产出的所有 `chXX_parsed.md`，提取知识点清单
2. **确定边界** — 根据考试范围，明确哪些知识点入、哪些不入
3. **规划章节** — 按逻辑递进排列章节顺序，每个知识点标注来源（哪份素材的第几页）
4. **输出规划给用户确认** — 列出章节目录和每章覆盖的知识点，用户确认后再开始编写

### 阶段二：分章并行编写

每章一个 `.tex` 文件，并行生成。使用 `references/latex_template.tex` 作为模板。

**文件命名**：`ch0X_章节名.tex`，如 `ch01_functions_limits.tex`

**每章结构**：
```
\section{第X章：章节名}
  概述（一两句话说明本章核心）
\subsection{知识点1}
  \subsubsection{概念/定义}
  \subsubsection{做题模板（如适用）}
  \subsubsection{例题演示}
\subsection{知识点2}
  ...
\section{学习检查清单}（可选，放在章末）
```

**严格要求**：
- 不得出现"综合知识补充""附录""典型综合题解析""常见误区集中警示"等独立章节名
- 补充知识、进阶技巧、易错点全部融入对应知识点小节内
- 例题紧随知识点，不要集中堆在末尾
- warning 提示放在对应知识点旁边，不要集中罗列

**内容要求**：
- 每个知识点配至少 1 个例题
- 例题要有完整推导过程，不能只写答案
- 关键公式用 `\boxed{}` 或 `\key{}` 突出
- 易错点用 `\warning{}` 标注

### 阶段三：审查与修改

审查采用分章并行 agent，agent **只读不写**，输出问题清单。三轮审查合并为一轮，每章一个 agent 一次性检查全部三个维度。

#### 合并审查（一次性完成）

启动 N 个并行 agent（每章一个），每个 agent 同时检查以下三个维度：

**维度一：内容完整性**
- 对比素材，是否有遗漏的知识点
- 例题是否覆盖各类考点
- 公式是否正确

**维度二：范围合规**
- 是否有超纲内容（标注具体行号和原因）
- 是否有教材/习题中没出现过的知识点
- 是否有机构名/教师名残留
- 例题难度是否匹配考试水平

**维度三：结构体系**
- 是否存在独立附录/综合补充章节
- 章节顺序是否逻辑递进
- 副标题、studybox 中是否有范围外的内容描述
- 是否有内容重复（同一知识点在多处出现）
- 补充知识是否已融入正文对应位置

Agent prompt 参见 `references/review_agents.md` 中的"合并审查"模板。

#### PDF 可视验证（视觉渲染抽查）

⚠️ 文本审查无法发现字体缺失、公式截断、布局错位等渲染问题。**必须在修改编译后抽查 PDF 的实际渲染效果**。

1. 每章随机抽 2-3 页（必须包含封面页、正文公式密集页、例题页），用 `pdftoppm` 转为图片：
   ```bash
   pdftoppm -f <页码> -l <页码> -r 150 -png <pdf文件> /tmp/review_page
   ```
2. 将图片传入 `vision-support` skill，提示词：
   > "检查这张课程讲义 PDF 截图：(1) 中文字符是否全部正常显示（有无方块/空白/叠字）；(2) LaTeX 数学公式是否渲染完整；(3) 排版布局是否正常（无文字越界/重叠）;(4) 颜色和框线是否正常。逐一说明。"
3. 如有问题，立即修复 .tex 源文件并重新编译验证，直到所有抽查页通过

#### 修改 + 最终核验（循环至通过）

1. **批量修改** — 汇总所有 agent 的问题清单，一次性修改所有 .tex 文件，同时全局搜索替换机构名/教师名为通用说法，然后重新编译所有 PDF
2. **最终核验** — 每章一个 agent 做关门检查：对照前置确认的原始需求，确认旧问题全部修复且无新增问题。核验模板参见 `references/review_agents.md` 中的"最终核验"模板
3. **如果有问题** → 回到步骤 1，只对有问题的章重新修改和编译，然后再次核验，直到所有章全部通过

### 阶段四：最终清理

1. 删除 LaTeX 编译垃圾文件（.aux, .log, .out, .toc, .synctex.gz, .fdb_latexmk, .fls）
2. 最终文件结构：
   ```
   项目目录/
   ├── 课程讲义PDF/     ← 编译输出，审查和分发同一份
   └── tex/             ← 所有 .tex 源文件
   ```

## LaTeX 模板

完整模板见 `references/latex_template.tex`。关键样式：
- 文档类：`ctexart`，**不指定 `fontset`（由 ctex 自动检测系统字体）**，字号 `10pt`
- 页面：A4，margin=2.05cm
- 颜色：mainblue(RGB 0,77,128) / softblue(RGB 235,247,255) / warn(RGB 150,70,0)
- 自定义命令：`\key{关键词}` 蓝色加粗、`\warning{提醒}` 棕色加粗
- studybox 环境：softblue 背景色的提示框

## 审查 Agent 模板

审查采用合并审查 + 最终核验两步。每次启动 N 个并行 agent，每个负责一章。Agent prompt 参见 `references/review_agents.md`。
