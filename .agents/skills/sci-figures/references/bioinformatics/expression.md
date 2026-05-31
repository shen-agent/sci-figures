# 差异表达与富集分析图表

当用户需要可视化差异表达分析结果（DESeq2、edgeR、limma 输出）或功能富集分析结果（clusterProfiler 输出的 GO/KEGG）时，阅读本文件。涵盖火山图、表达热图、MA Plot 和富集气泡图四类常用图表。

## 涵盖图表类型

- 火山图 (Volcano Plot)
- 表达热图 (Heatmap)
- MA Plot
- KEGG/GO 富集气泡图 (Enrichment Bubble Plot)

---

## 1. 火山图 (Volcano Plot)

### 适用场景

展示差异表达基因的全局分布，横轴为 log2FoldChange，纵轴为 -log10(p.adjust)，通过阈值线区分显著上调/下调/不显著基因，并标注关键基因名。

### 所需 R 包

`ggplot2`, `ggrepel`, `dplyr`

### 输入数据格式

data.frame，至少包含以下列：

| 列名 | 类型 | 说明 |
|------|------|------|
| gene | character | 基因名 |
| log2FoldChange | numeric | log2 倍数变化 |
| padj | numeric | 校正后 P 值 |

### R 代码模板

```r
library(ggplot2)
library(ggrepel)
library(dplyr)

npg_pal <- ggsci::pal_npg("nrc")(10)

# --- 参数设置 ---
fc_cutoff  <- 1      # log2FC 阈值
p_cutoff   <- 0.05   # padj 阈值
top_n      <- 10     # 标注前 N 个显著基因

# --- 数据预处理 ---
deg <- deg %>%
  mutate(
    change = case_when(
      log2FoldChange >=  fc_cutoff & padj < p_cutoff ~ "Up",
      log2FoldChange <= -fc_cutoff & padj < p_cutoff ~ "Down",
      TRUE ~ "NS"
    ),
    neg_log10_padj = -log10(padj)
  )

# 选取标注基因（按 padj 最小）
top_genes <- deg %>%
  filter(change != "NS") %>%
  slice_min(padj, n = top_n)

# --- 绑图 ---
p <- ggplot(deg, aes(x = log2FoldChange, y = neg_log10_padj, color = change)) +
  geom_point(size = 0.8, alpha = 0.7) +
  scale_color_manual(
    values = c("Up" = npg_pal[1], "Down" = npg_pal[2], "NS" = "grey70"),
    labels = c("Up" = "Up-regulated", "Down" = "Down-regulated", "NS" = "Not Sig.")
  ) +
  geom_vline(xintercept = c(-fc_cutoff, fc_cutoff), linetype = "dashed", color = "grey40", linewidth = 0.3) +
  geom_hline(yintercept = -log10(p_cutoff), linetype = "dashed", color = "grey40", linewidth = 0.3) +
  geom_text_repel(
    data = top_genes, aes(label = gene),
    size = 2.5, max.overlaps = 20, segment.size = 0.2, color = "black"
  ) +
  labs(x = expression(log[2]~Fold~Change), y = expression(-log[10]~p.adjust), color = NULL) +
  theme_insightlab() +
  theme(legend.position = c(0.85, 0.85))

ggsave("volcano_plot.pdf", p, width = 89, height = 80, units = "mm")
```

### 关键参数

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `fc_cutoff` | log2FC 显著性阈值 | 1 |
| `p_cutoff` | padj 显著性阈值 | 0.05 |
| `top_n` | 标注基因数量 | 10 |
| `max.overlaps` | ggrepel 允许的最大重叠数 | 20 |

---

## 2. 表达热图 (Heatmap)

### 适用场景

展示一组基因在多个样本中的表达模式，通过层次聚类揭示样本/基因的分组关系。常用于展示 Top DEGs 的表达矩阵。

### 所需 R 包

方案 A: `pheatmap`（简单快速）
方案 B: `ComplexHeatmap`（高度定制，推荐发表级别）

### 输入数据格式

- 表达矩阵：行为基因，列为样本，值为标准化后的表达量（如 z-score）
- 注释信息（可选）：data.frame，行名为样本名，列为分组变量

### R 代码模板 (ComplexHeatmap)

