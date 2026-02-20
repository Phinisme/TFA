# ITC 分析 Workflow
*Inter-Trial Coherence 分析指南*

---

## 目录

1. [什么是 ITC](#1-什么是-itc)
2. [数据类型](#2-数据类型)
3. [计算方式](#3-计算方式)
4. [统计方法](#4-统计方法)
5. [与 ERP P2 成分的关系](#5-与-erp-p2-成分的关系)

---

## 1. 什么是 ITC

ITC（Inter-Trial Coherence）衡量的是 alpha 相位在 trial 间的一致性，取值范围为 **[0, 1]**：

- **0**：完全随机，每个 trial 的相位不同
- **1**：完全同步，每个 trial 的相位相同

### ITC 与 Power 的核心区别

| 属性 | Power | ITC |
|------|-------|-----|
| 衡量什么 | 振荡能量 | 相位锁定程度 |
| 受什么影响 | ERD/ERS | Stimulus-locked 节律 |
| 值域 | 任意实数（dB） | 0–1 |
| Baseline correction | 需要 | 通常不需要（本身是归一化的） |
| 计算方式 | `output='power'` | `return_itc=True` |

---

## 2. 数据类型

ITC 必须使用 **single-trial epochs**，**不能**使用 averaged 数据，因为需要跨 trial 计算相位一致性。

---

## 3. 计算方式

```python
# ITC 必须 average=True，return_itc=True
tfr_power, itc = epochs_cond.compute_tfr(
    method='morlet',
    freqs=freqs,
    n_cycles=n_cycles,
    return_itc=True,   # <- 同时返回 ITC
    average=True,      # <- ITC 只能在 averaged 模式下计算
    output='power'
)
# itc.data shape: (ch, freq, time)，值域 0-1
```

---

## 4. 统计方法

ITC 的统计与 power 不同，因为值域是 **[0, 1]**，不能直接使用 *t* 检验（不符合正态分布假设）。常用方法如下：

- **Rayleigh test**：检验相位是否显著偏离随机分布，适合单条件分析。
- **条件间对比**：直接计算 ITC_ext − ITC_int，再配合 permutation test 进行统计检验。
- **Bootstrap**：对 trial 随机重采样，建立 null distribution。

对于条件间（ext vs. int）的 ITC 差异分析，最直接的方法如下：

```python
# 条件间 ITC 差异 + cluster permutation
# itc_ext - itc_int -> (n_subjects, ch, freq, time)
# 用 spatio_temporal_cluster_1samp_test
# 与 power 分析方式相同
```

---

## 5. 与 ERP P2 成分的关系

P2 成分大约出现在 **150–250 ms**，对应 theta/alpha 频段的 phase-locking。若 P2 的潜伏期存在条件间差异，则 ITC 在对应的时间–频率点上也应有相应差异。

**分析建议：** 重点关注以下时间–频率窗口：

$$t \in [0.1,\ 0.3]\ \text{s}, \quad f \in [8,\ 14]\ \text{Hz}$$
