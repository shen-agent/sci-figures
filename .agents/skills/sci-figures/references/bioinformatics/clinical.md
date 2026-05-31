# 临床图表

当用户需要可视化临床试验数据（治疗反应、诊断效能、样本集合关系）时，阅读本文件。涵盖泳道图、瀑布图、ROC 曲线和韦恩图四类临床研究常用图表。

## 涵盖图表类型

- 泳道图 (Swimmer Plot)
- 瀑布图 (Waterfall Plot)
- ROC 曲线 (Receiver Operating Characteristic Curve)
- 韦恩图 (Venn Diagram)

---

## 1. 泳道图 (Swimmer Plot)

### 适用场景

展示每位患者的治疗时间轴，以水平条形表示持续时间，叠加事件标记（如响应、进展、死亡等）。常见于临床试验中药物疗效评估的个体化展示。

### 所需 R 包

`ggplot2`, `dplyr`（可选 `swimplot` 辅助构建）

### 输入数据格式

**基础层（条形）**：

| 列名 | 类型 | 说明 |
|------|------|------|
| patient_id | character | 患者编号 |
| start | numeric | 开始时间（通常为 0） |
| end | numeric | 结束时间（月） |
| response | character | 最佳响应（CR/PR/SD/PD） |

**事件层（点/箭头）**：

| 列名 | 类型 | 说明 |
|------|------|------|
| patient_id | character | 患者编号 |
| time | numeric | 事件发生时间 |
| event | character | 事件类型 |

### R 代码模板（两层：条形 + 事件点）

```r
library(ggplot2)
library(dplyr)

npg_pal <- ggsci::pal_npg("nrc")(10)

# --- 数据准备 ---
# 按治疗持续时间排序
bar_df <- bar_df %>%
  mutate(patient_id = factor(patient_id,
                              levels = patient_id[order(end)]))

# --- 绑图 ---
p <- ggplot() +
  # 第一层：治疗持续时间条形
  geom_segment(
    data = bar_df,
    aes(x = start, xend = end, y = patient_id, yend = patient_id, color = response),
    linewidth = 3, lineend = "butt"
  ) +
  # 第二层：事件点
  geom_point(
    data = event_df,
    aes(x = time, y = patient_id, shape = event),
    size = 2.5, fill = "black", color = "black"
  ) +
  scale_color_manual(
    values = c("CR" = npg_pal[3], "PR" = npg_pal[2],
               "SD" = npg_pal[6], "PD" = npg_pal[1]),
    name = "Best Response"
  ) +
  scale_shape_manual(
    values = c("Progression" = 4, "Death" = 8, "Ongoing" = 16),
    name = "Event"
  ) +
  labs(x = "Time (months)", y = "Patient") +
  theme_insightlab() +
  theme(
    axis.text.y = element_text(size = 6),
    legend.position = "bottom",
    legend.box = "horizontal"
  )

ggsave("swimmer_2layers.pdf", p, width = 140, height = 120, units = "mm")
```

### R 代码模板（三层：条形 + 事件点 + 持续箭头）

```r
library(ggplot2)
library(dplyr)

npg_pal <- ggsci::pal_npg("nrc")(10)

# ongoing 患者添加箭头
ongoing_df <- bar_df %>% filter(ongoing == TRUE)

p <- ggplot() +
  # 第一层：条形
  geom_segment(
    data = bar_df,
    aes(x = start, xend = end, y = patient_id, yend = patient_id, color = response),
    linewidth = 3, lineend = "butt"
  ) +
  # 第二层：事件点
  geom_point(
    data = event_df,
    aes(x = time, y = patient_id, shape = event),
    size = 2.5, color = "black"
  ) +
  # 第三层：持续箭头（ongoing 患者）
  geom_segment(
    data = ongoing_df,
    aes(x = end, xend = end + 1, y = patient_id, yend = patient_id),
    arrow = arrow(length = unit(1.5, "mm"), type = "closed"),
    linewidth = 0.5, color = "black"
  ) +
  scale_color_manual(
    values = c("CR" = npg_pal[3], "PR" = npg_pal[2],
               "SD" = npg_pal[6], "PD" = npg_pal[1]),
    name = "Best Response"
  ) +
  scale_shape_manual(
    values = c("Progression" = 4, "Death" = 8, "Dose Reduction" = 6),
    name = "Event"
  ) +
  labs(x = "Time (months)", y = "Patient") +
  theme_insightlab() +
  theme(
    axis.text.y = element_text(size = 5),
    legend.position = "bottom",
    legend.box = "vertical"
  )

ggsave("swimmer_3layers.pdf", p, width = 140, height = 140, units = "mm")
```

### 关键参数

