# 出图规范

统一的 publication-ready 出图规范。所有 sci-figures 生成的图表都应遵循本规范，确保风格一致且满足期刊投稿要求。

## 字体规范

| 场景                 | 首选字体                        | 备选字体             | 说明                            |
| -------------------- | ------------------------------- | -------------------- | ------------------------------- |
| 英文正文/轴标签      | Arial                           | Helvetica            | 无衬线，Nature/Cell/Lancet 通用 |
| 英文等宽（数值注释） | Courier New                     | Monaco               | 等宽对齐                        |
| 中文标签             | Source Han Sans（思源黑体）     | Microsoft YaHei      | 无衬线，与 Arial 风格统一       |
| 字号 `base_size`   | 12 pt（默认 presentation 预设） | 7 pt（期刊单栏投稿） | 详见下方"字号与尺寸规范"        |

### R 字体处理

InsightLab 使用 `ragg` 作为图形设备——**ragg 通过 FreeType 自动识别系统字体**，不需要 `font_add()` 等任何字体注册代码：

```r
# 在 ggsave 中指定 device，theme 中通过 base_family 指定字体名
ggplot(...) +
  theme_insightlab(base_family = "Arial")          # 英文（默认）
  # theme_insightlab(base_family = "Source Han Sans")  # 中文

ggsave("fig.png", plot = p, device = ragg::agg_png, dpi = 300, bg = "white")
```

中文渲染要求**系统层面**预装思源黑体（Source Han Sans）或 Noto Sans CJK——R 中无需 `font_add()` / `font_add_google()`。如果系统未装相应字体，ragg 会回退到默认 sans-serif。

**禁止使用 showtext**：showtext 默认 dpi=96 与 ggsave 默认 dpi=300 不一致会导致字体按 1/3 缩放渲染，且在 Windows 常报"Arial not found" warning——这些问题 ragg 都没有。

## 配色方案

### 默认：ggsci NPG（Nature Publishing Group 风格）

InsightLab 默认使用 `ggsci` 包的 NPG 配色——对应 Nature 系期刊视觉风格，已通过 Coblis 色盲模拟器验证（Deuteranopia / Protanopia / Tritanopia）。

```r
library(ggsci)

p <- ggplot(data, aes(x, y, color = group, fill = group)) +
  geom_point() +
  scale_color_npg() +     # 离散：点 / 线 / 边
  scale_fill_npg()        # 填充：柱 / 面 / 箱
```

NPG 10 色调色板（等同 `ggsci::pal_npg("nrc")(10)`）：

| 序号 | 色值        | 色名   | 典型用途           |
| ---- | ----------- | ------ | ------------------ |
| 1    | `#E64B35` | 砖红   | 实验组、上调、阳性 |
| 2    | `#4DBBD5` | 青蓝   | 对照组、下调、阴性 |
| 3    | `#00A087` | 墨绿   | 第三组             |
| 4    | `#3C5488` | 深蓝   | 第四组             |
| 5    | `#F39B7F` | 珊瑚橙 | 辅助色             |
| 6    | `#8491B4` | 灰蓝   | 辅助色             |
| 7    | `#91D1C2` | 薄荷绿 | 辅助色             |
| 8    | `#DC0000` | 鲜红   | 辅助色             |
| 9    | `#7E6148` | 棕色   | 辅助色             |
| 10   | `#B09C85` | 卡其   | 辅助色             |

需要透明度：`scale_color_npg(alpha = 0.7)`。

### 期刊投稿快速切换

按目标期刊一行代码切换（其它代码保持不变）：

```r
scale_color_npg()      # Nature / Springer Nature（默认）
scale_color_aaas()     # Science / AAAS
scale_color_lancet()   # Lancet
scale_color_jama()     # JAMA
scale_color_nejm()     # NEJM
scale_color_d3()       # Cell（D3.js 风格，Cell Press 常用）
scale_color_jco()      # Journal of Clinical Oncology
```

`scale_fill_*()` 同名一一对应。

### 渐变色方案（热图 / diverging / 连续数据）

#### 双向渐变（diverging）速查表

按场景选用，**默认蓝-白-红**（与 NPG 主调色板视觉协调，符合 Nature/Cell 投稿审美）：

