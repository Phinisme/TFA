# EEG TFR Cluster Permutation — Diagnostic Workflow

> **目的**：当 cluster permutation test 没有显著结果时，系统性地排查原因。  
> **环境**：Python + MNE，Jupyter Notebook  
> **适用场景**：基于 TFR（时频分析）数据的 spatio-temporal cluster permutation test

---

## 概览：诊断层级

```
Level 1: 统计层面      → Effect size & 被试间符号一致性
Level 2: 数据层面      → t-map 可视化、条件间差异
Level 3: 上游处理层面  → Baseline correction、数据存储格式
Level 4: 实验设计层面  → Contrast逻辑是否符合假设
```

遵循**从下游到上游**的顺序：先确认统计结果是否真的为空，再排查数据问题。

---

## Step 1：检查效应量与被试间一致性

**目的**：判断"没有显著性"是真的没有效应，还是统计功效不足。

```python
from scipy import stats
import numpy as np

for contrast_name in ['ext_vs_int', 'rep_vs_swi', 'interaction']:
    X = np.array(data_contrasts[contrast_name])  # (n_subjects, freq, time, ch)
    
    # 对所有维度取均值，得到每个被试的标量效应
    subject_means = X.mean(axis=(1, 2, 3))  # (n_subjects,)
    
    t_stat, p_val = stats.ttest_1samp(subject_means, 0)
    cohen_d = subject_means.mean() / subject_means.std()
    
    print(f"{contrast_name}:")
    print(f"  Mean: {subject_means.mean():.6f}  Std: {subject_means.std():.6f}")
    print(f"  t = {t_stat:.3f}, p = {p_val:.4f}")
    print(f"  Cohen's d = {cohen_d:.3f}")
    print()
```

**判读标准**：

| Cohen's d | 解释 |
|-----------|------|
| < 0.1 | 效应极小，数据本身可能无信号 |
| 0.1–0.3 | 小效应，可能需要更多被试或更精准的ROI |
| > 0.5 | 中等效应，cluster test 应能检测到 |

```python
# 检查被试间符号一致性
for contrast_name in ['ext_vs_int', 'rep_vs_swi', 'interaction']:
    X = np.array(data_contrasts[contrast_name])
    subject_means = X.mean(axis=(1, 2, 3))
    signs = np.sign(subject_means)
    print(f"{contrast_name}: {(signs > 0).sum()} 正, {(signs < 0).sum()} 负")
```

**判读标准**：  
- 接近 50/50 → 效应方向在被试间不一致，数据无一致性信号  
- 70%+ 同向 → 有方向性效应，可能是统计功效问题

---

## Step 2：逐点 t-map 可视化

**目的**：排除"有局部效应但 cluster 没抓到"的可能性。

```python
import matplotlib.pyplot as plt
from scipy import stats

fig, axes = plt.subplots(1, 3, figsize=(15, 4))

for ax, contrast_name in zip(axes, ['ext_vs_int', 'rep_vs_swi', 'interaction']):
    X = np.array(data_contrasts[contrast_name])  # (n_subjects, freq, time, ch)
    X_avg_freq = X.mean(axis=1)  # 先对频率平均 → (n_subjects, time, ch)
    
    t_map, _ = stats.ttest_1samp(X_avg_freq, 0, axis=0)  # (time, ch)
    
    im = ax.imshow(t_map.T, aspect='auto', origin='lower',
                   extent=[times_alpha[0], times_alpha[-1], 0, len(info['ch_names'])],
                   cmap='RdBu_r', vmin=-4, vmax=4)
    ax.set_title(contrast_name)
    ax.set_xlabel('Time (s)')
    ax.set_ylabel('Channel index')
    plt.colorbar(im, ax=ax, label='t-value')

plt.tight_layout()
plt.show()
```

**判读标准**：  
- 图像呈现随机噪声（无空间连贯的色块）→ 确认无信号，转 Step 3  
- 存在局部连贯的色块（t > 2 或 < -2）→ 可能是 threshold 设置问题，调整后重跑 cluster test

---

## Step 3：检查数据存储格式

**目的**：确认加载的 TFR 文件是 single-trial 还是 trial-averaged。

```python
# 检查单个文件的 shape
sub = subjects[0]
cond = conditions[0]
tfr_file = os.path.join(data_dir, f"{sub}-{cond}-tfr.h5")
tfr = read_tfrs(tfr_file)[0]

print(f"Shape: {tfr.data.shape}")
# 预期: (n_trials, n_channels, n_freqs, n_times)
# ⚠️ 如果 shape[0] == 1，说明是 trial-averaged 数据
```

**如果是 trial-averaged（shape[0] == 1）**：  
这不影响 group-level 分析的正确性（每个被试贡献一个 average），但意味着被试内的 trial 信息已丢失，无法做 trial-level 分析。

---

## Step 4：检查条件间均值差异

**目的**：确认四个条件的 power 在数值上是否有差异。

```python
grand_avg = {cond: [] for cond in conditions}

for sub in subjects:
    for cond in conditions:
        # 取 [0] 去掉 trial 维度（如果是 averaged 数据）
        grand_avg[cond].append(data_all[sub][cond][0])

for cond in conditions:
    arr = np.array(grand_avg[cond])  # (n_subjects, ch, freq, time)
    print(f"{cond}:  mean = {arr.mean():.4f},  std across subjects = {arr.std():.4f}")
```

