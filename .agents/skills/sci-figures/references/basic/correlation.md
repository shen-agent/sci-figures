# 数据关系型图表

展示变量之间的关系和相关性。当用户需要散点图、气泡图、相关性热力图、Q-Q 图时阅读此文件。

## 包含的图表类型

1. [带趋势线的散点图](#带趋势线的散点图)
2. [多系列散点图](#多系列散点图)
3. [高密度散点图](#高密度散点图)
4. [气泡图](#气泡图)
5. [相关系数热力图](#相关系数热力图)
6. [Q-Q 图](#qq-图)
7. [二维核密度图](#二维核密度图)

## 通用前置

所有示例假定已加载：

```r
library(ggplot2)
library(ggsci)

npg_pal <- ggsci::pal_npg("nrc")(10)   # NPG 10 色调色板，详见 style-guide.md "配色方案"
# theme_insightlab() 定义见 style-guide.md "统一主题模板"
# 出图：PNG 用 ggsave(..., device = ragg::agg_png)；PDF 用 ggsave 默认（推断 .pdf）
```

## 带趋势线的散点图

适用：展示两个连续变量之间的关系趋势。

```r
p <- ggplot(df, aes(x = x, y = y)) +
  geom_point(size = 2, color = npg_pal[4], alpha = 0.7) +
  geom_smooth(method = "loess", span = 0.5, se = TRUE,
              color = npg_pal[1], fill = npg_pal[1],
              alpha = 0.2) +
  theme_insightlab() +
  labs(x = "Variable X", y = "Variable Y")
```

变体：
- 线性回归：`method = "lm"`
- 添加 R² 和 P 值：用 `ggpubr::stat_cor()`

```r
library(ggpubr)
p + stat_cor(method = "pearson", label.x.npc = 0.05, label.y.npc = 0.95)
```

## 多系列散点图

```r
p <- ggplot(df, aes(x = x, y = y, fill = group, shape = group)) +
  geom_point(size = 3, color = "black", stroke = 0.3, alpha = 0.8) +
  scale_fill_manual(values = npg_pal) +
  scale_shape_manual(values = c(21, 22, 24)) +
  stat_ellipse(aes(color = group), level = 0.95, linewidth = 0.5,
               show.legend = FALSE) +
  scale_color_manual(values = npg_pal) +
  theme_insightlab()
```

## 高密度散点图

适用：数据量大（>1000 点）时，避免过度绑制。

### 六角密度图

```r
p <- ggplot(df, aes(x = x, y = y)) +
  geom_hex(bins = 30) +
  scale_fill_distiller(palette = "Spectral", direction = -1) +
  theme_insightlab()
```

### 二维密度等高线 + 散点

```r
p <- ggplot(df, aes(x = x, y = y)) +
  geom_point(size = 0.5, alpha = 0.3, color = "grey40") +
  geom_density_2d(color = npg_pal[1], linewidth = 0.4) +
  theme_insightlab()
```

## 气泡图

适用：在散点图基础上用气泡大小编码第三个变量。

```r
p <- ggplot(df, aes(x = x, y = y, size = z, fill = z)) +
  geom_point(shape = 21, color = "black", stroke = 0.3, alpha = 0.8) +
  scale_size_area(max_size = 12) +
  scale_fill_gradient2(low = npg_pal[4], mid = "white",
                       high = npg_pal[1],
                       midpoint = mean(df$z)) +
  theme_insightlab() +
  labs(size = "Z Value", fill = "Z Value")
```

## 相关系数热力图

适用：展示多变量间的相关系数矩阵。

```r
library(reshape2)

cor_matrix <- round(cor(df_numeric, use = "complete.obs"), 2)
cor_melt <- melt(cor_matrix)

p <- ggplot(cor_melt, aes(x = Var1, y = Var2, fill = value)) +
  geom_tile(color = "white", linewidth = 0.5) +
  geom_text(aes(label = value), size = 3, color = "black") +
  scale_fill_gradient2(low = npg_pal[4], mid = "white",
                       high = npg_pal[1], midpoint = 0,
                       limits = c(-1, 1)) +
  coord_equal() +
  theme_insightlab() +
  theme(axis.text.x = element_text(angle = 45, hjust = 1)) +
  labs(x = NULL, y = NULL, fill = "Correlation")
```

### 气泡式相关图

```r
cor_melt$abs_value <- abs(cor_melt$value)

p <- ggplot(cor_melt, aes(x = Var1, y = Var2, fill = value, size = abs_value)) +
  geom_point(shape = 21, color = "black", stroke = 0.3) +
  scale_size_area(max_size = 10, guide = "none") +
  scale_fill_gradient2(low = npg_pal[4], mid = "white",
                       high = npg_pal[1], midpoint = 0) +
  coord_equal() +
  theme_insightlab() +
  theme(axis.text.x = element_text(angle = 45, hjust = 1))
```

## Q-Q 图

适用：检验数据是否服从正态分布。

```r
p <- ggplot(df, aes(sample = value)) +
  geom_qq(shape = 21, fill = npg_pal[1], color = "black",
          stroke = 0.3, size = 2) +
  geom_qq_line(linewidth = 0.5, color = "grey40") +
  theme_insightlab() +
  labs(x = "Theoretical Quantiles", y = "Sample Quantiles")
```

## 二维核密度图

### 填充型

```r
p <- ggplot(df, aes(x = x, y = y)) +
  stat_density_2d(aes(fill = after_stat(level)),
                  geom = "polygon", bins = 15) +
  scale_fill_distiller(palette = "Spectral", direction = -1) +
  theme_insightlab()
```

### 方块型

```r
p <- ggplot(df, aes(x = x, y = y)) +
  stat_density_2d(aes(fill = after_stat(density)),
                  geom = "tile", contour = FALSE) +
  scale_fill_distiller(palette = "Spectral", direction = -1) +
  theme_insightlab()
```

## 常见问题

- **散点太多看不清**：用 `geom_hex()` 或降低 `alpha`
- **相关系数热图对角线多余**：绑图前用 `cor_matrix[lower.tri(cor_matrix)] <- NA`
- **趋势线置信区间太宽**：减小 `span`（loess）或检查数据是否有极端值
- **气泡大小映射不直观**：用 `scale_size_area()` 确保面积与数值成比例
