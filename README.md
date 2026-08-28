# Paper Claim Audit：科学论文主张核查 Skill

`paper-claim-audit` 用于核查科学论文中的主张、数值参数和模型设置。它不止返回论文链接，而是把当前项目配置、实现代码、原始论文、补充材料和被引用的前序工作串成可审计的证据链。

默认采用 **arXiv 源码优先、同版本 PDF 复核** 的双通道流程，并把决定性原文制作成直接显示在回答中的 PDF 证据卡。

## 适用场景

- 核查某个数值、公式或模型假设是否真的出现在论文中；
- 追踪一个参数由哪篇论文、补充材料或作者配置首次定义；
- 比较论文描述、当前项目配置和实际代码行为；
- 判断配置文件中的参数是否启用，以及单位、归一化和适用范围；
- 为容易产生误报的科学结论提供可复查的原文证据。

普通的论文检索、BibTeX 修复或常规论文摘要不属于这个 skill 的主要范围。

## 触发方式

显式触发：

```text
使用 $paper-claim-audit 核查 GALPROP 中这个扩散系数的来源，并给出原文证据。
```

当请求明显涉及“核查参数是否正确”“追踪论文来源”“比较论文与代码实现”等目标时，Codex 也可以根据 `SKILL.md` 的描述自动触发。

## 核查流程

1. 先确定需要验证的原子主张、模型版本、单位和适用范围。
2. 项目相关问题先读取实际生产配置与实现版本，而不是只相信 README。
3. 用 ADS/SciX、DOI 或 arXiv 确认论文身份和版本。
4. 优先搜索 arXiv TeX 源码，定位公式、单位、表格、脚注和引用键。
5. 在同一版本 PDF 中复核最终排版内容和上下文。
6. 沿引用链追踪继承参数，检查当前实现中的开关、默认值和单位转换。
7. 必要时进行独立交叉检查或小规模复算，并明确标成复现结果而非论文原话。

## 证据交付

对于可访问 PDF 中的决定性段落，最终回答必须直接显示原文裁剪，而不是只提供论文链接。证据卡包含：

- 带淡紫色标注的 PDF 原文裁剪；
- 论文标识、版本、页码、章节、公式、表格或脚注位置；
- 从有效 TeX 源码核对过的短原文；
- `supports`：该证据直接支持什么；
- `does not establish`：该证据不能单独证明什么；
- PDF、源码、原始裁剪和标注裁剪的哈希及渲染参数。

默认标注样式为：

- 填充色：`#C4B5FD`；
- 填充透明度：`0.28`；
- 边框色：`#8B5CF6`；
- 边框宽度：4 像素。

脚本始终保留未标注的 `.raw.png`，并分别记录原图和标注图哈希。高亮只是展示层，不会冒充论文原始内容。

## 输出示例

下面是对 Strong et al. (2010) 表 1 脚注 b 的一张完整证据卡。图片由本仓库脚本直接从同版本 PDF 裁剪并标注，没有手工重写论文文字。

![Strong et al. 2010 表 1 脚注 b 的淡紫色高亮证据](assets/example-strong2010-page4.png)

- **结论**：该模型把径向源密度在 10–15 kpc 范围内固定为 10 kpc 处的值，并在 15 kpc 之外设为零。
- **原文定位**：[Strong et al. (2010), arXiv:1008.4330v1](https://arxiv.org/pdf/1008.4330v1)，PDF 第 4 页，表 1 脚注 b。
- **Supports**：论文明确规定了这段径向修正及其两个边界。
- **Does not establish**：这张图不能单独证明某个当前项目或新版程序实际启用了该源分布；还需要配置和实现代码证据。
- **可审计附件**：[未标注裁剪](assets/example-strong2010-page4.raw.png) · [JSON 证据清单](assets/example-strong2010-page4.json)

示例只保留核查所需的一行原文。JSON 清单记录 PDF 哈希、页码、裁剪坐标、淡紫色标注参数以及原图和标注图的独立哈希。

## 安装

需要 Python 3.9 或更新版本。无头 PDF 渲染需要 Poppler 提供的 `pdfinfo` 和 `pdftoppm`；添加高亮需要当前 Python 环境安装 Pillow。

克隆仓库：

```bash
git clone https://github.com/SOYONAOC/paper-claim-audit.git
cd paper-claim-audit
```

复制到个人 Codex skill 目录：

```bash
skill_target="${CODEX_HOME:-$HOME/.codex}/skills/paper-claim-audit"
mkdir -p "$skill_target"
cp -a SKILL.md agents references scripts "$skill_target/"
```

Codex 通常会自动发现新增或更新的 skill；如果当前任务没有刷新，可重新启动 Codex 或新建任务。

## 无头服务器生成证据图

```bash
python scripts/render_pdf_evidence.py PAPER.pdf \
  --page 4 \
  --crop 120 640 1800 520 \
  --highlight 20 347 1685 36 \
  --output evidence/paper-page4-highlighted.png \
  --paper-id arXiv:0000.00000v1 \
  --source-url https://arxiv.org/pdf/0000.00000v1 \
  --retrieved-date 2026-01-01
```

`--crop` 坐标相对于完整 PDF 页面在指定 DPI 下的像素坐标；`--highlight` 坐标相对于最终裁剪图。可以重复传入 `--highlight` 标注多个不连续区域。

输出包括：

- 带标注的 PNG；
- 未标注的 `.raw.png`；
- 包含哈希、页码、裁剪和标注参数的 JSON 清单；
- 可直接用于 Codex 桌面端的绝对路径 Markdown 图片语法。

脚本遇到缺少 Poppler、缺少 Pillow、页码越界、标注越界或覆盖已有输出时会明确失败，不会静默换工具或退化成无标注结果。

## 证据边界

- 元数据服务只能确认论文身份，不能替代正文证据；
- arXiv 源码便于精确检索，但必须由同版本 PDF 确认其进入最终排版；
- PDF 原文能证明作者写了什么，不能单独证明当前代码确实执行该设置；
- 高亮框只指示核查重点，未标注裁剪才是视觉原件；
- 一篇论文可以确认采用或描述，不能自动证明物理结论唯一正确；
- 证据不足时应报告 `unresolved`，不能用记忆、搜索摘要或方便的数值补齐缺口。

## 仓库结构

```text
paper-claim-audit/
├── SKILL.md
├── agents/openai.yaml
├── assets/
│   ├── example-strong2010-page4.json
│   ├── example-strong2010-page4.png
│   └── example-strong2010-page4.raw.png
├── references/
│   ├── report-template.md
│   └── source-pdf-workflow.md
└── scripts/render_pdf_evidence.py
```

本仓库是独立发行版。该 skill 同时收录在 [SOYONAOC/codex-skills](https://github.com/SOYONAOC/codex-skills) 合集仓库中。
