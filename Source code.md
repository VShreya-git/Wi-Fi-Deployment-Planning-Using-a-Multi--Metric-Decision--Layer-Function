import pandas as pd
import matplotlib.pyplot as plt
import numpy as np
================================================================
1. DATASET
================================================================
data = {
'Nodes': [2, 4, 6, 10, 14, 20],
'Throughput': [0.202378, 0.198715, 0.194769, 0.187337, 0.180857, 0.172437],
'CollisionRate':[1.916141, 5.038913, 8.196815, 13.950131, 19.028826, 25.203365],
'Delay': [0.566455, 0.886180, 1.229823, 1.965466, 2.828707, 4.151186],
'PacketLoss': [1.202020, 2.989899, 4.915825, 8.543434, 11.708514, 15.818182],
'BackoffFail': [9.691843, 13.205122,16.135136,20.833296,25.019293,29.806836],
}
df = pd.DataFrame(data)
================================================================
2. DYNAMIC WEIGHTS
================================================================
def compute_dynamic_weights(row, df):
performance_drop = 1 - row['Throughput'] / df['Throughput'].max()
w_T = 0.2 + 0.2 * performance_drop
w_PL = 0.2 + 0.2 * (row['PacketLoss'] / df['PacketLoss'].max())
w_CR = 0.15 + 0.2 * (row['CollisionRate'] / df['CollisionRate'].max())
w_D = 0.15 + 0.15 * (row['Delay'] / df['Delay'].max())
w_BF = 0.15 + 0.2 * (row['BackoffFail']/ df['BackoffFail'].max())
total = w_T + w_PL + w_CR + w_D + w_BF
return pd.Series({
'w_T': w_T / total,
'w_PL': w_PL / total,
'w_CR': w_CR /total,
'w_D': w_D / total,
'w_BF': w_BF / total
})
weights_df = df.apply(lambda row: compute_dynamic_weights(row, df), axis=1)
df = pd.concat([df, weights_df], axis=1)
================================================================
3. NORMALIZATION (FIXED - NO ZERO COLLAPSE)
================================================================
def normalize(col, invert=False):
epsilon = 1e-6
norm = (df[col] - df[col].min()) / (df[col].max() - df[col].min() + epsilon)
norm = norm.clip(0.05, 1) # prevents full collapse to 0
return 1 - norm if invert else norm
df['score_T'] = normalize('Throughput')
df['score_D'] = normalize('Delay', invert=True)
df['score_PL'] = normalize('PacketLoss', invert=True)
df['score_CR'] = normalize('CollisionRate', invert=True)
df['score_BF'] = normalize('BackoffFail', invert=True)
# ================================================================
# 4. NHI CALCULATION
# ================================================================
df['NHI'] = (
df['score_T'] * df['w_T'] +
df['score_D'] * df['w_D'] +
df['score_PL'] * df['w_PL'] +
df['score_CR'] * df['w_CR'] +
df['score_BF'] * df['w_BF']
) * 10
# ================================================================
# 5. ADAPTIVE THRESHOLDS
# ================================================================
high_thresh = df['NHI'].quantile(0.7)
low_thresh = df['NHI'].quantile(0.4)
def make_decision(score):
if score > high_thresh:
return 'SUITABLE (Optimal)'
elif score > low_thresh:
return 'RISKY (Degraded)'
else:
return 'UNSUITABLE (Failure)'
df['Decision'] = df['NHI'].apply(make_decision)
# ================================================================
# 6. PRINT RESULTS
# ================================================================
print("=" * 80)
print(" ADAPTIVE WiFi NETWORK HEALTH INDEX (FINAL MODEL)")
print("=" * 80)
print(f"\nThresholds → SUITABLE > {high_thresh:.2f} | RISKY > {low_thresh:.2f}\n")
print(df[['Nodes','Throughput','CollisionRate','Delay',
'PacketLoss','BackoffFail','NHI','Decision']].to_string(index=False))
# Critical node detection
critical_node = df.loc[df['NHI'].idxmin(), 'Nodes']
print(f"\nı. Critical failure starts at: {critical_node} nodes")
print("=" * 80)
# ================================================================
# 7. COLOR MAP
# ================================================================
color_map = {
'SUITABLE (Optimal)': '#2ecc71',
'RISKY (Degraded)': '#f1c40f',
'UNSUITABLE (Failure)': '#e74c3c'
}
node_colors = df['Decision'].map(color_map)
# ================================================================
# FIGURE 1: NHI BAR
# ================================================================
plt.figure(figsize=(10,6))
bars = plt.bar(df['Nodes'].astype(str), df['NHI'],
color=node_colors, edgecolor='black')
plt.axhline(high_thresh, linestyle='--', label='Suitable')
plt.axhline(low_thresh, linestyle='--', label='Risky')
for bar in bars:
h = bar.get_height()
plt.text(bar.get_x()+bar.get_width()/2, h+0.1, f'{h:.2f}', ha='center')
plt.title("Adaptive NHI vs Nodes")
plt.xlabel("Nodes")
plt.ylabel("NHI")
plt.legend()
plt.grid()
plt.savefig("NHI_dynamic.png", dpi=150)
# ================================================================
# FIGURE 2: METRIC TRENDS
# ================================================================
fig, axes = plt.subplots(3, 2, figsize=(12,10))
metrics = ['Throughput','Delay','PacketLoss','CollisionRate','BackoffFail']
ax_list = axes.flatten()
for ax, col in zip(ax_list, metrics):
ax.plot(df['Nodes'], df[col], 'o-', linewidth=2)
ax.set_title(col)
ax.grid()
for ax in ax_list[len(metrics):]:
ax.axis('off')
plt.tight_layout()
plt.savefig("metrics.png", dpi=150)
# ================================================================
# FIGURE 3: NHI BREAKDOWN
# ================================================================
plt.figure(figsize=(10,6))
bottom = np.zeros(len(df))
components = ['score_T','score_PL','score_CR','score_D','score_BF']
for comp in components:
weight_col = 'w_' + comp.split('_')[1]
contrib = df[comp] * df[weight_col] * 10
plt.bar(df['Nodes'].astype(str), contrib, bottom=bottom, label=comp)
bottom += contrib.values
plt.title("NHI Contribution Breakdown")
plt.legend()
plt.grid()
plt.savefig("NHI_breakdown.png", dpi=150)
# ================================================================
# FIGURE 4: WEIGHT EVOLUTION
# ================================================================
plt.figure(figsize=(10,6))
for w in ['w_T','w_PL','w_CR','w_D','w_BF']:
plt.plot(df['Nodes'], df[w], marker='o', label=w)
plt.title("Dynamic Weight Adaptation")
plt.xlabel("Nodes")
plt.ylabel("Weight")
plt.legend()
plt.grid()
plt.savefig("weights.png", dpi=150)
# ================================================================
# FIGURE 5: DECISION ZONES
# ================================================================
plt.figure(figsize=(10,5))
plt.plot(df['Nodes'], df['NHI'], marker='o')
plt.axhline(high_thresh, linestyle='--', label='Suitable')
plt.axhline(low_thresh, linestyle='--', label='Risky')
plt.fill_between(df['Nodes'], high_thresh, 10, alpha=0.1)
plt.fill_between(df['Nodes'], low_thresh, high_thresh, alpha=0.1)
plt.fill_between(df['Nodes'], 0, low_thresh, alpha=0.1)
plt.title("Decision Zones")
plt.xlabel("Nodes")
plt.ylabel("NHI")
plt.legend()
plt.grid()
plt.savefig("decision_zones.png", dpi=150)
plt.show()