| 参数 | 说明 | 建议值 |
|------|------|--------|
| `linewidth` (条形) | 泳道条粗细 | 2-4 |
| 患者排序 | 按 end 时间排序 | `reorder(patient_id, end)` |
| `arrow(length)` | 箭头大小 | `unit(1.5, "mm")` |
| 事件 shape | 用不同形状区分事件 | 4=x, 8=*, 16=实心圆 |

---

## 2. 瀑布图 (Waterfall Plot)

### 适用场景

展示每位患者的肿瘤体积变化百分比（如 RECIST 评估中的最佳变化率），按变化量排序，附加阈值线（+20% PD, -30% PR）。

### 所需 R 包

`ggplot2`, `dplyr`

### 输入数据格式

| 列名 | 类型 | 说明 |
|------|------|------|
| patient_id | character | 患者编号 |
| change_pct | numeric | 肿瘤变化百分比（如 -45, +12） |
| response | character | RECIST 评估（CR/PR/SD/PD，可选） |

### R 代码模板

```r
library(ggplot2)
library(dplyr)

npg_pal <- ggsci::pal_npg("nrc")(10)

# --- 数据准备 ---
wf_df <- wf_df %>%
  arrange(change_pct) %>%
  mutate(
    patient_id = factor(patient_id, levels = patient_id),
    fill_color = case_when(
      change_pct <= -30 ~ "PR/CR",
      change_pct >= 20  ~ "PD",
      TRUE              ~ "SD"
    )
  )

# --- 绑图 ---
p <- ggplot(wf_df, aes(x = patient_id, y = change_pct, fill = fill_color)) +
  geom_col(width = 0.7) +
  geom_hline(yintercept = c(-30, 20), linetype = "dashed",
             color = "grey40", linewidth = 0.3) +
  scale_fill_manual(
    values = c("PR/CR" = npg_pal[2], "SD" = npg_pal[6],
               "PD" = npg_pal[1]),
    name = "Response"
  ) +
  annotate("text", x = nrow(wf_df), y = -30, label = "-30%",
           hjust = 1.1, vjust = -0.5, size = 2.5, color = "grey40") +
  annotate("text", x = nrow(wf_df), y = 20, label = "+20%",
           hjust = 1.1, vjust = -0.5, size = 2.5, color = "grey40") +
  labs(x = "Patient", y = "Best Change from Baseline (%)") +
  theme_insightlab() +
  theme(
    axis.text.x = element_blank(),
    axis.ticks.x = element_blank(),
    legend.position = "top"
  )

ggsave("waterfall_plot.pdf", p, width = 160, height = 80, units = "mm")
```

### 带分组注释条的瀑布图

```r
# 在条形图下方添加分组注释（如突变状态、治疗方案）
library(patchwork)

p_main <- p  # 上面的瀑布图主体

# 注释条
p_anno <- ggplot(wf_df, aes(x = patient_id, y = 1, fill = mutation_status)) +
  geom_tile() +
  scale_fill_manual(values = c("Mut" = npg_pal[1],
                                "WT" = npg_pal[6]),
                    name = "EGFR") +
  theme_void() +
  theme(
    legend.position = "right",
    legend.text = element_text(size = 7),
    legend.title = element_text(size = 8)
  )

# 拼图
p_combined <- p_main / p_anno + plot_layout(heights = c(10, 1))
ggsave("waterfall_annotated.pdf", p_combined, width = 160, height = 100, units = "mm")
```

### 关键参数

| 参数 | 说明 | 建议值 |
|------|------|--------|
| 排序方式 | 按 change_pct 升序 | `arrange(change_pct)` |
| RECIST 阈值 | PR: -30%, PD: +20% | 虚线标注 |
| X 轴标签 | 患者多时隐藏 | `element_blank()` |

---

## 3. ROC 曲线 (Receiver Operating Characteristic Curve)

### 适用场景

评估分类模型/生物标志物的诊断效能，展示灵敏度 vs (1-特异度) 的权衡关系，计算 AUC 值。支持单曲线和多曲线比较。

### 所需 R 包

`pROC`, `ggplot2`

### 输入数据格式

| 列名 | 类型 | 说明 |
|------|------|------|
| label | integer/factor | 真实标签（0/1） |
| score | numeric | 预测分数/生物标志物值 |

### R 代码模板（单条 ROC 曲线）

