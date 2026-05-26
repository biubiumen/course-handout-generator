# 复习讲义生成器 (Review Handout Generator)

一个用于 **Claude Code** 的技能，根据教材、PPT、往年考卷等资料自动生成系统化的 LaTeX 复习讲义 PDF。

## 能做什么

- 读取教材 PDF、PPT、习题册、考卷，提取知识点清单
- 按逻辑递进分章编写 LaTeX 讲义（每章一个 PDF）
- 自动融入例题、易错提醒、做题模板
- 多轮并行 agent 审查：内容完整性 → 范围合规 → 结构体系
- 最终产出：格式统一的 PDF 讲义 + zip 打包

## 安装

1. 下载 `review-handout-generator.skill` 文件
2. 在 Claude Code 中安装：
   ```bash
   claude skills install review-handout-generator.skill
   ```
3. 或者直接放到 skills 目录：
   ```
   ~/.claude/skills/review-handout-generator/
   ```

## 使用方式

在 Claude Code 会话中说类似的话即可触发：

- "帮我把这本教材整理成复习讲义"
- "根据 PPT 和往年考卷出考前复习资料"
- "整理一下这一章的知识点，出个 PDF"

## 使用示例

```
用户：我有一本高数教材和660题，要考研数三，帮我整理复习讲义

Claude 会：
1. 先确认范围（数三考纲、以哪本教材为主）
2. 规划章节（函数极限→导数→积分→微分方程→多元→二重积分→级数）
3. 逐章生成 .tex 并编译 PDF
4. 三轮审查确保无超纲、无遗漏、体系完整
5. 打包输出
```

## 产出质量原则

- **教材没写的不加，考纲不考的不写**
- **所有内容融入正文，不搞"综合知识补充"等拼凑附录**
- **例题紧跟知识点，易错提醒放在对应位置**
- **多轮审查：完整性 → 合规性 → 结构性**

## 文件结构

```
review-handout-generator/
├── SKILL.md                    # 技能主体（方法论 + 完整流程）
├── README.md                   # 本文件
└── references/
    ├── latex_template.tex      # LaTeX 模板（ctexart，蓝色主题）
    └── review_agents.md       # 审查 agent prompt 模板
```

## 依赖

- XeLaTeX（中文编译）
- Python 3（辅助脚本）
- Claude Code（技能运行环境）

## 适用场景

- 考研数学（数一/数二/数三）知识点整理
- 大学期末考试复习资料
- 专业课（信息论、最优化等）讲义制作
- 任何"有教材 + 有考卷 → 需要系统化复习资料"的场景

## License

MIT
