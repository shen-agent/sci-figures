# 生存分析图表

当用户需要可视化生存分析结果（Kaplan-Meier 曲线、Cox 回归森林图、批量单因素分析）时，阅读本文件。涵盖 KM 曲线、森林图、批量 Cox 可视化三类临床研究核心图表。

## 涵盖图表类型

- Kaplan-Meier 曲线（分组 / 非分组）
- 森林图 (Forest Plot)
- 批量单因素 Cox 可视化

---

## 1. Kaplan-Meier 曲线

### 适用场景

展示不同分组（如高/低表达、突变/野生型、治疗/对照）的生存概率随时间的变化。是临床研究中最常见的生存分析图表。

### 所需 R 包

`survival`, `survminer`

### 输入数据格式

data.frame，至少包含以下列：

| 列名 | 类型 | 说明 |
|------|------|------|
| time | numeric | 生存时间（月/天） |
| status | integer | 事件状态（1=事件发生，0=删失） |
| group | character/factor | 分组变量（可选，无则画整体曲线） |

### R 代码模板（分组 KM 曲线）

```r
library(survival)
library(survminer)

npg_pal <- ggsci::pal_npg("nrc")(10)

# --- 构建生存对象 ---
surv_obj <- Surv(time = clinical$time, event = clinical$status)
fit <- survfit(surv_obj ~ group, data = clinical)

# --- 绑图 ---
p <- ggsurvplot(
  fit,
  data = clinical,
  palette = npg_pal[1:2],
  # 风险表
  risk.table = TRUE,
  risk.table.col = "strata",
  risk.table.y.text = FALSE,
  risk.table.height = 0.25,
  risk.table.fontsize = 3,
  # P 值
  pval = TRUE,
  pval.size = 3.5,
  pval.coord = c(0, 0.1),
  # 中位线
  surv.median.line = "hv",
  # 置信区间
  conf.int = FALSE,
  # 样式
  xlab = "Time (months)",
  ylab = "Overall Survival",
  legend.title = "",
  legend.labs = c("High", "Low"),
  ggtheme = theme_insightlab(),
  font.x = c(10, "plain"),
  font.y = c(10, "plain"),
  font.tickslab = c(8, "plain"),
  font.legend = c(9, "plain")
)

# --- 添加删失标记 ---
p$plot <- p$plot +
  geom_point(data = subset(fortify(fit), n.censor > 0),
             aes(x = time, y = surv), shape = "+", size = 2.5)

# --- 导出 ---
pdf("km_curve.pdf", width = 5, height = 4.5)
print(p)
dev.off()
```

### R 代码模板（非分组 / 整体 KM 曲线）

```r
library(survival)
library(survminer)

surv_obj <- Surv(time = clinical$time, event = clinical$status)
fit <- survfit(surv_obj ~ 1, data = clinical)

p <- ggsurvplot(
  fit,
  data = clinical,
  palette = npg_pal[4],
  risk.table = TRUE,
  risk.table.height = 0.2,
  conf.int = TRUE,
  conf.int.alpha = 0.15,
  surv.median.line = "hv",
  xlab = "Time (months)",
  ylab = "Survival Probability",
  ggtheme = theme_insightlab()
)

pdf("km_overall.pdf", width = 5, height = 4)
print(p)
dev.off()
```

### R 代码模板（多组比较 + pairwise P 值）

```r
library(survival)
library(survminer)

# 三组或更多分组比较
fit <- survfit(Surv(time, status) ~ stage, data = clinical)

p <- ggsurvplot(
  fit,
  data = clinical,
  palette = npg_pal[1:4],
  pval = TRUE,
  pval.method = TRUE,          # 显示检验方法
  risk.table = TRUE,
  risk.table.height = 0.3,
  xlab = "Time (months)",
  ylab = "Overall Survival",
  legend.title = "Stage",
  ggtheme = theme_insightlab()
)

# 两两比较 P 值
pairwise <- pairwise_survdiff(Surv(time, status) ~ stage,
                               data = clinical, p.adjust.method = "BH")
print(pairwise)

pdf("km_multigroup.pdf", width = 6, height = 5.5)
print(p)
dev.off()
```

### 关键参数

| 参数 | 说明 | 建议值 |
|------|------|--------|
| `risk.table` | 是否显示风险表 | TRUE（发表必备） |
| `risk.table.height` | 风险表高度占比 | 0.2-0.3 |
| `pval` | 显示 log-rank P 值 | TRUE |
| `surv.median.line` | 中位生存时间辅助线 | "hv"（水平+垂直） |
| `conf.int` | 置信区间阴影 | 两组时通常 FALSE |
| `break.time.by` | X 轴时间间隔 | 根据随访时间设定 |

---

## 2. 森林图 (Forest Plot)