```r
library(pROC)
library(ggplot2)

npg_pal <- ggsci::pal_npg("nrc")(10)

# --- ROC 计算 ---
roc_obj <- roc(response = data$label, predictor = data$score,
               levels = c(0, 1), direction = "<")
auc_val <- auc(roc_obj)
ci_val  <- ci.auc(roc_obj)

# 提取坐标
roc_df <- data.frame(
  specificity = roc_obj$specificities,
  sensitivity = roc_obj$sensitivities
)

# --- 绑图 ---
p <- ggplot(roc_df, aes(x = 1 - specificity, y = sensitivity)) +
  geom_line(color = npg_pal[1], linewidth = 0.8) +
  geom_abline(slope = 1, intercept = 0, linetype = "dashed",
              color = "grey60", linewidth = 0.3) +
  annotate("text", x = 0.6, y = 0.2,
           label = sprintf("AUC = %.3f\n(95%% CI: %.3f-%.3f)",
                           auc_val, ci_val[1], ci_val[3]),
           size = 3, hjust = 0, color = "black") +
  labs(x = "1 - Specificity", y = "Sensitivity") +
  coord_equal() +
  theme_insightlab()

ggsave("roc_single.pdf", p, width = 89, height = 89, units = "mm")
```

### R 代码模板（多曲线比较）

```r
library(pROC)
library(ggplot2)
library(dplyr)

npg_pal <- ggsci::pal_npg("nrc")(10)

# --- 多模型 ROC ---
roc1 <- roc(data$label, data$score1)
roc2 <- roc(data$label, data$score2)
roc3 <- roc(data$label, data$score3)

# 提取并合并坐标
extract_roc <- function(roc_obj, name) {
  data.frame(
    specificity = roc_obj$specificities,
    sensitivity = roc_obj$sensitivities,
    Model = paste0(name, " (AUC=", sprintf("%.3f", auc(roc_obj)), ")")
  )
}

roc_all <- bind_rows(
  extract_roc(roc1, "Model A"),
  extract_roc(roc2, "Model B"),
  extract_roc(roc3, "Model C")
)

# --- 绑图 ---
p <- ggplot(roc_all, aes(x = 1 - specificity, y = sensitivity, color = Model)) +
  geom_line(linewidth = 0.7) +
  geom_abline(slope = 1, intercept = 0, linetype = "dashed",
              color = "grey60", linewidth = 0.3) +
  scale_color_manual(values = npg_pal[1:3]) +
  labs(x = "1 - Specificity", y = "Sensitivity", color = NULL) +
  coord_equal() +
  theme_insightlab() +
  theme(legend.position = c(0.65, 0.2))

# DeLong 检验比较两条 ROC 曲线
delong_test <- roc.test(roc1, roc2, method = "delong")
cat("DeLong test p-value:", delong_test$p.value, "\n")

ggsave("roc_comparison.pdf", p, width = 100, height = 100, units = "mm")
```

### 关键参数

| 参数 | 说明 | 建议值 |
|------|------|--------|
| `direction` | 标签方向 | `"<"` 或 `">"` 取决于分数含义 |
| `coord_equal` | 等比例坐标 | 必须（ROC 图标准） |
| `ci.auc` | AUC 置信区间 | 发表必须标注 |
| `roc.test` | DeLong 检验 | 多曲线比较时报告 P 值 |

---

## 4. 韦恩图 (Venn Diagram)

### 适用场景

展示 2-5 个基因集/样本集之间的重叠与差异关系。常用于比较不同分析方法、不同条件下的差异基因集合。

### 所需 R 包

`VennDiagram`, `grid`

### 输入数据格式

命名列表（list），每个元素为一个字符向量，代表一个集合。

### R 代码模板（2-4 组）

```r
library(VennDiagram)
library(grid)

npg_pal <- ggsci::pal_npg("nrc")(10)

# --- 输入数据 ---
gene_lists <- list(
  "Method A" = c("TP53", "EGFR", "KRAS", "BRAF", "PIK3CA", "MYC"),
  "Method B" = c("TP53", "KRAS", "PTEN", "RB1", "PIK3CA", "APC"),
  "Method C" = c("TP53", "EGFR", "PTEN", "CDKN2A", "MYC", "BRAF")
)

# --- 绑图 ---
# 关闭默认日志
futile.logger::flog.threshold(futile.logger::ERROR, name = "VennDiagramLogger")

venn <- venn.diagram(
  x = gene_lists,
  filename = NULL,         # 返回 grid 对象而非直接保存
  category.names = names(gene_lists),
  # 配色
  fill = npg_pal[1:3],
  alpha = 0.3,
  col = npg_pal[1:3],
  lwd = 1,
  # 数字样式
  cex = 1,
  fontfamily = "Arial",
  # 分类名样式
  cat.cex = 0.9,
  cat.fontfamily = "Arial",
  cat.col = npg_pal[1:3],
  cat.dist = c(0.05, 0.05, 0.05),
  # 边距
  margin = 0.05
)

# 导出
pdf("venn_diagram.pdf", width = 5, height = 5)
grid.draw(venn)
dev.off()
```

