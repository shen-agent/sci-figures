# 局部整体型图表

展示各部分在整体中的占比。当用户需要饼图、环形图、华夫饼图、马赛克图时阅读此文件。

## 包含的图表类型

1. [饼图](#饼图)
2. [环形图（Donut Chart）](#环形图)
3. [华夫饼图（Waffle Chart）](#华夫饼图)
4. [马赛克图（Mosaic Chart）](#马赛克图)

## 通用前置

所有示例假定已加载：

```r
library(ggplot2)
library(ggsci)

npg_pal <- ggsci::pal_npg("nrc")(10)   # NPG 10 色调色板，详见 style-guide.md "配色方案"
# theme_insightlab() 定义见 style-guide.md "统一主题模板"
# 出图：PNG 用 ggsave(..., device = ragg::agg_png)；PDF 用 ggsave 默认（推断 .pdf）
```

## 饼图

适用：展示少量类别（≤6）的占比。类别过多时建议改用条形图。

```r
p <- ggplot(df, aes(x = "", y = value, fill = category)) +
  geom_col(width = 1, color = "white", linewidth = 0.5) +
  coord_polar(theta = "y") +
  scale_fill_manual(values = npg_pal) +
  theme_void() +
  theme(legend.text = element_text(size = 8),
        legend.title = element_blank())
```

### 带标签的饼图

```r
library(ggrepel)

df$pos <- cumsum(df$value) - df$value / 2

p <- ggplot(df, aes(x = "", y = value, fill = category)) +
  geom_col(width = 1, color = "white", linewidth = 0.5) +
  coord_polar(theta = "y") +
  geom_text_repel(aes(y = pos,
                      label = paste0(category, "\n", round(value/sum(value)*100, 1), "%")),
                  size = 3, nudge_x = 0.5, show.legend = FALSE) +
  scale_fill_manual(values = npg_pal) +
  theme_void() +
  theme(legend.position = "none")
```

输入格式：`category`（字符）、`value`（数值，正数）。

## 环形图

适用：饼图的变体，中心留空，视觉上更现代。

```r
p <- ggplot(df, aes(x = 2, y = value, fill = category)) +
  geom_col(width = 1, color = "white", linewidth = 0.5) +
  coord_polar(theta = "y") +
  xlim(0.5, 2.5) +
  scale_fill_manual(values = npg_pal) +
  theme_void()
```

`xlim(0.5, 2.5)` 控制环的粗细，调小左边界增大中心空洞。

## 华夫饼图

适用：用方块或圆点阵列表示比例，替代饼图的直观表达。

```r
library(waffle)

# 数据：命名向量
parts <- c(A = 30, B = 25, C = 20, D = 15, E = 10)

p <- waffle(parts, rows = 5, size = 0.5,
            colors = npg_pal[1:5],
            legend_pos = "bottom") +
  theme(legend.text = element_text(size = 8))
```

### 手动实现（ggplot2）

```r
nrows <- 10
total <- nrows * nrows
df_counts <- data.frame(
  category = c("A", "B", "C", "D"),
  count = c(40, 30, 20, 10)
)

grid_df <- data.frame(
  x = rep(1:nrows, nrows),
  y = rep(1:nrows, each = nrows),
  category = rep(df_counts$category, df_counts$count)
)

p <- ggplot(grid_df, aes(x = x, y = y, fill = category)) +
  geom_tile(color = "white", linewidth = 0.5) +
  scale_fill_manual(values = npg_pal) +
  coord_equal() +
  theme_void() +
  theme(legend.position = "right")
```

## 马赛克图

适用：展示二维交叉分类的比例关系。矩形的宽度和高度分别编码两个维度的比例。

```r
# 数据准备：计算每个矩形的坐标
df_mosaic <- df %>%
  group_by(row_var) %>%
  mutate(
    row_total = sum(value),
    col_pct = value / row_total * 100
  ) %>%
  ungroup() %>%
  mutate(
    row_pct = row_total / sum(value) * 100
  )

# 计算矩形坐标
row_breaks <- cumsum(unique(df_mosaic$row_pct))
df_mosaic <- df_mosaic %>%
  group_by(row_var) %>%
  mutate(
    ymax = cumsum(col_pct),
    ymin = ymax - col_pct
  ) %>%
  ungroup()

# 添加 xmin/xmax
df_mosaic$xmax <- rep(row_breaks, each = n_distinct(df_mosaic$col_var))
df_mosaic$xmin <- df_mosaic$xmax - rep(unique(df_mosaic$row_pct),
                                        each = n_distinct(df_mosaic$col_var))

p <- ggplot(df_mosaic) +
  geom_rect(aes(xmin = xmin, xmax = xmax, ymin = ymin, ymax = ymax,
                fill = col_var), color = "black", linewidth = 0.3) +
  geom_text(aes(x = (xmin + xmax) / 2, y = (ymin + ymax) / 2,
                label = value), size = 3) +
  scale_fill_manual(values = npg_pal) +
  theme_insightlab() +
  labs(x = NULL, y = "Proportion (%)", fill = NULL)
```

输入格式：`row_var`（行分类）、`col_var`（列分类）、`value`（数值）。

## 常见问题

- **饼图类别太多（>6）**：改用水平条形图或百分比堆积柱形图
- **华夫饼方块总数不是 100**：调整 `count` 使总和为 100（或 nrows²）
- **环形图中心文字**：用 `annotate("text", x = 0.5, y = 0, label = "Total\n100", size = 5)` 添加
- **马赛克图坐标计算复杂**：使用 `ggmosaic` 包简化：`geom_mosaic(aes(weight = value, x = product(col_var, row_var), fill = col_var))`