| 方案                         | Low（-2）                    | Mid（0）                   | High（2）                    | 视觉效果                   | 适用场景                                          |
| ---------------------------- | ---------------------------- | -------------------------- | ---------------------------- | -------------------------- | ------------------------------------------------- |
| **默认（蓝-白-红）** | `#3C5488` 深蓝             | `#FFFFFF` 白             | `#E64B35` 砖红             | 蓝 → 白 → 红，对比鲜明   | Nature/Cell 投稿首选，色盲安全                    |
| NPG                          | `#E64B3599` 砖红（半透明） | `#FFFFFFCC` 白（半透明） | `#4DBBD599` 青蓝（半透明） | 红 → 白 → 青（反转方向） | Nature Publishing Group 风格，与 ggsci 主调色一致 |
| RdYlBu                       | `#D53E4F` 红               | `#FFFFBF` 淡黄           | `#3288BD` 蓝               | 红 → 黄 → 蓝             | ColorBrewer 经典热图，通用                        |
| Viridis                      | `#440154` 深紫             | `#21908C` 青绿           | `#FDE725` 亮黄             | 紫 → 绿 → 黄，感知均匀   | 需要精确感知数值差异，色盲友好                    |
| Blue-Yellow                  | `#0000FF` 蓝               | `#FFFFEE` 淡黄           | `#FFD700` 金黄             | 蓝 → 淡黄 → 金，对比鲜明 | 基因组学、甲基化热图                              |

> **ComplexHeatmap 注意**：ComplexHeatmap 的 `col` 参数**不接受** ggplot2 的 `scale_fill_gradient2()` / `scale_fill_distiller()`，必须用 `circlize` 包的 `colorRamp2()` 构造色阶函数：
>
> ```r
> library(circlize)
> col_fun <- colorRamp2(c(-2, 0, 2), c("#3C5488", "#FFFFFF", "#E64B35"))  # InsightLab 默认
> ComplexHeatmap::Heatmap(mat, col = col_fun)
> ```
>
> `colorRamp2(breaks, colors)` 的 `breaks` 与 `colors` 长度必须相等，且 `breaks` 单调递增。

## 字号与尺寸规范

InsightLab 默认采用方案 D，适合屏幕预览/PPT 嵌入/迭代展示。期刊投稿时把 `base_size` 改为 7 + ggsave 尺寸切换为对应期刊单栏 mm 即可。

### 画布

| 参数 | 值                    |
| ---- | --------------------- |
| 宽度 | 7 英寸（178 mm）      |
| 高度 | 4 英寸（102 mm）      |
| DPI  | 300                   |
| 像素 | 2100 × 1200 px       |
| 背景 | 白色 `bg = "white"` |

### 字体

| 元素                      | 字体  | 字号                        | 样式       | 颜色        |
| ------------------------- | ----- | --------------------------- | ---------- | ----------- |
| 全局基准                  | Arial | 12 pt（`base_size = 12`） | —         | 黑色        |
| 标题 `plot.title`       | Arial | 14.4 pt（`rel(1.2)`）     | 加粗，居中 | 黑色        |
| 副标题 `plot.subtitle`  | Arial | 10.8 pt（`rel(0.9)`）     | 常规，居中 | 灰色 grey40 |
| 坐标轴标题 `axis.title` | Arial | 13.2 pt（`rel(1.1)`）     | 常规       | 黑色        |
| 坐标轴刻度 `axis.text`  | Arial | 12 pt（`rel(1)`）         | 常规       | 黑色        |
| 图例标题 `legend.title` | Arial | 12 pt（`rel(1)`）         | 加粗       | 黑色        |
| 图例文字 `legend.text`  | Arial | 10.8 pt（`rel(0.9)`）     | 常规       | 黑色        |
| 分面标签 `strip.text`   | Arial | 12 pt（`rel(1)`）         | 加粗       | 黑色        |
| 图内注释 `annotate`     | Arial | `size = 3.6`（≈ 10 pt）  | 斜体       | 灰色 grey30 |

> `geom_text` / `geom_label` / `annotate` 的 `size=` 单位是 **mm 不是 pt**。换算：`size = pt 数值 / ggplot2::.pt`（`.pt ≈ 2.845`）。表中"图内注释 size = 3.6 ≈ 10 pt"就是这么算出来的。

### 间距

| 元素                     | margin                                           |
| ------------------------ | ------------------------------------------------ |
| 标题下方                 | 8 pt                                             |
| 副标题上方               | 2 pt                                             |
| 副标题下方               | 10 pt                                            |
| 画布边距 `plot.margin` | 上 5、右 40、下 5、左 5 pt（右侧留给参考线标签） |

### 线条

| 元素                      | 粗细   | 颜色   |
| ------------------------- | ------ | ------ |
| 坐标轴线 `axis.line`    | 0.3 pt | 黑色   |
| 坐标轴刻度 `axis.ticks` | 0.3 pt | 黑色   |
| 柱形边框                  | 0.2 pt | 黑色   |
| 参考虚线                  | 0.4 pt | grey40 |

### 导出模板

```r
# PNG（屏幕/PPT 默认）— 用 ragg::agg_png 设备
ggsave("output.png", p, width = 7, height = 4,
       device = ragg::agg_png, dpi = 300, bg = "white")

# PDF（期刊投稿/矢量首选）— ggsave 默认 device 推断
ggsave("output.pdf", p, width = 7, height = 4)
```

