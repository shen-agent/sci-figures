# 突变景观图表

当用户需要可视化基因组突变数据（MAF 文件、突变矩阵）时，阅读本文件。涵盖 OncoPrint、Lollipop Plot、Rainfall Plot、TiTv Plot、Manhattan Plot 和基因互作图六类突变分析常用图表。

## 涵盖图表类型

- OncoPrint（突变全景图）
- Lollipop Plot（突变位点棒棒糖图）
- Rainfall Plot（突变密度降雨图）
- TiTv Plot（转换/颠换比例图）
- Manhattan Plot（曼哈顿图）
- Gene Interaction Plot（基因互作/互斥图）

---

## 1. OncoPrint（突变全景图）

### 适用场景

展示多个样本在多个基因上的突变类型分布，以矩阵形式呈现，顶部和右侧附加汇总条形图。常见于 TCGA 类队列研究。

### 所需 R 包

`ComplexHeatmap`, `circlize`

### 输入数据格式

突变矩阵：行为基因，列为样本，值为突变类型字符串（如 `"Missense_Mutation"`, `"Frame_Shift_Del"` 等），无突变为空字符串 `""`。

### R 代码模板

```r
library(ComplexHeatmap)

npg_pal <- ggsci::pal_npg("nrc")(10)

# --- 突变类型配色 ---
mut_colors <- c(
  "Missense_Mutation"   = npg_pal[1],
  "Frame_Shift_Del"     = npg_pal[2],
  "Frame_Shift_Ins"     = npg_pal[3],
  "Nonsense_Mutation"   = npg_pal[4],
  "Splice_Site"         = npg_pal[5],
  "In_Frame_Del"        = npg_pal[6],
  "In_Frame_Ins"        = npg_pal[7],
  "Multi_Hit"           = npg_pal[9]   # 棕色，避开与 npg_pal[1] 砖红的撞色
)

# --- 自定义绘制函数 ---
alter_fun <- list(
  background = function(x, y, w, h) {
    grid.rect(x, y, w * 0.9, h * 0.9,
              gp = gpar(fill = "#F0F0F0", col = NA))
  },
  Missense_Mutation = function(x, y, w, h) {
    grid.rect(x, y, w * 0.9, h * 0.9,
              gp = gpar(fill = mut_colors["Missense_Mutation"], col = NA))
  },
  Frame_Shift_Del = function(x, y, w, h) {
    grid.rect(x, y, w * 0.9, h * 0.4,
              gp = gpar(fill = mut_colors["Frame_Shift_Del"], col = NA))
  },
  Frame_Shift_Ins = function(x, y, w, h) {
    grid.rect(x, y, w * 0.9, h * 0.4,
              gp = gpar(fill = mut_colors["Frame_Shift_Ins"], col = NA))
  },
  Nonsense_Mutation = function(x, y, w, h) {
    grid.rect(x, y, w * 0.9, h * 0.9,
              gp = gpar(fill = mut_colors["Nonsense_Mutation"], col = NA))
  },
  Splice_Site = function(x, y, w, h) {
    grid.rect(x, y, w * 0.9, h * 0.4,
              gp = gpar(fill = mut_colors["Splice_Site"], col = NA))
  },
  In_Frame_Del = function(x, y, w, h) {
    grid.rect(x, y, w * 0.9, h * 0.4,
              gp = gpar(fill = mut_colors["In_Frame_Del"], col = NA))
  },
  In_Frame_Ins = function(x, y, w, h) {
    grid.rect(x, y, w * 0.9, h * 0.4,
              gp = gpar(fill = mut_colors["In_Frame_Ins"], col = NA))
  },
  Multi_Hit = function(x, y, w, h) {
    grid.rect(x, y, w * 0.9, h * 0.9,
              gp = gpar(fill = mut_colors["Multi_Hit"], col = NA))
  }
)

# --- 绑图 ---
# mat: 基因 x 样本矩阵，以 ";" 分隔多突变（如 "Missense_Mutation;Frame_Shift_Del"）
onco <- oncoPrint(
  mat,
  alter_fun = alter_fun,
  col = mut_colors,
  column_title = "OncoPrint",
  column_title_gp = gpar(fontsize = 10, fontface = "bold"),
  row_names_gp = gpar(fontsize = 8, fontface = "italic"),
  pct_gp = gpar(fontsize = 7),
  show_pct = TRUE,
  row_order = NULL,       # NULL 表示按突变频率排序
  column_order = NULL,    # NULL 表示按突变数排序
  top_annotation = HeatmapAnnotation(
    cbar = anno_oncoprint_barplot(height = unit(1, "cm")),
    annotation_name_gp = gpar(fontsize = 7)
  ),
  right_annotation = rowAnnotation(
    rbar = anno_oncoprint_barplot(width = unit(1.5, "cm")),
    annotation_name_gp = gpar(fontsize = 7)
  ),
  heatmap_legend_param = list(
    title = "Mutation Type",
    title_gp = gpar(fontsize = 8, fontface = "bold"),
    labels_gp = gpar(fontsize = 7)
  )
)

pdf("oncoprint.pdf", width = 10, height = 6)
draw(onco)
dev.off()
```

