# 复习讲义生成器 (Review Handout Generator)

一个用于 **Claude Code** 的技能，根据教材、PPT、往年考卷等资料自动生成系统化的 LaTeX 复习讲义 PDF。

## 能做什么

- **mineru 结构化解析**：PDF → Markdown + LaTeX 公式 + 表格 + 提取图片，保留数学内容完整性
- **视觉模型按需补位**：对关键图表（函数图像、几何示意图等）批量送入视觉模型获取语义描述
- **系统化分章编写**：按逻辑递进分章，每章独立 .tex，知识点 + 做题模板 + 例题一体化
- **合并审查 + 最终核验**：每章一个 agent 同时检查完整性/范围/结构，问题一次性修改
- **最终产出**：格式统一的 PDF 讲义 + zip 打包

## 安装

1. 克隆仓库：
   ```bash
   git clone git@github.com:biubiumen/review-handout-generator.git
   ```
2. 放到 Claude Code skills 目录：
   ```
   ~/.claude/skills/review-handout-generator/
   ```
3. 首次使用前运行**环境检测**（详见下一节）

## 部署后环境检测

本 skill 首次部署到新机器上需要验证以下依赖。将 SKILL.md 中「环境检测」的 4 步依次执行：

| 依赖 | 必要性 | 用途 |
|------|--------|------|
| `mineru` skill | **必需** | PDF 结构化解析（OCR、LaTeX 公式、表格、图片提取） |
| `vision-support` skill | **按需** | 关键图表的语义理解，额度有限仅必要时调用 |
| XeLaTeX + ctex | **必需** | 编译中文 LaTeX 讲义 |
| Claude Code | **必需** | 技能运行环境 |

```bash
# 快速自检
xelatex --version          # LaTeX 编译器
kpsewhich ctexart.cls      # 中文支持
# mineru / vision-support 在 Claude Code 会话中调用对应 skill 验证
```

## 使用方式

在 Claude Code 会话中说类似的话即可触发：

- "帮我把这本教材整理成复习讲义"
- "根据 PPT 和往年考卷出考前复习资料"
- "整理一下这一章的知识点，出个 PDF"

skill 会先确认考试范围、参考教材、习题来源等 5 个问题，然后按流程执行。

## 工作流程

```
阶段〇：素材预处理
  mineru 解析 PDF → Markdown + 提取图片
  → 按需筛选关键图片 → vision-support 批量描述
  → 得到 chXX_parsed.md（唯一素材源）

阶段一：素材分析与规划
  提取知识点清单 → 确定边界 → 规划章节 → 用户确认

阶段二：分章并行编写
  每章一个 .tex → 使用 latex_template.tex → 并行生成

阶段三：审查与修改
  合并审查（完整性 + 范围 + 结构，一次性完成）
  → 批量修改 → 最终核验

阶段四：最终整理
  全局替换敏感词 → 清编译垃圾 → 打包 PDF
```

## 使用示例

```
用户：我有一本高数教材和660题，要考研数三，帮我整理复习讲义

Claude 会：
1. 环境检测（mineru / vision-support / LaTeX 是否就绪）
2. 阶段〇：mineru 解析每章 PDF，按需调 vision-support 描述关键图表
3. 前置确认：数三范围、以哪本教材为主、习题来源、章节规划
4. 阶段一：从 chXX_parsed.md 提取知识点，规划章节（函数极限→导数→积分→...）
5. 阶段二：逐章生成 .tex 并编译 PDF
6. 阶段三：合并审查（完整性→范围→结构） + 最终核验
7. 阶段四：打包输出
```

## 产出质量原则

- **教材没写的不加，考纲不考的不写**
- **所有内容融入正文，不搞"综合知识补充"等拼凑附录**
- **例题紧跟知识点，易错提醒放在对应位置**
- **例题只从用户提供的习题/考卷选取或改编**
- **mineru 是主力，vision-support 是后备，视觉模型按需调用不浪费**

## 文件结构

```
review-handout-generator/
├── SKILL.md                    # 技能主体（完整方法论 + 环境检测 + 阶段流程）
├── README.md                   # 本文件
└── references/
    ├── latex_template.tex      # LaTeX 模板（ctexart，蓝色主题）
    └── review_agents.md        # 审查 agent prompt 模板（合并审查 + 最终核验）
```

## 依赖

| 依赖 | 必要性 | 说明 |
|------|--------|------|
| mineru skill | 必需 | PDF → Markdown + LaTeX + 图片提取 |
| vision-support skill | 按需 | 关键图表语义描述 |
| XeLaTeX | 必需 | 中文 LaTeX 编译 |
| ctex 宏包 | 必需 | 中文排版支持 |
| Claude Code | 必需 | 技能运行环境 |

## 适用场景

- 考研数学（数一/数二/数三）知识点整理
- 大学期末考试复习资料
- 专业课（信息论、最优化等）讲义制作
- 任何"有教材 + 有考卷 → 需要系统化复习资料"的场景

## License

MIT
