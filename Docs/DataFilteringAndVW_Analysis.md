# Data Filtering & Universe Selection — 技术分析报告

本文档基于对仓库代码的深入检索，详细记录了构建投资组合（尤其是市值加权 VW 组合）之前，底层股票数据所经历的预处理与过滤步骤。

---

## 目录

1. [股票池清洗：交易所（EXCHCD）与股票类型（SHRCD）过滤](#1-股票池清洗exchcd-与-shrcd-过滤)
2. [极值与垃圾股剔除：仙股与微盘股](#2-极值与垃圾股剔除仙股与微盘股)
3. [行业剔除：金融与公用事业](#3-行业剔除金融与公用事业)
4. [VW（市值加权）的底层逻辑](#4-vw市值加权的底层逻辑)
5. [替代组合变体（Alternative Portfolios）的生成逻辑](#5-替代组合变体alternative-portfolios的生成逻辑)

---

## 1. 股票池清洗：EXCHCD 与 SHRCD 过滤

### 1.1 信号生成阶段（Python）

在 `SignalMasterTable.py`（信号生成的"骨架表"）中，**全局性地**对 CRSP 月度数据进行了交易所和股票类型过滤：

**文件**: `Signals/pyCode/SignalMasterTable.py`，第 44–49 行

```python
# Screen on Stock market information: common stocks and major exchanges
# TBC: remove and use this filter as default in SignalDoc.csv

# keep if (shrcd == 10 | shrcd == 11 | shrcd == 12) & (exchcd == 1 | exchcd == 2 | exchcd == 3)
df = df[(df['shrcd'].isin([10, 11, 12])) & (df['exchcd'].isin([1, 2, 3]))].copy()
```

**结论**：
- **SHRCD 过滤**：只保留 10、11、12，即美国普通股（common stocks）。剔除了优先股、ADR、封闭式基金等。
- **EXCHCD 过滤**：只保留 1 (NYSE)、2 (AMEX)、3 (NASDAQ)，剔除了其他交易所（如 ARCA 等）。
- 此过滤对所有下游 ~300 个信号生效，因为所有 Predictor 脚本都从 `SignalMasterTable.parquet` 加载数据。

### 1.2 组合构建阶段（R）

在 Fama-French 2×3 风格组合中，**再次**显式过滤了 EXCHCD 和 SHRCD：

**文件**: `Portfolios/Code/32_Predictor2x3Ports.R`，第 42–57 行

```r
# For NYSE subset, compute signal quantiles for high and low
nysebreaks = signaljune %>%
  filter(exchcd == 1, shrcd %in% c(10, 11)) %>%   # 仅 NYSE 且 shrcd 10/11（比信号阶段更严格，排除了 shrcd 12）
  group_by(yyyymm) %>%
  summarise(
    qsignal_l = quantile(signal, 0.3, na.rm = T)
    , qsignal_h = quantile(signal, 0.7, na.rm = T)
    , qme_mid = quantile(me, 0.5, na.rm = T)
  )

# Only exchcd in (1,2,3), shrcd in (10,11), (FF1993 p8-9)
port6 = signaljune %>%
  filter(
    exchcd %in% c(1, 2, 3)            # NYSE, AMEX, NASDAQ
    , shrcd %in% c(10, 11)            # 普通股
  ) %>% ...
```

**注意差异**：信号阶段保留了 `shrcd == 12`，而 2×3 组合构建阶段只保留了 `shrcd %in% c(10, 11)`，更为严格。

### 1.3 通用组合函数中的过滤

在 `01_PortfolioFunction.R` 中，分位点断点的计算可选地仅使用 NYSE 股票：

**文件**: `Portfolios/Code/01_PortfolioFunction.R`，第 124–128 行

```r
if (q_filt=='NYSE'){
  tempbreak = tempbreak %>% filter(exchcd == 1)   # 仅 NYSE 用于分位点断点
}
```

此外，`SignalDoc.csv` 中每个信号的 `Filter` 列可以指定额外过滤条件。数据显示以下过滤模式被广泛使用：

| Filter 值 | 含义 |
|---|---|
| `shrcd<=11` | 普通股 |
| `shrcd%in%c(10,11)` | 普通股（更严格） |
| `exchcd%in%c(1,2,3)` | 三大交易所 |
| `exchcd==1` | 仅 NYSE |
| `abs(prc)>1` | 剔除价格 ≤ $1 |
| `abs(prc)>5` | 剔除价格 ≤ $5 |
| `me>me_nyse20` | 剔除 NYSE 第 20 百分位以下微盘股 |

---

## 2. 极值与垃圾股剔除：仙股与微盘股

### 2.1 仙股（Penny Stock）剔除

代码中**没有全局性的统一仙股剔除**。相反，仙股过滤是**按信号**通过 `SignalDoc.csv` 的 `Filter` 列配置的。

**常见配置**（来自 `SignalDoc.csv` 的 `Filter` 列）：
- `abs(prc) > 1`：剔除价格 ≤ $1 的股票
- `abs(prc) > 5`：剔除价格 ≤ $5 的股票

**示例：FirmAgeMom.py** 中在信号层面直接剔除了低价股：

**文件**: `Signals/pyCode/Predictors/FirmAgeMom.py`，第 40–44 行

```python
# Filter: exclude stocks with price < $5 or less than 12 months of history
price_condition = (df["prc"].abs() >= 5) | df["prc"].isna()
df = df[price_condition & (df["age"] >= 12)].copy()
```

**组合构建阶段的过滤机制**（`01_PortfolioFunction.R`，第 46–51 行）：

```r
## apply filters and sign
if (!is.na(filterstr)){
  evalme = paste0('signal = signal %>% filter('
                  , filterstr
                  , ')')
  eval(parse(text=evalme))
}
```

该机制动态执行 `SignalDoc.csv` 中的 `filterstr`，因此价格过滤只在指定了相关 Filter 的信号中生效。

### 2.2 微盘股（Micro-cap）剔除

代码中**没有全局性的微盘股剔除**。但提供了以下机制：

1. **NYSE 市值百分位断点**（`Portfolios/Code/11_ProcessCRSP.R`，第 78–86 行）：

```r
tempcut <- crspminfo %>%
  filter(exchcd == 1) %>%                           # 仅 NYSE
  group_by(yyyymm) %>%
  summarize(
    me_nyse10 = quantile(me, probs = 0.1, na.rm = T),  # NYSE 第 10 百分位
    me_nyse20 = quantile(me, probs = 0.2, na.rm = T)   # NYSE 第 20 百分位
  )
```

2. **按信号的市值过滤**（通过 `SignalDoc.csv` 的 `Filter` 列）：
   - `me > me_nyse10`：剔除 NYSE 第 10 百分位以下
   - `me > me_nyse20`：剔除 NYSE 第 20 百分位以下

3. **市值缺失的股票**：在多个信号中，市值缺失（`mve_c` 为 NaN）会导致信号值被设为 NaN，从而在组合构建时被排除。例如：

**文件**: `Signals/pyCode/Predictors/CBOperProf.py`，第 136–144 行

```python
exclusion_mask = (
    (df["shrcd"] > 11)
    | df["mve_c"].isna()          # 市值缺失 → 排除
    | df["BM"].isna()
    | df["at"].isna()
    | ((df["sicCRSP"] >= 6000) & (df["sicCRSP"] < 7000))
)
df.loc[exclusion_mask, "CBOperProf"] = np.nan
```

---

## 3. 行业剔除：金融与公用事业

### 3.1 金融行业（SIC 6000–6999）

代码中**没有全局性的金融行业剔除**。金融行业的剔除在**特定信号的脚本中**根据原始论文的方法论单独实施。

**示例 1 — CBOperProf.py**（Cash-based Operating Profitability）：

```python
# Signals/pyCode/Predictors/CBOperProf.py, 第 141 行
| ((df["sicCRSP"] >= 6000) & (df["sicCRSP"] < 7000))   # 剔除金融业
```

**示例 2 — NetDebtPrice.py**：

```python
# Signals/pyCode/Predictors/NetDebtPrice.py, 第 73 行
df.loc[(df["sic"] >= 6000) & (df["sic"] <= 6999), "NetDebtPrice"] = np.nan
```

**示例 3 — GP.py**（Gross Profitability）：

```python
# Signals/pyCode/Predictors/GP.py
df = df[(df["sic"] < 6000) | (df["sic"] >= 7000)]
```

**示例 4 — BrandInvest.py**：

```python
# Signals/pyCode/Predictors/BrandInvest.py, 第 114-115 行
# Drop utilities (4900-4999) and financials (6000-6999)
df = df[~((df["sic_numeric"] >= 6000) & (df["sic_numeric"] <= 6999))]
```

### 3.2 公用事业（SIC 4900–4999）

同样，公用事业的剔除也是**按信号**、非全局性的：

**示例 — BrandInvest.py**：

```python
# Signals/pyCode/Predictors/BrandInvest.py
df = df[~((df["sic_numeric"] >= 4900) & (df["sic_numeric"] <= 4999))]
```

**总结**：行业剔除策略遵循"忠于原始论文"的原则——只有当原始论文明确排除了金融或公用事业时，相应的信号脚本才会实施剔除，而非全局统一。

---

## 4. VW（市值加权）的底层逻辑

### 4.1 核心结论

**是的，代码严格使用了 t-1 期的市值（Lagged Market Equity, `melag`）作为 t 期收益率的权重。**

### 4.2 `melag` 的构建

**文件**: `Portfolios/Code/11_ProcessCRSP.R`，第 47–63 行

```r
# 第 52 行：计算当月市值 = |价格| × 流通股数
me = abs(prc) * shrout

# 第 57-63 行：创建滞后市值 melag（上月的 me）
templag <- crspm %>%
  select(permno, yyyymm, me) %>%
  mutate(
    yyyymm = yyyymm + 1,                                              # 月份 +1
    yyyymm = if_else(yyyymm %% 100 == 13, yyyymm + 100 - 12, yyyymm) # 处理 12 月→1 月的跨年
  ) %>%
  transmute(permno, yyyymm, melag = me)   # melag = 上月的 me
```

**机制**：将当月的 `me` 赋值给下个月的 `melag`，即 `melag_t = me_{t-1}`。通过将 `yyyymm` 加 1（并处理 12→1 月的跨年），实现了时间上的前移，使得 `melag` 在 t 月对应的正好是 t-1 月的市值。

### 4.3 `melag` 合并到收益数据

**文件**: `Portfolios/Code/11_ProcessCRSP.R`，第 67–71 行

```r
crspmret <- crspm %>%
  select(permno, date, yyyymm, ret) %>%
  filter(!is.na(ret)) %>%
  left_join(templag, by = c("permno", "yyyymm")) %>%   # 将 melag 匹配到收益数据
  arrange(permno, yyyymm)
```

### 4.4 VW 权重赋值（核心代码）

**文件**: `Portfolios/Code/01_PortfolioFunction.R`，第 267–273 行

```r
## stock weights
# equal vs value-weighting
if (sweight == 'VW'){
  crspret$weight = crspret$melag       # ★ VW 权重 = 滞后市值 melag
} else {
  crspret$weight = 1                   # EW 权重 = 1（等权）
}
```

**这是最关键的一行代码**：`crspret$weight = crspret$melag`。它直接将滞后市值（t-1 期）设为当期（t 期）的权重。

### 4.5 VW 收益率的加权平均计算

**文件**: `Portfolios/Code/01_PortfolioFunction.R`，第 281–290 行

```r
port = crspret[
  !is.na(port) & !is.na(ret) & !is.na(weight)        # 排除缺失值
][
  , .(
    ret = weighted.mean(ret, weight)                   # ★ 组合收益 = 加权平均(ret, melag)
    , signallag = weighted.mean(signallag, weight)
    , Nlong = .N
  )
  , by = list(port, date)                              # 按组合×日期分组
]
```

### 4.6 Fama-French 2×3 组合中的 VW

**文件**: `Portfolios/Code/32_Predictor2x3Ports.R`，第 92–98 行

```r
# Find value-weighted returns and signal by port6_lag month
group_by(port6_lag, date) %>%
summarize(
  ret_vw = weighted.mean(ret, melag, na.rm = TRUE)    # ★ VW 收益
  , signallag = weighted.mean(signal_lag, melag, na.rm = TRUE)
  , n_firms = n()
)
```

### 4.7 每日数据的 VW 处理

对于日频数据，同样使用了月度 `melag`，并额外引入了 `passgain`（月内被动收益）来调整权重：

**文件**: `Portfolios/Code/11_ProcessCRSP.R`，第 130–145 行

```r
# merge on last month's lagged me for fast (monthly-rebalanced) value-weighting
templag = read_fst(
  paste0(pathProject,'Portfolios/Data/Intermediate/crspminfo.fst')
  , columns = c('permno','yyyymm','me')
) %>%
  setDT() %>%
  mutate(
    yyyymm = yyyymm + 1
    , yyyymm = if_else(yyyymm %% 100 == 13, yyyymm+100-12,yyyymm)
  ) %>%
  transmute(permno, yyyymm, melag = me)
```

**文件**: `Portfolios/Code/01_PortfolioFunction.R`，第 274–278 行

```r
# adjustments for passive gains
# (this is only used for the daily ports right now)
if (passive_gain){
  crspret$weight = crspret$weight * crspret$passgain   # melag × 月内累计收益
}
```

### 4.8 退市收益处理

在计算 `melag` 之前，代码还处理了退市收益（delisting return），遵循 Johnson & Zhao (2007) 和 Shumway & Warther (1999) 的方法：

**文件**: `Portfolios/Code/11_ProcessCRSP.R`，第 13–45 行

```r
# 对于 NYSE/AMEX (exchcd == 1 or 2) 的退市：缺失退市收益默认为 -35%
# 对于 NASDAQ (exchcd == 3) 的退市：缺失退市收益默认为 -55%
# 退市收益被合并到月度收益中：ret = (1+ret)*(1+dlret)-1
```

---

## 5. 替代组合变体（Alternative Portfolios）的生成逻辑

除了基线组合（`PredictorPortsFull.csv`，由 `20_PredictorPorts.R` 生成）之外，`30_PredictorAltPorts.R` 脚本统一生成了一系列**替代组合变体**，涵盖不同的分位排序方式、加权方式和流动性筛选。

### 5.1 核心机制

所有变体都通过同一个 `loop_over_strategies()` 函数生成（定义在 `01_PortfolioFunction.R`），只是传入不同的参数覆盖了 `SignalDoc.csv` 中的默认配置。

**文件**: `Portfolios/Code/30_PredictorAltPorts.R`

关键参数：
- **`q_cut`**：分位排序的切分比例。`q_cut = 0.1` → 十分位（Decile，10 组）；`q_cut = 0.2` → 五分位（Quintile，5 组）
- **`sweight`**：股票权重方式。`'VW'` → 市值加权；`'EW'` → 等权；省略则使用 `SignalDoc.csv` 中原始论文的设定（"OP stock weighting"）
- **`strategylistcts`**：仅包含 `Cat.Form == 'continuous'`（连续型信号）的信号列表，因为分位排序仅适用于连续型信号

### 5.2 十分位（Decile）组合

**文件**: `Portfolios/Code/30_PredictorAltPorts.R`，第 99–128 行

```r
strategylistcts = strategylist0 %>% filter(Cat.Form == 'continuous')

# Deciles: 使用原始论文的加权方式（OP stock weighting）
port <- loop_over_strategies(
  strategylistcts %>% mutate(q_cut = 0.1)
)
writestandard(port, pathDataPortfolios, "PredictorAltPorts_Deciles.csv")

# DecilesVW: 强制使用市值加权
port <- loop_over_strategies(
  strategylistcts %>% mutate(q_cut = 0.1, sweight = 'VW')
)
writestandard(port, pathDataPortfolios, "PredictorAltPorts_DecilesVW.csv")

# DecilesEW: 强制使用等权加权
port <- loop_over_strategies(
  strategylistcts %>% mutate(q_cut = 0.1, sweight = 'EW')
)
writestandard(port, pathDataPortfolios, "PredictorAltPorts_DecilesEW.csv")
```

### 5.3 五分位（Quintile）组合 — `PredictorAltPorts_QuintilesVW` 的来源

**文件**: `Portfolios/Code/30_PredictorAltPorts.R`，第 132–153 行

```r
## QUINTILE SORTS

# Quintiles: 使用原始论文的加权方式
port <- loop_over_strategies(
  strategylistcts %>% mutate(q_cut = 0.2)
)
writestandard(port, pathDataPortfolios, "PredictorAltPorts_Quintiles.csv")

# QuintilesVW: 强制使用市值加权 ★
port <- loop_over_strategies(
  strategylistcts %>% mutate(q_cut = 0.2, sweight = 'VW')
)
writestandard(port, pathDataPortfolios, "PredictorAltPorts_QuintilesVW.csv")

# QuintilesEW: 强制使用等权加权
port <- loop_over_strategies(
  strategylistcts %>% mutate(q_cut = 0.2, sweight = 'EW')
)
writestandard(port, pathDataPortfolios, "PredictorAltPorts_QuintilesEW.csv")
```

**`PredictorAltPorts_QuintilesVW`** 的含义：与 `PredictorAltPorts_Deciles` 和 `PredictorAltPorts_DecilesVW` 类似，但使用**五分位**排序（`q_cut = 0.2`，将股票分为 5 组），并**强制所有信号使用市值加权**（`sweight = 'VW'`），覆盖了 `SignalDoc.csv` 中各信号原始论文的加权方式。

### 5.4 其他替代组合

`30_PredictorAltPorts.R` 还生成了以下变体：

| 输出文件名 | 变化维度 | 代码行 |
|---|---|---|
| `PredictorAltPorts_HoldPer_{1,3,6,12}.csv` | 不同持有期（1/3/6/12 个月） | 第 20–40 行 |
| `PredictorAltPorts_LiqScreen_ME_gt_NYSE20pct.csv` | 剔除 NYSE 第 20 百分位以下微盘股 | 第 50–57 行 |
| `PredictorAltPorts_LiqScreen_Price_gt_5.csv` | 剔除价格 ≤ $5 的股票 | 第 62–68 行 |
| `PredictorAltPorts_LiqScreen_NYSEonly.csv` | 仅保留 NYSE 股票 | 第 74–81 行 |
| `PredictorAltPorts_LiqScreen_VWforce.csv` | 强制所有信号使用 VW（不改变分位数） | 第 87–94 行 |
| `PredictorAltPorts_Deciles.csv` | 十分位，原始论文加权 | 第 104–110 行 |
| `PredictorAltPorts_DecilesVW.csv` | 十分位，强制 VW | 第 113–119 行 |
| `PredictorAltPorts_DecilesEW.csv` | 十分位，强制 EW | 第 122–128 行 |
| `PredictorAltPorts_Quintiles.csv` | 五分位，原始论文加权 | 第 135–139 行 |
| **`PredictorAltPorts_QuintilesVW.csv`** | **五分位，强制 VW** | **第 142–146 行** |
| `PredictorAltPorts_QuintilesEW.csv` | 五分位，强制 EW | 第 149–153 行 |

此外，`32_Predictor2x3Ports.R` 还生成了 Fama-French 1993 风格的 2×3 组合（`PredictorAltPorts_FF93style.csv`）。

### 5.5 数据打包与输出

这些替代组合在 `Shipping/Code/2_pack_portfolios_and_results.r` 中被打包输出：

```r
write_indiv('PredictorAltPorts_LiqScreen_VWforce.csv', 'Original_CutsVW')
write_indiv('PredictorAltPorts_Deciles.csv', 'Cts_Deciles')
write_indiv('PredictorAltPorts_Quintiles.csv', 'Cts_Quintiles')
write_indiv('PredictorAltPorts_DecilesVW.csv', 'Cts_DecilesVW')
write_indiv('PredictorAltPorts_QuintilesVW.csv', 'Cts_QuintilesVW')
```

---

## 总结

| 维度 | 结论 |
|---|---|
| **EXCHCD 过滤** | ✅ 全局过滤为 NYSE(1), AMEX(2), NASDAQ(3) |
| **SHRCD 过滤** | ✅ 全局过滤为普通股 (10, 11, 12)；2×3 组合中更严格 (10, 11) |
| **仙股剔除** | ⚠️ 非全局，按信号配置（通过 `SignalDoc.csv` 的 `Filter` 列） |
| **微盘股剔除** | ⚠️ 非全局，部分信号通过 NYSE 百分位断点过滤 |
| **市值缺失** | ✅ 在 VW 计算中，`melag` 为 NaN 的股票被自动排除 |
| **金融行业剔除** | ⚠️ 非全局，特定信号按原始论文要求剔除 (SIC 6000–6999) |
| **公用事业剔除** | ⚠️ 非全局，仅少数信号剔除 (SIC 4900–4999) |
| **VW 权重** | ✅ **严格使用 t-1 期市值 (`melag`) 作为 t 期的权重** |
| **退市收益** | ✅ 已处理，NYSE/AMEX 默认 -35%，NASDAQ 默认 -55% |