### 关键参数

| 参数 | 说明 | 建议值 |
|------|------|--------|
| `alter_fun` | 每种突变类型的绘制函数 | 全高矩形=点突变，半高=插入缺失 |
| `show_pct` | 显示每行突变百分比 | TRUE |
| `row_order` | 基因排序 | NULL（按频率自动排序） |
| `column_order` | 样本排序 | NULL（按突变数排序） |

---

## 2. Lollipop Plot（突变位点棒棒糖图）

### 适用场景

展示单个基因上的突变位点分布，纵轴为突变次数，横轴为蛋白质位置，蛋白质结构域以色块标注。常用于展示 driver gene 的突变 hotspot。

### 所需 R 包

`maftools`

### 输入数据格式

MAF 对象（由 `maftools::read.MAF()` 读取）

### R 代码模板

```r
library(maftools)

npg_pal <- ggsci::pal_npg("nrc")(10)

# --- 读取 MAF 文件 ---
maf <- read.MAF(maf = "mutations.maf", clinicalData = "clinical.tsv")

# --- 绑图 ---
lollipopPlot(
  maf = maf,
  gene = "TP53",
  AACol = "HGVSp_Short",          # 氨基酸变异列名
  showMutationRate = TRUE,
  showDomainLabel = TRUE,
  labelPos = "all",                # 标注所有 hotspot
  labPosSize = 0.8,
  colors = npg_pal[1:7],    # 突变类型配色
  domainLabelSize = 2.5,
  axisTextSize = c(0.8, 0.8),
  pointSize = 1.5
)

# 导出：maftools 使用 base graphics，需用 pdf/png 包裹
pdf("lollipop_TP53.pdf", width = 8, height = 3)
lollipopPlot(maf = maf, gene = "TP53", AACol = "HGVSp_Short",
             colors = npg_pal[1:7], showDomainLabel = TRUE)
dev.off()
```

### 关键参数

| 参数 | 说明 | 建议值 |
|------|------|--------|
| `gene` | 目标基因 | 如 "TP53", "EGFR" |
| `AACol` | MAF 中氨基酸变化列 | "HGVSp_Short" 或 "Protein_Change" |
| `labelPos` | 标注位置 | "all" 或具体位点编号 |
| `showDomainLabel` | 显示蛋白质结构域名称 | TRUE |

---

## 3. Rainfall Plot（突变密度降雨图）

### 适用场景

展示单个样本全基因组范围的突变密度分布，纵轴为相邻突变间的距离（log10），用于识别突变密集区域（kataegis）。

### 所需 R 包

`maftools`

### R 代码模板

```r
library(maftools)

maf <- read.MAF(maf = "mutations.maf")

# --- 绑图 ---
pdf("rainfall_plot.pdf", width = 10, height = 4)
rainfallPlot(
  maf = maf,
  tsb = "SAMPLE_001",        # 指定样本 ID
  detectChangePoints = TRUE,  # 检测 kataegis 区域
  pointSize = 0.4,
  fontSize = 0.8
)
dev.off()
```

### 关键参数

