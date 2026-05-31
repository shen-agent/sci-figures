# 降维可视化图表

当用户需要对高维数据（基因表达矩阵、单细胞数据等）进行降维并可视化样本/细胞的分群结构时，阅读本文件。涵盖 PCA、t-SNE、UMAP 散点图以及分面散点图四种降维可视化方案。

## 涵盖图表类型

- PCA 散点图
- t-SNE 散点图
- UMAP 散点图
- 分面散点图（多条件比较）

---

## 1. PCA 散点图

### 适用场景

对样本进行主成分分析，以 PC1/PC2 为坐标展示样本空间分布，用椭圆标注分组边界，轴标签标注方差解释比例。适用于批次效应检测、样本分群评估等。

### 所需 R 包

`ggplot2`, `stats`（内置 `prcomp`）

### 输入数据格式

- 表达矩阵：行为样本，列为基因/特征（或转置后使用）
- 分组信息：data.frame，包含样本名和分组列

### R 代码模板

```r
library(ggplot2)

npg_pal <- ggsci::pal_npg("nrc")(10)

# --- PCA 计算 ---
# expr_mat: 样本 x 基因 矩阵（行=样本，列=基因）
pca_res <- prcomp(expr_mat, center = TRUE, scale. = TRUE)

# 提取方差解释比例
var_explained <- summary(pca_res)$importance[2, ] * 100

# 构建绑图数据
pca_df <- data.frame(
  PC1   = pca_res$x[, 1],
  PC2   = pca_res$x[, 2],
  Group = sample_info$group
)

# --- 绑图 ---
p <- ggplot(pca_df, aes(x = PC1, y = PC2, color = Group)) +
  geom_point(size = 2, alpha = 0.8) +
  stat_ellipse(level = 0.95, linewidth = 0.5, linetype = "dashed") +
  scale_color_manual(values = npg_pal) +
  labs(
    x = sprintf("PC1 (%.1f%%)", var_explained[1]),
    y = sprintf("PC2 (%.1f%%)", var_explained[2]),
    color = NULL
  ) +
  theme_insightlab() +
  theme(legend.position = "right")

ggsave("pca_scatter.pdf", p, width = 100, height = 80, units = "mm")
```

### 附加：碎石图（Scree Plot）

```r
# 展示各主成分的方差贡献
scree_df <- data.frame(
  PC = paste0("PC", 1:10),
  Variance = var_explained[1:10]
)
scree_df$PC <- factor(scree_df$PC, levels = scree_df$PC)

p_scree <- ggplot(scree_df, aes(x = PC, y = Variance)) +
  geom_col(fill = npg_pal[4], width = 0.6) +
  geom_line(aes(group = 1), color = npg_pal[1], linewidth = 0.5) +
  geom_point(color = npg_pal[1], size = 2) +
  labs(x = NULL, y = "Variance Explained (%)") +
  theme_insightlab()

ggsave("scree_plot.pdf", p_scree, width = 89, height = 60, units = "mm")
```

### 关键参数

| 参数 | 说明 | 建议值 |
|------|------|--------|
| `center` | 数据中心化 | TRUE（必须） |
| `scale.` | 数据标准化 | TRUE（各基因量级不同时） |
| `stat_ellipse(level)` | 椭圆置信水平 | 0.95 |
| 方差解释标注 | 轴标签中标注百分比 | 必须标注 |

---

## 2. t-SNE 散点图

### 适用场景

非线性降维，擅长保留局部结构，适用于样本量较大、分群结构复杂的数据。在单细胞分析和大队列样本聚类中广泛使用。

### 所需 R 包

`Rtsne`, `ggplot2`

### 输入数据格式

- 数值矩阵：行为样本，列为特征（不允许重复行）
- 分组信息：data.frame

### R 代码模板

