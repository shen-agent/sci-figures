# 类别比较型图表

用于不同类别之间的数值对比。当用户需要柱形图、条形图、棒棒糖图、哑铃图、坡度图时阅读此文件。

## 包含的图表类型

1. [单系列柱形图](#单系列柱形图)
2. [多系列柱形图](#多系列柱形图)
3. [堆积柱形图](#堆积柱形图)
4. [百分比堆积柱形图](#百分比堆积柱形图)
5. [水平条形图](#水平条形图)
6. [棒棒糖图](#棒棒糖图)
7. [哑铃图](#哑铃图)
8. [坡度图](#坡度图)

## 通用前置

所有示例假定已加载：

```r
library(ggplot2)
library(ggsci)

npg_pal <- ggsci::pal_npg("nrc")(10)   # NPG 10 色调色板，详见 style-guide.md "配色方案"
# theme_insightlab() 定义见 style-guide.md "统一主题模板"
# 出图：PNG 用 ggsave(..., device = ragg::agg_png)；PDF 用 ggsave 默认（推断 .pdf）
```

## 单系列柱形图

适用：单个变量在不同类别间的比较。

```r
library(ggplot2)

p <- ggplot(df, aes(x = reorder(category, -value), y = value)) +
  geom_col(fill = npg_pal[1], color = "black", width = 0.7, linewidth = 0.3) +
  theme_insightlab() +
  labs(x = NULL, y = "Value")
```

关键参数：
- `reorder(category, -value)`：按值降序排列
- `width`：柱宽，0.6-0.8 为宜

输入格式：`category`（字符）、`value`（数值）两列。

## 多系列柱形图

适用：多组数据在同一类别下的并列比较。

```r
p <- ggplot(df_long, aes(x = category, y = value, fill = group)) +
  geom_col(position = position_dodge(0.8), width = 0.7,
           color = "black", linewidth = 0.25) +
  scale_fill_manual(values = npg_pal) +
  theme_insightlab() +
  labs(x = NULL, y = "Value", fill = "Group")
```

输入格式：长格式（`category`、`group`、`value`），用 `tidyr::pivot_longer()` 转换宽格式。

## 堆积柱形图

适用：展示各组成部分对总量的贡献。

```r
p <- ggplot(df_long, aes(x = category, y = value, fill = component)) +
  geom_col(position = "stack", color = "black", linewidth = 0.25, width = 0.7) +
  scale_fill_manual(values = npg_pal) +
  theme_insightlab()
```

## 百分比堆积柱形图

适用：展示各组成部分的比例构成。

```r
p <- ggplot(df_long, aes(x = category, y = value, fill = component)) +
  geom_col(position = "fill", color = "black", linewidth = 0.25, width = 0.7) +
  scale_y_continuous(labels = scales::percent) +
  scale_fill_manual(values = npg_pal) +
  theme_insightlab() +
  labs(y = "Proportion")
```

## 水平条形图

适用：类别名称较长时，水平展示更可读。

```r
p <- ggplot(df, aes(x = reorder(category, value), y = value)) +
  geom_col(fill = npg_pal[1], color = "black", width = 0.6, linewidth = 0.25) +
  coord_flip() +
  theme_insightlab() +
  labs(x = NULL, y = "Value")
```

## 棒棒糖图

适用：替代柱形图的轻量表达，视觉更清爽。

```r
p <- ggplot(df, aes(x = reorder(category, value), y = value)) +
  geom_segment(aes(xend = category, y = 0, yend = value),
               color = "grey60", linewidth = 0.5) +
  geom_point(size = 3, color = npg_pal[1]) +
  coord_flip() +
  theme_insightlab() +
  labs(x = NULL, y = "Value")
```

## 哑铃图

适用：两个时间点或两组之间的对比，直观显示变化方向和幅度。

```r
p <- ggplot(df) +
  geom_segment(aes(x = value_before, xend = value_after,
                   y = reorder(category, value_after), yend = category),
               color = "grey60", linewidth = 0.5) +
  geom_point(aes(x = value_before, y = category), size = 3,
             color = npg_pal[2]) +
  geom_point(aes(x = value_after, y = category), size = 3,
             color = npg_pal[1]) +
  theme_insightlab() +
  labs(x = "Value", y = NULL)
```

输入格式：宽格式，含 `category`、`value_before`、`value_after` 三列。

## 坡度图

适用：两个时间点之间多个类别的变化趋势对比。

```r
library(ggrepel)

p <- ggplot(df) +
  geom_segment(aes(x = 1, xend = 2, y = year1, yend = year2,
                   color = ifelse(year2 > year1, "up", "down")),
               linewidth = 0.8, show.legend = FALSE) +
  geom_point(aes(x = 1, y = year1), size = 3, color = "grey40") +
  geom_point(aes(x = 2, y = year2), size = 3, color = "grey40") +
  geom_text_repel(aes(x = 0.95, y = year1, label = paste(name, year1)),
                  hjust = 1, size = 3) +
  geom_text_repel(aes(x = 2.05, y = year2, label = paste(name, year2)),
                  hjust = 0, size = 3) +
  scale_color_manual(values = c(up = "#00A087", down = "#E64B35")) +
  scale_x_continuous(limits = c(0.5, 2.5), breaks = 1:2,
                     labels = c("2020", "2024")) +
  theme_void() +
  theme(axis.text.x = element_text(size = 10, face = "bold"))
```

输入格式：`name`、`year1`、`year2` 三列。

## 常见问题

- **柱子排序不对**：用 `reorder()` 或提前设置 `factor(levels = ...)`
- **堆积顺序反了**：调整 `fill` 变量的 factor levels 顺序
- **柱子太窄/太宽**：调整 `width` 参数（0.5-0.9）
- **多系列柱子没对齐**：确保 `position_dodge()` 的宽度与 `width` 一致