| 参数 | 说明 | 建议值 |
|------|------|--------|
| `tsb` | 样本 Tumor_Sample_Barcode | 具体样本 ID |
| `detectChangePoints` | 是否自动检测 kataegis 区域 | TRUE |
| `pointSize` | 点大小 | 0.4 |

---

## 4. TiTv Plot（转换/颠换比例图）

### 适用场景

展示 SNV 的六种碱基替换类型（Ti: C>T, T>C; Tv: C>A, C>G, T>A, T>G）在各样本中的比例分布，用于质控和突变特征分析。

### 所需 R 包

`maftools`

### R 代码模板

```r
library(maftools)

maf <- read.MAF(maf = "mutations.maf")

# --- 计算 Ti/Tv ---
titv_result <- titv(maf = maf, plot = FALSE, useSyn = TRUE)

# --- 绑图 ---
pdf("titv_plot.pdf", width = 8, height = 5)
plotTiTv(res = titv_result,
         showBarcodes = TRUE,
         textSize = 0.6,
         baseFontSize = 0.8)
dev.off()
```

### 关键参数

| 参数 | 说明 | 建议值 |
|------|------|--------|
| `useSyn` | 是否包含同义突变 | FALSE（默认）或 TRUE |
| `showBarcodes` | X 轴是否显示样本名 | 样本少时 TRUE |

---

## 5. Manhattan Plot（曼哈顿图）

### 适用场景

展示 GWAS 或全基因组关联分析结果，横轴为染色体位置，纵轴为 -log10(P)，用于识别显著关联位点。

### 所需 R 包

`ggplot2`, `dplyr`

### 输入数据格式

| 列名 | 类型 | 说明 |
|------|------|------|
| CHR | integer | 染色体编号 |
| BP | integer | 碱基位置 |
| P | numeric | P 值 |
| SNP | character | SNP ID（可选，用于标注） |

### R 代码模板

```r
library(ggplot2)
library(dplyr)

npg_pal <- ggsci::pal_npg("nrc")(10)

# --- 数据准备 ---
gwas <- gwas %>%
  group_by(CHR) %>%
  summarise(chr_len = max(BP), .groups = "drop") %>%
  mutate(tot = cumsum(as.numeric(chr_len)) - chr_len) %>%
  select(-chr_len) %>%
  left_join(gwas, by = "CHR") %>%
  arrange(CHR, BP) %>%
  mutate(BPcum = BP + tot)

# X 轴刻度位置
axis_df <- gwas %>%
  group_by(CHR) %>%
  summarise(center = (max(BPcum) + min(BPcum)) / 2, .groups = "drop")

# 显著性阈值
genome_wide  <- 5e-8
suggestive   <- 1e-5

# --- 绑图 ---
p <- ggplot(gwas, aes(x = BPcum, y = -log10(P))) +
  geom_point(aes(color = as.factor(CHR %% 2)), size = 0.5, alpha = 0.7) +
  scale_color_manual(values = c(npg_pal[4], npg_pal[6])) +
  scale_x_continuous(labels = axis_df$CHR, breaks = axis_df$center) +
  geom_hline(yintercept = -log10(genome_wide), color = npg_pal[1],
             linetype = "dashed", linewidth = 0.3) +
  geom_hline(yintercept = -log10(suggestive), color = npg_pal[2],
             linetype = "dashed", linewidth = 0.3) +
  labs(x = "Chromosome", y = expression(-log[10](P))) +
  theme_insightlab() +
  theme(legend.position = "none",
        panel.grid.major.x = element_blank())

ggsave("manhattan_plot.pdf", p, width = 183, height = 70, units = "mm")
```

### 关键参数

| 参数 | 说明 | 建议值 |
|------|------|--------|
| `genome_wide` | 全基因组显著性阈值 | 5e-8 |
| `suggestive` | 提示性显著阈值 | 1e-5 |
| 交替染色体配色 | 区分相邻染色体 | 双色交替 |

---

## 6. Gene Interaction Plot（基因互作/互斥图）

### 适用场景

检验并展示一组基因之间的共突变 (co-occurrence) 或互斥 (mutual exclusivity) 关系，以热图矩阵形式展示 Fisher 检验的 P 值和 odds ratio。

### 所需 R 包

