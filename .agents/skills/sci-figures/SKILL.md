---
name: sci-figures
description: >
  使用 R + ggplot2 生成 publication-ready 的科研图表。
  当用户说"画个火山图"、"做一张热图"、"画生存曲线"、"出一张 PCA 图"、
  "帮我画个箱线图"、"做 publication-ready 的图"、"科研绑图"时触发。
  也适用于："draw a volcano plot"、"bindot chart"、"bindot figure"、
  柱形图、条形图、小提琴图、散点图、气泡图、相关性热图、
  OncoPrint、Lollipop plot、森林图、ROC 曲线、Kaplan-Meier 曲线、
  泳道图、韦恩图、KEGG/GO 富集气泡图、曼哈顿图、
  降维可视化、patchwork 拼图、以及任何需要生成科研级别图表的场景。
---
# 科研绑图 (sci-figures)

使用 R + ggplot2 生态生成 publication-ready 的科研图表。覆盖从基础统计图到生信个性化分析图的完整场景，统一遵循 Nature/Cell 等期刊的投稿规范。

## 前置条件

- R 3.6+
- 核心包：ggplot2、ggpubr、ggsci、ragg、patchwork、scales
- 生信包（按需）：ComplexHeatmap、survminer、forestplot、pheatmap、maftools、VennDiagram、clusterProfiler、swimplot
- 验证：`Rscript -e "library(ggplot2); cat(packageVersion('ggplot2'), '\n')"`

## 出图规范

**所有图表统一应用 InsightLab 视觉规范，完整说明见 `references/style-guide.md`**——生成代码前必须查阅。涵盖：

- 字体（Arial / 思源黑体；ragg 自动识别系统字体，无需 font_add）
- 配色（默认 `scale_color_npg()` / `scale_fill_npg()`；ggsci 一行切换 aaas / lancet / jama / nejm / d3 / jco；diverging 5 套预设 + ComplexHeatmap colorRamp2 注意事项）
- 字号与尺寸规范（方案 D 双栏紧凑：7×4 英寸、base_size=12、画布/字体/间距/线条 4 张速查表）
- `theme_insightlab()` 函数定义与使用方式
- `geom_text(size=)` 单位是 mm 不是 pt（要除以 `ggplot2::.pt ≈ 2.845`）
- 按 `target_journal` 自动切换的代码模板（presentation 默认 + nature/cell/lancet/jama/pnas 期刊投稿覆盖）

## 图表分类

### A. 基础通用图表

按数据表达目的分为 5 类：

**类别比较**（柱形图、条形图、棒棒糖图、哑铃图、坡度图、不等宽柱形图）：
→ `references/basic/comparison.md`

**数据分布**（箱线图、小提琴图、密度图、抖动图、蜂群图、直方图、峰峦图、带误差线图）：
→ `references/basic/distribution.md`

**数据关系**（散点图、气泡图、相关性热图、Q-Q 图、高密度散点图）：
→ `references/basic/correlation.md`

**时间序列**（折线图、面积图、堆积面积图、百分比堆积面积图）：
→ `references/basic/time-series.md`

**局部整体**（饼图、华夫饼图、马赛克图、百分比堆积图）：
→ `references/basic/proportion.md`

### B. 生信个性化图表

按分析场景分为 5 类：

**差异表达与富集**（火山图、表达热图、MA plot、KEGG/GO 富集气泡图）：
→ `references/bioinformatics/expression.md`

**突变景观**（OncoPrint、Lollipop plot、Rainfall plot、TiTv 图、曼哈顿图、基因互作图）：
→ `references/bioinformatics/mutation.md`

**生存分析**（Kaplan-Meier 曲线、森林图、批量 Cox 可视化）：
→ `references/bioinformatics/survival.md`

**降维可视化**（PCA、t-SNE、UMAP、分面散点图）：
→ `references/bioinformatics/dimensional-reduction.md`

**临床图表**（泳道图、瀑布图、ROC 曲线、韦恩图）：
→ `references/bioinformatics/clinical.md`

### C. 多面板组合图

**patchwork 拼图、面板标签（A/B/C/D）、不等尺寸布局、ggsave 导出**：
→ `references/composite.md`

## 工作流

1. 确认数据来源和格式（CSV、TSV、RDS、RData、Excel 等）
2. **询问用户**："你要画什么类型的图？有没有目标期刊的格式要求？"
3. 根据图表类型加载对应 reference
4. 应用 `theme_insightlab()` + `scale_color_npg()` / `scale_fill_npg()`（ggsci）生成 R 代码
5. 执行绑图并展示结果
6. **询问用户**："需要调整配色、字体、尺寸或其他细节吗？"
7. 迭代优化直到满意
8. 导出最终图表（PDF 优先 / PNG / SVG / TIFF）

## 故障排查

- **中文显示异常**：确认系统已装思源黑体或 Noto Sans CJK；ragg 会自动识别。在 theme 中显式指定 `base_family = "Source Han Sans"`
- **图表被裁切**：`ggsave(..., width = X, height = Y, units = "mm")`，确保尺寸充足
- **配色不理想**：默认 `scale_color_npg()`；按期刊切换 `scale_color_aaas()` / `scale_color_lancet()` / `scale_color_jama()` / `scale_color_nejm()` / `scale_color_d3()` / `scale_color_jco()`
- **ComplexHeatmap 与 ggplot2 冲突**：两者使用不同绘图系统，不能用 `+` 拼接，用 `patchwork::wrap_elements()` 桥接
- **大数据量绘图卡顿**：使用 `geom_hex()` 替代 `geom_point()`，或先对数据降采样