```r
library(Rtsne)
library(ggplot2)

npg_pal <- ggsci::pal_npg("nrc")(10)

# --- t-SNE 计算 ---
set.seed(42)
tsne_res <- Rtsne(
  expr_mat,
  dims = 2,
  perplexity = 30,     # 建议 5-50，样本量的 1/3-1/5
  max_iter = 1000,
  check_duplicates = FALSE,
  pca = TRUE,           # 先 PCA 降维加速
  pca_center = TRUE,
  initial_dims = 50     # PCA 保留维度数
)

tsne_df <- data.frame(
  tSNE1 = tsne_res$Y[, 1],
  tSNE2 = tsne_res$Y[, 2],
  Group = sample_info$group
)

# --- 绑图 ---
p <- ggplot(tsne_df, aes(x = tSNE1, y = tSNE2, color = Group)) +
  geom_point(size = 1.5, alpha = 0.8) +
  scale_color_manual(values = npg_pal) +
  labs(x = "t-SNE 1", y = "t-SNE 2", color = NULL) +
  theme_insightlab() +
  theme(legend.position = "right")

ggsave("tsne_scatter.pdf", p, width = 100, height = 80, units = "mm")
```

### 关键参数

| 参数 | 说明 | 建议值 |
|------|------|--------|
| `perplexity` | 困惑度，影响局部/全局结构平衡 | 5-50，通常 30 |
| `max_iter` | 最大迭代次数 | 1000（至少 500） |
| `initial_dims` | PCA 预降维维度 | 30-50 |
| `set.seed` | 随机种子 | 必须设定以保证可重复性 |

---

## 3. UMAP 散点图

### 适用场景

非线性降维，比 t-SNE 更好地保留全局结构，运算速度更快。是当前单细胞分析和高维组学数据降维的主流方法。

### 所需 R 包

方案 A: `umap`（R 原生实现）
方案 B: `uwot`（速度更快，推荐）

### R 代码模板 (uwot)

```r
library(uwot)
library(ggplot2)

npg_pal <- ggsci::pal_npg("nrc")(10)

# --- UMAP 计算 ---
set.seed(42)
umap_res <- umap(
  expr_mat,
  n_neighbors = 15,    # 近邻数，影响局部结构
  n_components = 2,
  min_dist = 0.1,      # 最小距离，越小点越聚集
  metric = "euclidean",
  n_threads = 4
)

umap_df <- data.frame(
  UMAP1 = umap_res[, 1],
  UMAP2 = umap_res[, 2],
  Group = sample_info$group
)

# --- 绑图 ---
p <- ggplot(umap_df, aes(x = UMAP1, y = UMAP2, color = Group)) +
  geom_point(size = 1.2, alpha = 0.8) +
  scale_color_manual(values = npg_pal) +
  labs(x = "UMAP 1", y = "UMAP 2", color = NULL) +
  theme_insightlab() +
  theme(legend.position = "right")

ggsave("umap_scatter.pdf", p, width = 100, height = 80, units = "mm")
```

### R 代码模板 (umap 包)

```r
library(umap)
library(ggplot2)

npg_pal <- ggsci::pal_npg("nrc")(10)

set.seed(42)
umap_config <- umap.defaults
umap_config$n_neighbors <- 15
umap_config$min_dist    <- 0.1
umap_config$n_components <- 2

umap_res <- umap(expr_mat, config = umap_config)

umap_df <- data.frame(
  UMAP1 = umap_res$layout[, 1],
  UMAP2 = umap_res$layout[, 2],
  Group = sample_info$group
)

p <- ggplot(umap_df, aes(x = UMAP1, y = UMAP2, color = Group)) +
  geom_point(size = 1.2, alpha = 0.8) +
  scale_color_manual(values = npg_pal) +
  labs(x = "UMAP 1", y = "UMAP 2", color = NULL) +
  theme_insightlab()

ggsave("umap_scatter_v2.pdf", p, width = 100, height = 80, units = "mm")
```

### 连续变量着色（如基因表达）

```r
# 用连续色阶展示单基因表达
umap_df$Expression <- expr_mat[, "TP53"]

p_expr <- ggplot(umap_df, aes(x = UMAP1, y = UMAP2, color = Expression)) +
  geom_point(size = 1, alpha = 0.8) +
  scale_color_gradient(low = "grey90", high = npg_pal[1], name = "TP53") +
  labs(x = "UMAP 1", y = "UMAP 2") +
  theme_insightlab()

ggsave("umap_expression.pdf", p_expr, width = 110, height = 80, units = "mm")
```

### 关键参数

| 参数 | 说明 | 建议值 |
|------|------|--------|
| `n_neighbors` | 近邻数，越大越关注全局结构 | 10-30，通常 15 |
| `min_dist` | 嵌入点最小距离，越小簇越紧 | 0.01-0.5，通常 0.1 |
| `metric` | 距离度量 | "euclidean" 或 "cosine" |
| `set.seed` | 随机种子 | 必须设定 |