方案 A: `maftools`（内置函数）
方案 B: 自定义 Fisher 检验 + `ggplot2`

### R 代码模板 (maftools)

```r
library(maftools)

maf <- read.MAF(maf = "mutations.maf")

# --- 绑图 ---
pdf("gene_interaction_maftools.pdf", width = 6, height = 5)
somaticInteractions(
  maf = maf,
  top = 25,               # 取突变频率前 25 的基因
  pvalue = c(0.05, 0.01), # 显著性阈值
  fontSize = 0.7,
  showCounts = TRUE
)
dev.off()
```

### R 代码模板 (自定义 Fisher 检验热图)

```r
library(ggplot2)
library(dplyr)
library(tidyr)

npg_pal <- ggsci::pal_npg("nrc")(10)

# --- 构建突变矩阵 ---
# mut_mat: 基因 x 样本，0/1 矩阵
genes <- rownames(mut_mat)
n <- length(genes)

# Fisher 检验
result_list <- list()
for (i in 1:(n-1)) {
  for (j in (i+1):n) {
    tbl <- table(
      factor(mut_mat[i, ], levels = c(0, 1)),
      factor(mut_mat[j, ], levels = c(0, 1))
    )
    ft <- fisher.test(tbl)
    result_list[[length(result_list) + 1]] <- data.frame(
      gene1 = genes[i], gene2 = genes[j],
      pvalue = ft$p.value, odds_ratio = ft$estimate,
      type = ifelse(ft$estimate > 1, "Co-occurrence", "Mutual Exclusivity"),
      stringsAsFactors = FALSE
    )
  }
}
interaction_df <- bind_rows(result_list)

# --- 绑图 ---
p <- ggplot(interaction_df, aes(x = gene1, y = gene2)) +
  geom_tile(aes(fill = type), color = "white", linewidth = 0.5) +
  geom_text(aes(label = ifelse(pvalue < 0.05,
                                sprintf("%.2e", pvalue), "")),
            size = 2, color = "black") +
  scale_fill_manual(values = c(
    "Co-occurrence" = npg_pal[1],
    "Mutual Exclusivity" = npg_pal[2]
  ), na.value = "grey90") +
  labs(x = NULL, y = NULL, fill = "Interaction") +
  theme_insightlab() +
  theme(
    axis.text.x = element_text(angle = 45, hjust = 1, face = "italic", size = 7),
    axis.text.y = element_text(face = "italic", size = 7)
  )

ggsave("gene_interaction.pdf", p, width = 120, height = 100, units = "mm")
```

### 关键参数

| 参数 | 说明 | 建议值 |
|------|------|--------|
| `top` | maftools 中取前 N 高频基因 | 20-30 |
| `pvalue` | 显著性阈值（双层） | c(0.05, 0.01) |
| Fisher 检验 | odds_ratio > 1 为共现，< 1 为互斥 | — |

---

## 常见问题

1. **MAF 文件列名不匹配**：`read.MAF()` 默认期望标准 MAF 列名（Hugo_Symbol, Chromosome, Start_Position 等）。如果列名不同，需用 `read.MAF(..., vc = "Variant_Classification")` 等参数显式指定。

2. **OncoPrint 矩阵构建**：从 MAF 到 OncoPrint 矩阵需要转换——对每个基因-样本对，将突变类型拼接为字符串（多突变用 `;` 分隔），无突变为 `""`。

3. **Lollipop Plot 报错 "No mutations found"**：检查 `AACol` 参数是否匹配 MAF 文件中的列名；确保该基因确实存在蛋白编码区突变。

4. **Manhattan Plot 内存溢出**：全基因组 SNP 数据量大（数百万行），先过滤低质量 SNP（如 P > 0.1），或使用 `geom_point(shape = ".")` 加速绘制。

5. **maftools 图形导出模糊**：maftools 使用 base graphics，必须用 `pdf()`/`png()` 函数包裹，不能使用 `ggsave()`。推荐 `pdf()` 导出矢量图。

6. **ComplexHeatmap 版本兼容**：`oncoPrint()` 需要 ComplexHeatmap >= 2.0。使用 `BiocManager::install("ComplexHeatmap")` 确保最新版本。