### 适用场景

展示多变量 Cox 回归或亚组分析的 HR (Hazard Ratio) / OR (Odds Ratio) 及其 95% CI，以直观比较各因素的效应大小和方向。

### 所需 R 包

方案 A: `forestplot` 包（经典表格+图形组合）
方案 B: `ggplot2` 手动绑制（完全可定制）

### 输入数据格式

data.frame，包含：

| 列名 | 类型 | 说明 |
|------|------|------|
| variable | character | 变量名 |
| HR | numeric | 风险比 |
| lower | numeric | 95% CI 下界 |
| upper | numeric | 95% CI 上界 |
| pvalue | numeric | P 值 |

### R 代码模板 (forestplot 包)

```r
library(forestplot)

npg_pal <- ggsci::pal_npg("nrc")(10)

# --- 数据准备 ---
# 从 Cox 回归结果构建
cox_model <- coxph(Surv(time, status) ~ age + sex + stage + grade, data = clinical)
cox_summary <- summary(cox_model)

forest_df <- data.frame(
  variable = rownames(cox_summary$conf.int),
  HR       = cox_summary$conf.int[, 1],
  lower    = cox_summary$conf.int[, 3],
  upper    = cox_summary$conf.int[, 4],
  pvalue   = cox_summary$coefficients[, 5]
)

# 构建表格文本
tabletext <- cbind(
  c("Variable", forest_df$variable),
  c("HR (95% CI)", sprintf("%.2f (%.2f-%.2f)", forest_df$HR, forest_df$lower, forest_df$upper)),
  c("P value", ifelse(forest_df$pvalue < 0.001, "<0.001", sprintf("%.3f", forest_df$pvalue)))
)

# --- 绑图 ---
pdf("forest_plot.pdf", width = 8, height = 4)
forestplot(
  labeltext = tabletext,
  mean      = c(NA, forest_df$HR),
  lower     = c(NA, forest_df$lower),
  upper     = c(NA, forest_df$upper),
  is.summary = c(TRUE, rep(FALSE, nrow(forest_df))),
  zero = 1,
  col = fpColors(
    box = npg_pal[1],
    line = npg_pal[4],
    summary = npg_pal[3],
    zero = "grey40"
  ),
  boxsize = 0.2,
  lineheight = unit(8, "mm"),
  colgap = unit(3, "mm"),
  xlog = TRUE,
  xlab = "Hazard Ratio",
  txt_gp = fpTxtGp(
    label  = gpar(fontfamily = "Arial", cex = 0.8),
    ticks  = gpar(fontfamily = "Arial", cex = 0.7),
    xlab   = gpar(fontfamily = "Arial", cex = 0.9)
  )
)
dev.off()
```

### R 代码模板 (ggplot2 手绘森林图)

```r
library(ggplot2)
library(dplyr)

npg_pal <- ggsci::pal_npg("nrc")(10)

# forest_df 需包含 variable, HR, lower, upper, pvalue 列
forest_df <- forest_df %>%
  mutate(
    variable = factor(variable, levels = rev(variable)),
    sig = ifelse(pvalue < 0.05, "Significant", "Not Significant")
  )

p <- ggplot(forest_df, aes(x = HR, y = variable)) +
  geom_vline(xintercept = 1, linetype = "dashed", color = "grey40", linewidth = 0.3) +
  geom_errorbarh(aes(xmin = lower, xmax = upper), height = 0.2,
                 color = npg_pal[4], linewidth = 0.4) +
  geom_point(aes(color = sig), size = 2.5) +
  scale_color_manual(values = c("Significant" = npg_pal[1],
                                "Not Significant" = "grey60")) +
  scale_x_log10() +
  labs(x = "Hazard Ratio (95% CI)", y = NULL, color = NULL) +
  theme_insightlab() +
  theme(legend.position = "bottom")

ggsave("forest_ggplot.pdf", p, width = 120, height = 80, units = "mm")
```

### 关键参数

| 参数 | 说明 | 建议值 |
|------|------|--------|
| `zero = 1` | 无效线位置 | HR=1, OR=1 |
| `xlog = TRUE` | X 轴对数刻度 | TRUE（HR/OR 图必须） |
| `boxsize` | 效应值方块大小 | 0.15-0.25 |
| `is.summary` | 标题行标记 | 第一行为 TRUE |

---

## 3. 批量单因素 Cox 可视化

### 适用场景

对多个基因/变量分别进行单因素 Cox 回归，汇总结果并以森林图展示。常用于筛选 prognostic markers。

### 所需 R 包

`survival`, `ggplot2`, `dplyr`, `purrr`（可选）

### 输入数据格式

- 临床数据：包含 `time`, `status` 列
- 表达矩阵或多变量列：每列为一个待检验的变量