```r
library(ComplexHeatmap)
library(circlize)

npg_pal <- ggsci::pal_npg("nrc")(10)

# --- 数据准备 ---
# mat: 基因 x 样本 的数值矩阵（建议 z-score 标准化）
mat_scaled <- t(scale(t(mat)))

# 样本注释
anno_col <- HeatmapAnnotation(
  Group = sample_info$group,
  col = list(Group = c("Tumor" = npg_pal[1], "Normal" = npg_pal[2])),
  annotation_name_gp = gpar(fontsize = 8),
  simple_anno_size = unit(3, "mm")
)

# 色阶
col_fun <- colorRamp2(c(-2, 0, 2), c("#3C5488", "#FFFFFF", "#E64B35"))

# --- 绑图 ---
ht <- Heatmap(
  mat_scaled,
  name = "z-score",
  col = col_fun,
  top_annotation = anno_col,
  cluster_rows = TRUE,
  cluster_columns = TRUE,
  show_row_names = nrow(mat_scaled) <= 50,
  show_column_names = TRUE,
  row_names_gp = gpar(fontsize = 6),
  column_names_gp = gpar(fontsize = 7),
  row_title_gp = gpar(fontsize = 8),
  column_title_gp = gpar(fontsize = 8),
  heatmap_legend_param = list(
    title_gp = gpar(fontsize = 8),
    labels_gp = gpar(fontsize = 7)
  )
)

pdf("heatmap.pdf", width = 7, height = 5)
draw(ht)
dev.off()
```

### R 代码模板 (pheatmap，简洁版)

```r
library(pheatmap)

npg_pal <- ggsci::pal_npg("nrc")(10)

mat_scaled <- t(scale(t(mat)))

annotation_col <- data.frame(
  Group = sample_info$group,
  row.names = colnames(mat_scaled)
)
ann_colors <- list(Group = c("Tumor" = npg_pal[1], "Normal" = npg_pal[2]))

pheatmap(
  mat_scaled,
  color = colorRampPalette(c("#3C5488", "#FFFFFF", "#E64B35"))(100),
  annotation_col = annotation_col,
  annotation_colors = ann_colors,
  cluster_rows = TRUE,
  cluster_cols = TRUE,
  show_rownames = nrow(mat_scaled) <= 50,
  fontsize = 8,
  fontsize_row = 6,
  fontsize_col = 7,
  border_color = NA,
  filename = "heatmap_pheatmap.pdf",
  width = 7, height = 5
)
```

### 关键参数

| 参数 | 说明 | 建议值 |
|------|------|--------|
| `cluster_rows/columns` | 是否聚类 | 通常 TRUE |
| `show_row_names` | 是否显示基因名 | 基因数 <= 50 时 TRUE |
| `scale` | pheatmap 内置标准化 | `"row"` 或预先手动 z-score |
| `colorRamp2` | ComplexHeatmap 色阶函数 | `c(-2,0,2)` 对应蓝白红 |

---

## 3. MA Plot

### 适用场景

展示差异表达的另一种视角：横轴为平均表达量（baseMean），纵轴为 log2FoldChange。适合评估差异表达与表达丰度的关系。

### 所需 R 包

`ggplot2`, `dplyr`

### 输入数据格式

data.frame，至少包含：`baseMean`（平均表达量）、`log2FoldChange`、`padj`

### R 代码模板

```r
library(ggplot2)
library(dplyr)

npg_pal <- ggsci::pal_npg("nrc")(10)

fc_cutoff <- 1
p_cutoff  <- 0.05

deg <- deg %>%
  mutate(
    change = case_when(
      log2FoldChange >=  fc_cutoff & padj < p_cutoff ~ "Up",
      log2FoldChange <= -fc_cutoff & padj < p_cutoff ~ "Down",
      TRUE ~ "NS"
    )
  )

p <- ggplot(deg, aes(x = log10(baseMean + 1), y = log2FoldChange, color = change)) +
  geom_point(size = 0.6, alpha = 0.6) +
  scale_color_manual(
    values = c("Up" = npg_pal[1], "Down" = npg_pal[2], "NS" = "grey70")
  ) +
  geom_hline(yintercept = c(-fc_cutoff, 0, fc_cutoff),
             linetype = c("dashed", "solid", "dashed"),
             color = c("grey40", "black", "grey40"),
             linewidth = c(0.3, 0.3, 0.3)) +
  labs(x = expression(log[10]~(baseMean + 1)), y = expression(log[2]~Fold~Change), color = NULL) +
  theme_insightlab()

ggsave("ma_plot.pdf", p, width = 89, height = 70, units = "mm")
```

### 关键参数

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `baseMean` | DESeq2 输出的平均表达量 | — |
| `log10(baseMean + 1)` | 对数变换避免零值 | +1 伪计数 |

---

## 4. KEGG/GO 富集气泡图 (Enrichment Bubble Plot)

### 适用场景

展示 clusterProfiler 富集分析结果（enrichGO / enrichKEGG / GSEA 等），横轴为 GeneRatio，纵轴为通路名称 (Term)，点大小表示基因数 (Count)，颜色表示显著性 (p.adjust)。

### 所需 R 包

`ggplot2`, `clusterProfiler`（上游分析）, `dplyr`, `forcats`

