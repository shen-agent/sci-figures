# 多面板组合图

当需要将多个子图拼接为一张完整的 figure 时阅读此文件。适用于论文中的 Figure 1（多面板 A/B/C/D）、补充图、以及需要统一排版的组合图。

## 包含的图表类型

1. 等尺寸网格拼图
2. 不等尺寸自定义布局
3. 面板标签（A/B/C/D）
4. ggplot2 + 非 ggplot2 对象混合拼图
5. 导出与尺寸控制

## 核心工具：patchwork

patchwork 是 ggplot2 生态中最推荐的拼图方案，语法简洁，支持任意布局。

```r
library(patchwork)
```

## 1. 等尺寸网格拼图

### 水平并排

```r
p1 + p2
```

### 垂直堆叠

```r
p1 / p2
```

### 2x2 网格

```r
(p1 + p2) / (p3 + p4)
```

### 完整示例

```r
library(ggplot2)
library(patchwork)

npg_pal <- ggsci::pal_npg("nrc")(10)

p1 <- ggplot(mtcars, aes(wt, mpg)) +
  geom_point(color = npg_pal[1]) +
  theme_insightlab()

p2 <- ggplot(mtcars, aes(factor(cyl), mpg, fill = factor(cyl))) +
  geom_boxplot() +
  scale_fill_manual(values = npg_pal[1:3]) +
  theme_insightlab()

p3 <- ggplot(mtcars, aes(hp, mpg)) +
  geom_point(color = npg_pal[2]) +
  geom_smooth(method = "lm", color = npg_pal[4]) +
  theme_insightlab()

p4 <- ggplot(mtcars, aes(factor(gear), fill = factor(gear))) +
  geom_bar() +
  scale_fill_manual(values = npg_pal[1:3]) +
  theme_insightlab()

combined <- (p1 + p2) / (p3 + p4)
combined
```

## 2. 不等尺寸自定义布局

### 使用 plot_layout() 控制比例

```r
# 第一行占 2/3 高度，第二行占 1/3
(p1 + p2) / p3 +
  plot_layout(heights = c(2, 1))

# 左边占 2/3 宽度，右边占 1/3
p1 + p2 +
  plot_layout(widths = c(2, 1))
```

### 使用 design 字符串实现复杂布局

```r
design <- "
  AABB
  CCDD
  CCEE
"
p1 + p2 + p3 + p4 + p5 +
  plot_layout(design = design)
```

### 空白占位

```r
# 用 plot_spacer() 留空
p1 + plot_spacer() + p2
```

## 3. 面板标签（A/B/C/D）

```r
combined <- (p1 + p2) / (p3 + p4) +
  plot_annotation(
    tag_levels = "A",
    tag_prefix = "",
    tag_suffix = "",
    theme = theme(
      plot.tag = element_text(size = 12, face = "bold", family = "Arial")
    )
  )
```

### 自定义标签样式

```r
# 使用小写字母
plot_annotation(tag_levels = "a")

# 使用罗马数字
plot_annotation(tag_levels = "I")

# 手动指定标签
plot_annotation(tag_levels = list(c("A", "B", "C", "D")))
```

### 添加总标题

```r
combined + plot_annotation(
  title = "Figure 1. Multi-panel overview",
  subtitle = "Data from XYZ study",
  tag_levels = "A"
)
```

## 4. 混合拼图（ggplot2 + 非 ggplot2 对象）

ComplexHeatmap、pheatmap 等非 ggplot2 对象不能直接用 `+` 拼接。用 `wrap_elements()` 桥接。

### ggplot2 + ComplexHeatmap

```r
library(ComplexHeatmap)
library(grid)

ht <- Heatmap(matrix_data, name = "Expression")

p_heatmap <- wrap_elements(full = grid.grabExpr(draw(ht)))

p1 + p_heatmap +
  plot_layout(widths = c(1, 2)) +
  plot_annotation(tag_levels = "A")
```

### ggplot2 + pheatmap

```r
library(pheatmap)

ph <- pheatmap(matrix_data, silent = TRUE)
p_ph <- wrap_elements(full = ph$gtable)

p1 + p_ph
```

### ggplot2 + base R 图

```r
base_plot <- wrap_elements(full = ~ {
  plot(1:10, main = "Base R Plot")
})

p1 + base_plot
```

## 5. 导出与尺寸控制

### 单栏 figure（Nature: 89mm 宽）

```r
ggsave("figure1.pdf", combined,
       width = 89, height = 120, units = "mm")
```

### 双栏 figure（Nature: 183mm 宽）

```r
ggsave("figure1.pdf", combined,
       width = 183, height = 150, units = "mm")
```

### 高分辨率 TIFF

```r
ggsave("figure1.tiff", combined,
       width = 183, height = 150, units = "mm",
       dpi = 300, compression = "lzw")
```

### 尺寸计算技巧

面板数量与尺寸的经验公式：
- 每个子图建议最小 40mm x 40mm
- 2x2 布局 → 至少 183mm x 130mm（双栏）
- 1x3 布局 → 至少 183mm x 70mm（双栏）
- 3x2 布局 → 至少 183mm x 180mm（双栏）

## 常见问题

- **子图之间间距太大**：`plot_layout(guides = "collect")` 合并图例，或用 `theme(plot.margin = margin(2, 2, 2, 2, "pt"))` 减小边距
- **标签与图重叠**：增加 `plot.tag.position` 或调整 `plot.margin`
- **图例重复**：`plot_layout(guides = "collect") & theme(legend.position = "bottom")` 统一图例位置
- **字体大小不一致**：确保所有子图使用相同的 `theme_insightlab(base_size = 12)`（默认）或统一指定其它值
