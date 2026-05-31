# 数据分布型图表

展示数据的分布特征和统计量。当用户需要箱线图、小提琴图、密度图、直方图、带误差线图时阅读此文件。

## 包含的图表类型

1. [箱线图](#箱线图)
2. [小提琴图](#小提琴图)
3. [密度图与峰峦图](#密度图与峰峦图)
4. [直方图](#直方图)
5. [抖动散点图](#抖动散点图)
6. [蜂群图](#蜂群图)
7. [带误差线的均值图](#带误差线的均值图)
8. [带误差线的柱形图](#带误差线的柱形图)
9. [组合图：箱线+抖动/小提琴+箱线](#组合图)

## 通用前置

所有示例假定已加载：

```r
library(ggplot2)
library(ggsci)

npg_pal <- ggsci::pal_npg("nrc")(10)   # NPG 10 色调色板，详见 style-guide.md "配色方案"
# theme_insightlab() 定义见 style-guide.md "统一主题模板"
# 出图：PNG 用 ggsave(..., device = ragg::agg_png)；PDF 用 ggsave 默认（推断 .pdf）
```

## 箱线图

适用：展示数据的中位数、四分位距和异常值。

```r
p <- ggplot(df, aes(x = group, y = value, fill = group)) +
  geom_boxplot(width = 0.6, outlier.shape = 21, outlier.size = 1.5,
               linewidth = 0.3, show.legend = FALSE) +
  scale_fill_manual(values = npg_pal) +
  theme_insightlab() +
  labs(x = NULL, y = "Value")
```

变体：
- 缺口箱线图：`geom_boxplot(notch = TRUE)` — 缺口不重叠说明中位数有显著差异
- 变宽箱线图：`geom_boxplot(varwidth = TRUE)` — 宽度反映样本量

### 多系列箱线图

```r
p <- ggplot(df, aes(x = group, y = value, fill = subgroup)) +
  geom_boxplot(position = position_dodge(0.8), width = 0.6, linewidth = 0.3) +
  scale_fill_manual(values = npg_pal) +
  theme_insightlab()
```

## 小提琴图

适用：在箱线图基础上展示数据密度分布形态。通常与内嵌箱线图组合使用。

```r
p <- ggplot(df, aes(x = group, y = value, fill = group)) +
  geom_violin(trim = FALSE, linewidth = 0.3, show.legend = FALSE) +
  geom_boxplot(width = 0.1, fill = "white", linewidth = 0.3,
               outlier.shape = NA, show.legend = FALSE) +
  scale_fill_manual(values = npg_pal) +
  theme_insightlab()
```

### 分半小提琴图（两组对比）

```r
library(gghalves)

p <- ggplot(df, aes(x = group, y = value, fill = treatment)) +
  geom_split_violin(trim = FALSE) +
  scale_fill_manual(values = npg_pal[1:2]) +
  theme_insightlab()
```

## 密度图与峰峦图

### 核密度图

```r
p <- ggplot(df, aes(x = value, fill = group)) +
  geom_density(alpha = 0.5, linewidth = 0.3, color = "black") +
  scale_fill_manual(values = npg_pal) +
  theme_insightlab()
```

### 峰峦图（Ridgeline Plot）

适用：多组分布的纵向排列对比，适合展示时间序列或多类别密度。

```r
library(ggridges)

p <- ggplot(df, aes(x = value, y = group, fill = stat(x))) +
  geom_density_ridges_gradient(scale = 1.5, rel_min_height = 0.01,
                                linewidth = 0.3) +
  scale_fill_viridis_c(option = "C") +
  theme_insightlab() +
  theme(legend.position = "none")
```

## 直方图

```r
p <- ggplot(df, aes(x = value, fill = group)) +
  geom_histogram(binwidth = 1, color = "black", linewidth = 0.25,
                 alpha = 0.7, position = "identity") +
  scale_fill_manual(values = npg_pal) +
  theme_insightlab()
```

## 抖动散点图

适用：小样本量时展示每个数据点的分布。

```r
p <- ggplot(df, aes(x = group, y = value, fill = group)) +
  geom_jitter(width = 0.2, size = 2, shape = 21, stroke = 0.3,
              show.legend = FALSE) +
  scale_fill_manual(values = npg_pal) +
  theme_insightlab()
```

## 蜂群图

适用：替代抖动图，数据点不重叠，分布形态更真实。

```r
library(ggbeeswarm)

p <- ggplot(df, aes(x = group, y = value, fill = group)) +
  geom_beeswarm(size = 2, shape = 21, stroke = 0.3,
                show.legend = FALSE) +
  scale_fill_manual(values = npg_pal) +
  theme_insightlab()
```

## 带误差线的均值图

适用：展示均值 ± 标准差/标准误。

```r
p <- ggplot(df, aes(x = group, y = value, fill = group)) +
  stat_summary(fun = mean, geom = "point", size = 4, shape = 21,
               stroke = 0.5, show.legend = FALSE) +
  stat_summary(fun.data = mean_sdl, fun.args = list(mult = 1),
               geom = "errorbar", width = 0.2, linewidth = 0.5) +
  scale_fill_manual(values = npg_pal) +
  theme_insightlab()
```

## 带误差线的柱形图

```r
p <- ggplot(df, aes(x = group, y = value, fill = group)) +
  stat_summary(fun = mean, geom = "col", width = 0.6,
               color = "black", linewidth = 0.3, show.legend = FALSE) +
  stat_summary(fun.data = mean_sdl, fun.args = list(mult = 1),
               geom = "errorbar", width = 0.2, linewidth = 0.5) +
  scale_fill_manual(values = npg_pal) +
  theme_insightlab()
```

## 组合图

### 箱线图 + 抖动散点

```r
p <- ggplot(df, aes(x = group, y = value, fill = group)) +
  geom_boxplot(width = 0.5, outlier.shape = NA, linewidth = 0.3,
               show.legend = FALSE) +
  geom_jitter(width = 0.15, size = 1.5, shape = 21, stroke = 0.2,
              alpha = 0.7, show.legend = FALSE) +
  scale_fill_manual(values = npg_pal) +
  theme_insightlab()
```

### 小提琴图 + 抖动散点

```r
p <- ggplot(df, aes(x = group, y = value, fill = group)) +
  geom_violin(trim = FALSE, linewidth = 0.3, show.legend = FALSE) +
  geom_jitter(width = 0.1, size = 1, shape = 16, alpha = 0.5,
              show.legend = FALSE) +
  scale_fill_manual(values = npg_pal) +
  theme_insightlab()
```

## 常见问题

- **箱线图异常值与抖动点重复**：组合图中用 `outlier.shape = NA` 隐藏箱线图自带的异常值点
- **小提琴图底部被截**：设置 `trim = FALSE`
- **密度图面积不为 1**：`geom_density()` 默认标准化，多组时用 `position = "identity"` + `alpha`
- **误差线代表什么**：`mean_sdl(mult = 1)` 是均值 ± 1 SD，`mean_se` 是均值 ± SE