### 输入数据格式

clusterProfiler 输出对象（enrichResult），或提取后的 data.frame，包含：

| 列名 | 类型 | 说明 |
|------|------|------|
| Description | character | 通路/GO Term 名称 |
| GeneRatio | character | 如 "15/200"，需转为数值 |
| Count | integer | 富集基因数 |
| p.adjust | numeric | 校正后 P 值 |

### R 代码模板

```r
library(ggplot2)
library(dplyr)
library(forcats)

# --- 从 clusterProfiler 结果提取数据 ---
# ego 为 enrichGO() 或 enrichKEGG() 的输出对象
enrich_df <- as.data.frame(ego)

# 转换 GeneRatio 为数值
enrich_df <- enrich_df %>%
  mutate(
    GeneRatio_num = sapply(GeneRatio, function(x) {
      parts <- as.numeric(strsplit(x, "/")[[1]])
      parts[1] / parts[2]
    })
  )

# 取 Top N 通路
top_n_terms <- 20
plot_df <- enrich_df %>%
  slice_min(p.adjust, n = top_n_terms) %>%
  mutate(Description = fct_reorder(Description, GeneRatio_num))

# --- 绑图 ---
p <- ggplot(plot_df, aes(x = GeneRatio_num, y = Description)) +
  geom_point(aes(size = Count, color = p.adjust)) +
  scale_color_gradient(low = "#E64B35", high = "#3C5488",
                       name = "p.adjust") +
  scale_size_continuous(range = c(2, 6), name = "Gene Count") +
  labs(x = "Gene Ratio", y = NULL, title = "GO Enrichment Analysis") +
  theme_insightlab() +
  theme(
    axis.text.y = element_text(size = 7),
    legend.position = "right"
  )

ggsave("enrichment_bubble.pdf", p, width = 140, height = 120, units = "mm")
```

### 分面展示多类别（BP / CC / MF 同时展示）

```r
# 合并三类 GO 结果
bp_df <- as.data.frame(ego_bp) %>% mutate(Category = "BP") %>% slice_min(p.adjust, n = 10)
cc_df <- as.data.frame(ego_cc) %>% mutate(Category = "CC") %>% slice_min(p.adjust, n = 10)
mf_df <- as.data.frame(ego_mf) %>% mutate(Category = "MF") %>% slice_min(p.adjust, n = 10)

combined_df <- bind_rows(bp_df, cc_df, mf_df) %>%
  mutate(
    GeneRatio_num = sapply(GeneRatio, function(x) {
      parts <- as.numeric(strsplit(x, "/")[[1]])
      parts[1] / parts[2]
    }),
    Description = fct_reorder(Description, GeneRatio_num)
  )

p <- ggplot(combined_df, aes(x = GeneRatio_num, y = Description)) +
  geom_point(aes(size = Count, color = p.adjust)) +
  scale_color_gradient(low = "#E64B35", high = "#3C5488") +
  scale_size_continuous(range = c(2, 5)) +
  facet_grid(Category ~ ., scales = "free_y", space = "free_y") +
  labs(x = "Gene Ratio", y = NULL) +
  theme_insightlab() +
  theme(strip.text = element_text(face = "bold", size = 9))

ggsave("go_three_category.pdf", p, width = 160, height = 180, units = "mm")
```

### 关键参数

| 参数 | 说明 | 建议值 |
|------|------|--------|
| `top_n_terms` | 展示的通路数量 | 15-25 |
| `scale_size range` | 点大小范围 | `c(2, 6)` |
| `scales = "free_y"` | 分面时各面板独立 Y 轴 | 推荐 |
| `space = "free_y"` | 分面时面板高度按条目数自适应 | 推荐 |

---

## 常见问题

1. **火山图基因标签重叠严重**：增大 `max.overlaps`（如 30），或减少 `top_n`，或使用 `geom_label_repel` 代替 `geom_text_repel`。

2. **热图行名太密看不清**：基因数超过 50 时建议 `show_row_names = FALSE`，或只展示 Top DEGs。使用 `row_split` 参数按聚类分块显示。

3. **GeneRatio 列为字符串无法画图**：需要用 `strsplit` 转换为数值，模板已包含转换代码。

4. **ComplexHeatmap 与 ggplot2 混排**：两者绘图系统不同，不能直接用 `+` 拼接。用 `patchwork::wrap_elements(grid.grabExpr(draw(ht)))` 桥接。

5. **富集气泡图 Y 轴标签被截断**：增大 `width` 参数，或用 `stringr::str_wrap(Description, width = 40)` 自动换行。

6. **padj 列存在 NA 值导致报错**：在绑图前用 `filter(!is.na(padj))` 去除 NA 行。