---

## 4. 分面散点图（多条件比较）

### 适用场景

在同一降维空间中，按条件（如批次、时间点、处理组）分面展示样本分布差异。便于比较不同条件下的聚类变化。

### 所需 R 包

`ggplot2`

### R 代码模板 (facet_wrap)

```r
library(ggplot2)

npg_pal <- ggsci::pal_npg("nrc")(10)

# umap_df 需额外包含 Condition 列（分面变量）
# 例如：Batch, Timepoint, Treatment 等

p <- ggplot(umap_df, aes(x = UMAP1, y = UMAP2, color = CellType)) +
  geom_point(size = 0.8, alpha = 0.7) +
  scale_color_manual(values = npg_pal) +
  facet_wrap(~ Condition, ncol = 3) +
  labs(x = "UMAP 1", y = "UMAP 2", color = "Cell Type") +
  theme_insightlab() +
  theme(
    strip.text = element_text(face = "bold", size = 9),
    legend.position = "bottom"
  )

ggsave("umap_faceted.pdf", p, width = 183, height = 80, units = "mm")
```

### R 代码模板 (facet_grid，双变量分面)

```r
# 行=Treatment，列=Timepoint
p <- ggplot(umap_df, aes(x = UMAP1, y = UMAP2, color = CellType)) +
  geom_point(size = 0.6, alpha = 0.7) +
  scale_color_manual(values = npg_pal) +
  facet_grid(Treatment ~ Timepoint) +
  labs(x = "UMAP 1", y = "UMAP 2", color = "Cell Type") +
  theme_insightlab() +
  theme(strip.text = element_text(face = "bold", size = 8))

ggsave("umap_facet_grid.pdf", p, width = 183, height = 120, units = "mm")
```

### 高亮特定分组（灰底突出法）

```r
# 在每个分面中高亮该条件的细胞，其他灰色背景
library(dplyr)

# 创建背景层数据（全部灰色）
bg_data <- umap_df %>% select(-Condition)

p <- ggplot(umap_df, aes(x = UMAP1, y = UMAP2)) +
  geom_point(data = bg_data, color = "grey85", size = 0.3) +
  geom_point(aes(color = CellType), size = 0.8, alpha = 0.8) +
  scale_color_manual(values = npg_pal) +
  facet_wrap(~ Condition, ncol = 3) +
  labs(x = "UMAP 1", y = "UMAP 2", color = "Cell Type") +
  theme_insightlab() +
  theme(strip.text = element_text(face = "bold", size = 9))

ggsave("umap_highlight.pdf", p, width = 183, height = 80, units = "mm")
```

### 关键参数

| 参数 | 说明 | 建议值 |
|------|------|--------|
| `facet_wrap(ncol)` | 每行面板数 | 2-4 |
| `facet_grid` | 行列双变量分面 | 变量组合不超过 12 格 |
| `scales` | 是否共享坐标轴 | 降维图通常 `"fixed"`（共享） |
| 灰底高亮法 | 背景层不含分面变量 | 自动在所有面板中重复 |

---

## 常见问题

1. **PCA 结果轴标签未标注方差比例**：必须在轴标签中标注（如 "PC1 (32.5%)"），这是审稿人必查项。使用 `summary(pca_res)$importance[2, ]` 提取。

2. **t-SNE 每次运行结果不同**：t-SNE 是随机算法，必须设定 `set.seed()` 保证可重复性。注意 `Rtsne` 与 `uwot` 的实现细节不同，即使相同 seed 结果也不同。

3. **UMAP 点太密看不清**：减小 `point size`（0.3-0.8），调整 `alpha`（0.5-0.7），或增大 `min_dist`（0.3-0.5）让簇更分散。

4. **降维前是否需要先选特征**：是。建议先筛选高变异基因（如 top 2000 HVG）再降维，全基因组降维噪声大且慢。

5. **PCA 输入矩阵方向错误**：`prcomp()` 要求行=样本、列=变量。如果表达矩阵是基因x样本，需先 `t()` 转置。

6. **分面散点图图例太长**：使用 `guides(color = guide_legend(ncol = 2))` 设置图例列数，或将图例放到底部 `legend.position = "bottom"`。