### R 代码模板（快速分析：画图 + 提取交集）

```r
library(VennDiagram)
library(grid)

npg_pal <- ggsci::pal_npg("nrc")(10)

# --- 快速函数 ---
venn_fast <- function(gene_lists, output_prefix = "venn") {

  n <- length(gene_lists)
  stopifnot(n >= 2 && n <= 5)

  futile.logger::flog.threshold(futile.logger::ERROR, name = "VennDiagramLogger")

  # 绘图
  venn <- venn.diagram(
    x = gene_lists,
    filename = NULL,
    category.names = names(gene_lists),
    fill = npg_pal[1:n],
    alpha = 0.3,
    col = npg_pal[1:n],
    lwd = 1,
    cex = 1,
    fontfamily = "Arial",
    cat.cex = 0.85,
    cat.fontfamily = "Arial",
    cat.col = npg_pal[1:n],
    margin = 0.05
  )

  pdf(paste0(output_prefix, ".pdf"), width = 5, height = 5)
  grid.draw(venn)
  dev.off()

  # 提取交集
  if (n == 2) {
    overlap <- intersect(gene_lists[[1]], gene_lists[[2]])
  } else {
    overlap <- Reduce(intersect, gene_lists)
  }

  # 提取各组特有元素
  unique_per_group <- lapply(seq_along(gene_lists), function(i) {
    setdiff(gene_lists[[i]], Reduce(union, gene_lists[-i]))
  })
  names(unique_per_group) <- names(gene_lists)

  # 输出结果
  result <- list(
    all_overlap = overlap,
    unique_elements = unique_per_group,
    counts = sapply(gene_lists, length)
  )

  cat("All-group overlap:", length(overlap), "genes\n")
  cat("Overlap genes:", paste(overlap, collapse = ", "), "\n")
  for (nm in names(unique_per_group)) {
    cat(sprintf("Unique to %s: %d genes\n", nm, length(unique_per_group[[nm]])))
  }

  return(result)
}

# --- 使用 ---
gene_lists <- list(
  "DESeq2"  = readLines("deseq2_genes.txt"),
  "edgeR"   = readLines("edger_genes.txt"),
  "limma"   = readLines("limma_genes.txt")
)

result <- venn_fast(gene_lists, output_prefix = "venn_deg_methods")
```

### UpSet 图替代方案（5 组以上推荐）

```r
# 当集合数 > 5 时，韦恩图不易阅读，推荐使用 UpSet 图
library(UpSetR)

# 将列表转为 0/1 矩阵
all_genes <- unique(unlist(gene_lists))
upset_mat <- data.frame(
  row.names = all_genes,
  sapply(gene_lists, function(x) as.integer(all_genes %in% x))
)

pdf("upset_plot.pdf", width = 8, height = 5)
upset(upset_mat, nsets = length(gene_lists),
      sets.bar.color = npg_pal[4],
      main.bar.color = npg_pal[1],
      text.scale = c(1.2, 1, 1, 1, 1.2, 1),
      order.by = "freq")
dev.off()
```

### 关键参数

| 参数 | 说明 | 建议值 |
|------|------|--------|
| `filename` | 设为 NULL 返回 grid 对象 | NULL（更灵活） |
| `alpha` | 填充透明度 | 0.25-0.4 |
| `cat.dist` | 分类名与圆的距离 | 0.03-0.08，需手动调整 |
| 集合数上限 | 韦恩图 <= 5 组 | > 5 组改用 UpSet 图 |

---

## 常见问题

1. **泳道图患者排序不理想**：默认按字母排序，必须手动设定 factor levels。推荐按治疗持续时间 `arrange(end)` 排序后设定 levels。

2. **瀑布图条形缺少颜色区分**：确保 `fill_color` 列正确映射。RECIST 标准：CR 完全缓解（-100%）、PR 部分缓解（<= -30%）、SD 疾病稳定、PD 疾病进展（>= +20%）。

3. **ROC 曲线方向反了（AUC < 0.5）**：检查 `direction` 参数。`direction = "<"` 表示 score 越大越可能为阳性；`direction = ">"` 则相反。也可设 `direction = "auto"`。

4. **韦恩图标签位置偏移**：调整 `cat.dist`（分类名到圆的距离）和 `cat.pos`（分类名角度）。两组和三组的默认位置参数不同，可能需要逐个调整。

5. **VennDiagram 产生日志文件干扰**：默认会在工作目录生成日志文件。使用 `futile.logger::flog.threshold(futile.logger::ERROR, name = "VennDiagramLogger")` 抑制。

6. **ROC 图不是正方形**：必须使用 `coord_equal()` 或导出时 width = height，确保 ROC 图为正方形，否则视觉上误导判读。