**判读标准**：  
- 四个条件均值几乎相同 → 条件间差异极小，转 Step 5 排查 baseline  
- 条件间有数值差异但方向不一致 → 转 Step 5 检查 baseline

---

## Step 5：排查 Baseline Correction

**目的**：baseline correction 方式不当可能消除条件间差异。

```python
# 检查数据范围和 baseline/task 期间的均值
sub = subjects[0]
cond = conditions[0]
tfr_file = os.path.join(data_dir, f"{sub}-{cond}-tfr.h5")
tfr = read_tfrs(tfr_file)[0]

print(f"Time range: {tfr.times[0]:.3f} to {tfr.times[-1]:.3f} s")
print(f"Data range: {tfr.data.min():.4f} to {tfr.data.max():.4f}")
print(f"Data mean: {tfr.data.mean():.6f}")

baseline_mask = tfr.times < 0
task_mask = (tfr.times >= 0) & (tfr.times <= 1.0)
print(f"Baseline mean: {tfr.data[:, :, :, baseline_mask].mean():.6f}")
print(f"Task mean:     {tfr.data[:, :, :, task_mask].mean():.6f}")
```

**常见 baseline correction 问题**：

| 问题 | 症状 | 解决方案 |
|------|------|----------|
| Baseline 包含 task 信号 | baseline 均值与 task 均值接近 | 重新选择纯 pre-stimulus 窗口 |
| 跨条件 baseline（同一 baseline 减去所有条件）| 条件间差异消失 | 改为条件内 baseline correction |
| dB 或 percent change 方向问题 | 效应方向反转 | 检查 baseline correction 公式 |
| Baseline 期间已有条件差异 | 效应被抵消 | 用更早的中性 baseline 窗口 |

---

## Step 6：验证 Contrast 逻辑

**目的**：确认 contrast 计算符合实验假设。

```python
# 打印 contrast 计算逻辑，对照假设检查
print("ext_vs_int = 0.5*(ext_rep + ext_swi) - 0.5*(int_rep + int_swi)")
print("rep_vs_swi = 0.5*(ext_rep + int_rep) - 0.5*(ext_swi + int_swi)")
print("interaction = (ext_rep - ext_swi) - (int_rep - int_swi)")

# 可视化单个被试的 contrast topomap（用于核查方向）
import mne

sub = subjects[0]
ext_rep = data_all[sub]['cue_ext_rep_sat'][0]  # (ch, freq, time)
int_rep = data_all[sub]['cue_int_rep_sat'][0]

diff = (ext_rep - int_rep).mean(axis=(1, 2))  # 对 freq 和 time 平均 → (ch,)

# 用 MNE 画 topomap
evoked_diff = mne.EvokedArray(diff[:, np.newaxis], info)
evoked_diff.plot_topomap(times=[0], title=f'{sub}: ext - int (alpha avg)', show=True)
```

---

## 决策树总结

```
Cluster test 无显著结果
│
├── Step 1: Cohen's d < 0.1 AND 符号一致性 ≈ 50/50?
│   ├── YES → 数据本身无信号，继续排查上游
│   └── NO  → 统计功效问题，考虑降维（ROI分析）
│
├── Step 2: t-map 有局部色块?
│   ├── YES → 调整 cluster threshold 或改用 ROI
│   └── NO  → 确认无空间信号，继续排查
│
├── Step 3: TFR shape[0] == 1?
│   ├── YES → trial-averaged，正常，不影响 group 分析
│   └── NO  → single-trial，检查 mean(axis=0) 是否正确
│
├── Step 4: 四个条件均值几乎相同?
│   ├── YES → baseline correction 可能消除了差异
│   └── NO  → contrast 逻辑可能有问题
│
└── Step 5: 检查 baseline correction 方式
    ├── 重新做 baseline correction（条件内，纯 pre-stimulus 窗口）
    └── 用未做 baseline correction 的原始数据重跑验证
```

---

## 附：Cluster Test 参数参考

```python
from scipy import stats

n_subjects = len(subjects)

# Threshold: 对应单尾 p=0.05（推荐用于双尾 tail=0）
threshold_value = stats.t.ppf(1 - 0.05, n_subjects - 1)

t_obs, clusters, cluster_pv, H0 = spatio_temporal_cluster_1samp_test(
    X,                          # (n_subjects, dim1, dim2, ..., n_channels)
    n_permutations=1000,
    threshold=threshold_value,
    tail=0,                     # 双尾
    n_jobs=-1,
    adjacency=tfr_adjacency,
    verbose=True
)

good_cluster_inds = np.where(cluster_pv < 0.05)[0]
```

> **注意**：`tail=0` 配合 `t.ppf(0.90, df)` 实际上是双尾 p=0.20 的 threshold，过于宽松。  
> 建议改用 `t.ppf(0.95, df)`（对应单尾 p=0.05，双尾 p=0.10）或 `t.ppf(0.975, df)`（双尾 p=0.05）。