ragg 自动识别系统字体且 dpi 处理正确——零字体配置代码。

## 统一主题模板

```r
# 默认面向 7×4 英寸屏幕/PPT/迭代场景；期刊投稿用 theme_insightlab(base_size = 7)
theme_insightlab <- function(base_size = 12, base_family = "Arial") {
  theme_classic(base_size = base_size, base_family = base_family) %+replace%
    theme(
      # 轴线（细黑线 0.3pt）
      axis.line  = element_line(color = "black", linewidth = 0.3),
      axis.ticks = element_line(color = "black", linewidth = 0.3),
      axis.text  = element_text(color = "black", size = rel(1)),
      axis.title = element_text(color = "black", size = rel(1.1)),
      # 图例（透明背景）
      legend.background = element_blank(),
      legend.key        = element_blank(),
      legend.text  = element_text(size = rel(0.9)),
      legend.title = element_text(size = rel(1), face = "bold"),
      # 分面（无边框，粗体标签）
      strip.background = element_blank(),
      strip.text       = element_text(face = "bold", size = rel(1)),
      # 标题（居中粗体；下方留 8pt）
      plot.title    = element_text(hjust = 0.5, face = "bold", size = rel(1.2),
                                   margin = margin(b = 8)),
      plot.subtitle = element_text(hjust = 0.5, size = rel(0.9), color = "grey40",
                                   margin = margin(t = 2, b = 10)),
      # 边距（右侧 40pt 留给参考线/图外标签）
      plot.margin = margin(5, 40, 5, 5, "pt")
    )
}
```

### 使用方式

```r
library(ggplot2)

library(ggsci)
p <- ggplot(data, aes(x, y, fill = group)) +
  geom_boxplot() +
  scale_fill_npg() +                          # 默认 NPG 配色（Nature 风格）
  theme_insightlab() +                        # 默认 base_size = 12
  labs(x = "Group", y = "Expression")

# 默认尺寸：7×4 英寸 PNG（屏幕/PPT）
ggsave("figure.png", p, width = 7, height = 4,
       device = ragg::agg_png, dpi = 300, bg = "white")

# 期刊投稿（如 Nature 单栏）：传 base_size = 7 + mm 尺寸 + PDF
# p <- p + theme_insightlab(base_size = 7)
# ggsave("figure.pdf", p, width = 89, height = 60, units = "mm")
```

## 通用代码模板

每个绑图脚本建议遵循以下结构：

```r
# 1. 加载包
library(ggplot2)
library(ggsci)
# 注：不需要 showtext —— ragg 自动识别系统字体

# 2. 主题定义（或 source 共享文件）
# theme_insightlab() 定义...

# 3. 读取数据
data <- read.csv("input.csv")

# 4. 绑图
p <- ggplot(data, aes(...)) +
  geom_xxx() +
  scale_color_npg() +            # 默认 NPG 配色
  scale_fill_npg() +
  theme_insightlab() +           # 默认 base_size = 12
  labs(...)

# 5. 导出（默认 7×4 英寸，PPT/屏幕场景）
ggsave("output.png", p, width = 7, height = 4,
       device = ragg::agg_png, dpi = 300, bg = "white")    # PNG 用 ragg
ggsave("output.pdf", p, width = 7, height = 4, bg = "white")  # PDF 用默认
```

## 按预设自动切换的代码模板

当需要根据 `target_journal` 参数动态选择尺寸 / `base_size` 时，使用预设表统一管理：

```r
# 输出预设表（presentation 走 PNG/ragg；其余期刊走 PDF/默认）
journal_specs <- list(
  presentation = list(width = 7,  height = 4,  units = "in", base_size = 12,
                      file = "fig.png", device = ragg::agg_png),
  nature       = list(width = 89, height = 60, units = "mm", base_size = 7,
                      file = "fig.pdf", device = NULL),
  cell         = list(width = 85, height = 60, units = "mm", base_size = 7,
                      file = "fig.pdf", device = NULL),
  lancet       = list(width = 89, height = 60, units = "mm", base_size = 7,
                      file = "fig.pdf", device = NULL),
  jama         = list(width = 86, height = 60, units = "mm", base_size = 7,
                      file = "fig.pdf", device = NULL),
  pnas         = list(width = 87, height = 60, units = "mm", base_size = 7,
                      file = "fig.pdf", device = NULL)
)
spec <- journal_specs[[target_journal]]   # target_journal 由 agent 注入

# 不需要 showtext —— ragg 自动识别系统字体

p <- ggplot(...) +
  ...  +
  theme_insightlab(base_size = spec$base_size)

ggsave(spec$file, plot = p,
       width  = spec$width,
       height = spec$height,
       units  = spec$units,
       device = spec$device,    # presentation → ragg::agg_png；期刊 → NULL（按扩展名推断 PDF）
       dpi    = 300,
       bg     = "white")
```