### R 代码模板

```r
library(survival)
library(ggplot2)
library(dplyr)

npg_pal <- ggsci::pal_npg("nrc")(10)

# --- 批量单因素 Cox ---
# genes: 待检验的变量名向量
# clinical: 数据框，包含 time, status, 以及 genes 中的各列
genes <- c("EGFR", "TP53", "KRAS", "BRAF", "PIK3CA", "PTEN",
           "MYC", "RB1", "CDKN2A", "APC")

cox_batch <- function(gene, data) {
  formula <- as.formula(paste0("Surv(time, status) ~ ", gene))
  fit <- tryCatch(
    coxph(formula, data = data),
    error = function(e) NULL
  )
  if (is.null(fit)) return(NULL)
  s <- summary(fit)
  data.frame(
    gene    = gene,
    HR      = s$conf.int[1, 1],
    lower   = s$conf.int[1, 3],
    upper   = s$conf.int[1, 4],
    pvalue  = s$coefficients[1, 5],
    stringsAsFactors = FALSE
  )
}

results <- do.call(rbind, lapply(genes, cox_batch, data = clinical))
results <- results %>%
  mutate(
    padj = p.adjust(pvalue, method = "BH"),
    sig  = ifelse(padj < 0.05, "Significant", "NS"),
    gene = factor(gene, levels = rev(gene[order(HR)]))  # 按 HR 排序
  )

# --- 森林图展示 ---
p <- ggplot(results, aes(x = HR, y = gene)) +
  geom_vline(xintercept = 1, linetype = "dashed", color = "grey40", linewidth = 0.3) +
  geom_errorbarh(aes(xmin = lower, xmax = upper), height = 0.25,
                 color = npg_pal[4], linewidth = 0.4) +
  geom_point(aes(fill = sig), shape = 22, size = 3, color = "black", stroke = 0.3) +
  scale_fill_manual(values = c("Significant" = npg_pal[1], "NS" = "grey80")) +
  # 右侧标注 HR 和 P 值
  geom_text(aes(x = max(upper) * 1.3,
                label = sprintf("%.2f (%.2f-%.2f)", HR, lower, upper)),
            size = 2.3, hjust = 0) +
  geom_text(aes(x = max(upper) * 2.5,
                label = ifelse(padj < 0.001, "<0.001", sprintf("%.3f", padj))),
            size = 2.3, hjust = 0) +
  scale_x_log10() +
  coord_cartesian(xlim = c(min(results$lower) * 0.8, max(results$upper) * 4)) +
  labs(x = "Hazard Ratio (95% CI)", y = NULL, fill = NULL,
       title = "Univariate Cox Regression") +
  theme_insightlab() +
  theme(legend.position = "bottom",
        axis.text.y = element_text(face = "italic"))

ggsave("cox_batch_forest.pdf", p, width = 140, height = 100, units = "mm")
```

### 表格输出辅助

```r
# 如需同时输出表格
library(writexl)

output_table <- results %>%
  select(gene, HR, lower, upper, pvalue, padj) %>%
  mutate(across(where(is.numeric), ~ round(.x, 4)))

write_xlsx(output_table, "cox_univariate_results.xlsx")
```

### 关键参数

| 参数 | 说明 | 建议值 |
|------|------|--------|
| `p.adjust` 方法 | 多重检验校正 | "BH"（Benjamini-Hochberg） |
| `tryCatch` | 容错处理 | 必要，部分基因可能不收敛 |
| 排序方式 | 基因按 HR 或 P 值排序 | 按 HR 排序最直观 |
| `coord_cartesian(xlim)` | 留出文字标注空间 | 上限设为 CI 上界的 3-4 倍 |

---

## 常见问题

1. **KM 曲线 P 值显示为 "p = 0"**：padj 极小时 survminer 默认显示 "p = 0"，使用 `pval = paste0("p ", format.pval(surv_pvalue(fit)$pval, digits = 2))` 自定义格式。

2. **风险表与曲线对不齐**：确保 `risk.table.height` 参数合理（0.2-0.3），用 `pdf()` 而非 `ggsave()` 导出 survminer 图表效果更好。

3. **森林图 X 轴不对称**：HR 图必须使用对数坐标（`xlog = TRUE` 或 `scale_x_log10()`），否则视觉上会误导效应方向。

4. **批量 Cox 部分基因报错**：低变异基因可能导致模型不收敛，`tryCatch` 包裹并过滤 NULL 结果即可。

5. **survminer 导出时风险表被截断**：不要使用 `ggsave()`，改用 `pdf()`/`png()` + `print(p)` + `dev.off()` 组合导出。

6. **分组变量含 NA 导致曲线缺失**：在拟合前 `filter(!is.na(group))` 去除缺失值。
