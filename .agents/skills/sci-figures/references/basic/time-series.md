# 时间序列型图表

展示数据随时间的变化趋势。当用户需要折线图、面积图、堆积面积图时阅读此文件。

## 包含的图表类型

1. [折线图](#折线图)
2. [面积图（纯色/渐变填充）](#面积图)
3. [堆积面积图](#堆积面积图)
4. [百分比堆积面积图](#百分比堆积面积图)
5. [夹层填充面积图](#夹层填充面积图)

## 通用前置

所有示例假定已加载：

```r
library(ggplot2)
library(ggsci)

npg_pal <- ggsci::pal_npg("nrc")(10)   # NPG 10 色调色板，详见 style-guide.md "配色方案"
# theme_insightlab() 定义见 style-guide.md "统一主题模板"
# 出图：PNG 用 ggsave(..., device = ragg::agg_png)；PDF 用 ggsave 默认（推断 .pdf）
```

## 折线图

### 单系列

```r
p <- ggplot(df, aes(x = date, y = value)) +
  geom_line(color = npg_pal[1], linewidth = 0.8) +
  scale_x_date(date_labels = "%Y", date_breaks = "1 year") +
  theme_insightlab() +
  labs(x = "Year", y = "Value")
```

### 多系列

```r
p <- ggplot(df_long, aes(x = date, y = value, color = group)) +
  geom_line(linewidth = 0.7) +
  scale_color_manual(values = npg_pal) +
  scale_x_date(date_labels = "%Y", date_breaks = "2 year") +
  theme_insightlab() +
  labs(x = "Year", y = "Value", color = "Group")
```

输入格式：长格式，含 `date`（Date 类型）、`value`（数值）、`group`（分组，多系列时需要）。

日期转换：
```r
df$date <- as.Date(df$date, format = "%Y/%m/%d")
```

## 面积图

### 纯色填充

```r
p <- ggplot(df, aes(x = date, y = value)) +
  geom_area(fill = npg_pal[1], alpha = 0.6, color = "black",
            linewidth = 0.5) +
  scale_x_date(date_labels = "%Y", date_breaks = "2 year") +
  theme_insightlab()
```

### 渐变填充

使用 `geom_bar` 逐条着色模拟渐变效果：

```r
p <- ggplot(df, aes(x = date, y = value, fill = value)) +
  geom_bar(stat = "identity", width = 2) +
  geom_line(color = "black", linewidth = 0.5) +
  scale_fill_distiller(palette = "Reds", direction = 1) +
  scale_x_date(date_labels = "%Y", date_breaks = "2 year") +
  theme_insightlab()
```

## 堆积面积图

适用：展示多个组成部分随时间的总量变化及各部分占比。

```r
p <- ggplot(df_long, aes(x = date, y = value, fill = component)) +
  geom_area(position = "stack", alpha = 0.9, color = "black",
            linewidth = 0.25) +
  scale_fill_manual(values = npg_pal) +
  scale_x_date(date_labels = "%Y", date_breaks = "2 year") +
  theme_insightlab() +
  labs(x = "Year", y = "Value", fill = "Component")
```

输入格式：长格式，`date`、`component`、`value`。堆积顺序由 `component` 的 factor levels 控制。

## 百分比堆积面积图

```r
p <- ggplot(df_long, aes(x = date, y = value, fill = component)) +
  geom_area(position = "fill", alpha = 0.9, color = "black",
            linewidth = 0.25) +
  scale_y_continuous(labels = scales::percent) +
  scale_fill_manual(values = npg_pal) +
  scale_x_date(date_labels = "%Y", date_breaks = "2 year") +
  theme_insightlab() +
  labs(x = "Year", y = "Proportion", fill = "Component")
```

## 夹层填充面积图

适用：展示两条曲线之间的差异区域。

### 单色填充

```r
p <- ggplot(df) +
  geom_ribbon(aes(x = date, ymin = series_a, ymax = series_b),
              fill = "grey80", alpha = 0.5) +
  geom_line(aes(x = date, y = series_a), color = npg_pal[1],
            linewidth = 0.7) +
  geom_line(aes(x = date, y = series_b), color = npg_pal[2],
            linewidth = 0.7) +
  scale_x_date(date_labels = "%Y", date_breaks = "2 year") +
  theme_insightlab()
```

### 双色填充（A > B 与 A < B 不同颜色）

```r
df$above <- ifelse(df$series_a > df$series_b, df$series_a, df$series_b)
df$color_group <- ifelse(df$series_a > df$series_b, "A > B", "B > A")

p <- ggplot(df) +
  geom_ribbon(aes(x = date,
                  ymin = pmin(series_a, series_b),
                  ymax = pmax(series_a, series_b),
                  fill = color_group), alpha = 0.4) +
  geom_line(aes(x = date, y = series_a), color = npg_pal[1],
            linewidth = 0.7) +
  geom_line(aes(x = date, y = series_b), color = npg_pal[2],
            linewidth = 0.7) +
  scale_fill_manual(values = c("A > B" = npg_pal[1],
                                "B > A" = npg_pal[2])) +
  theme_insightlab()
```

## 常见问题

- **日期轴显示不对**：确保 x 轴是 `Date` 类型，用 `as.Date()` 转换
- **堆积顺序不对**：调整 `fill` 变量的 factor levels
- **面积图有缝隙**：检查日期是否连续，有缺失日期会产生空白
- **图例顺序与堆积顺序不一致**：用 `guides(fill = guide_legend(reverse = TRUE))` 反转图例
