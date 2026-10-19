# script_ATDE_25C
import numpy as np
import pandas as pd
from scipy.optimize import least_squares
import matplotlib.pyplot as plt
from google.colab import drive

# ==========================================
# 1. CONFIGURATION & DATA LOADING
# ==========================================

drive.mount('/content/drive')

file_path = '/content/drive/MyDrive/modelsofstability/solution.xlsx'
sheet_name = '25C'

# Define the variations (The Grid)
x_values = [0.25, 0.20, 0.15, 0.10]
y_values = [0.80, 0.85, 0.90, 0.95]

try:
    # Rows 3-14 for MA+ at 25C
    df = pd.read_excel(file_path, sheet_name=sheet_name, usecols="B:AG", skiprows=2, nrows=12, header=None)
    print(f"✅ Data Loaded from Rows 3-14 (Sheet 25C). Shape: {df.shape}")
except Exception as e:
    print(f"❌ Error loading data: {e}")
    raise

# ==========================================
# 2. SOLVER & MODEL DEFINITIONS
# ==========================================

FIXED_Y0 = 100.0

def ode_model(y, c1, c2):
    return c1 * y + c2 * y**2

def improved_euler_step(yi, h, c1, c2):
    k1 = ode_model(yi, c1, c2)
    y_predict = yi + h * k1
    k2 = ode_model(y_predict, c1, c2)
    return yi + (h / 2.0) * (k1 + k2)

def solve_ode_numerical(params, t_data):
    c1, c2 = params
    y_solve = np.zeros(len(t_data))
    y_solve[0] = FIXED_Y0
    for i in range(len(t_data) - 1):
        h = t_data[i+1] - t_data[i]
        y_solve[i+1] = improved_euler_step(y_solve[i], h, c1, c2)
    return y_solve

def residuals(params, t_data, y_data):
    y_model = solve_ode_numerical(params, t_data)
    return y_model - y_data

# ==========================================
# 3. LOOPING & PLOTTING
# ==========================================

fig, axes = plt.subplots(4, 4, figsize=(24, 20))
fig.suptitle(r'MA$^{+}$ fitted for 25$^{\circ}$C', fontsize=22)

col_idx = 0
results_summary = []

for i, x_val in enumerate(x_values):
    for j, y_val in enumerate(y_values):

        ax = axes[i, j]

        try:
            # 1. Extract & Clean Data
            t_raw = df.iloc[:, col_idx]
            y_raw = df.iloc[:, col_idx+1]
            pair_df = pd.DataFrame({'t': t_raw, 'y': y_raw}).dropna()

            t_data = pair_df['t'].values.astype(float)
            y_data = pair_df['y'].values.astype(float)

            if len(t_data) < 3:
                ax.text(0.5, 0.5, "Insufficient Data", ha='center')
                col_idx += 2
                continue

            # 2. Optimization
            initial_guess = [0.05, -0.005]
            res = least_squares(residuals, initial_guess, args=(t_data, y_data), loss='soft_l1')
            c1, c2 = res.x

            # 3. Calculate Metrics
            y_model_at_data = solve_ode_numerical([c1, c2], t_data)
            sse = np.sum((y_data - y_model_at_data) ** 2)
            sst = np.sum((y_data - np.mean(y_data)) ** 2)
            r_squared = 1 - (sse / sst) if sst > 0 else 0

            # --- FIX IS HERE: Added 'y0': FIXED_Y0 to the dictionary ---
            results_summary.append({
                'x': x_val, 'y': y_val,
                'c1': c1, 'c2': c2,
                'y0': FIXED_Y0,
                'R2': r_squared, 'SSE': sse
            })
            # -----------------------------------------------------------

            # 4. Plotting
            t_smooth = np.linspace(t_data[0], t_data[-1], 50)
            y_smooth = np.zeros(len(t_smooth))
            y_smooth[0] = FIXED_Y0
            for k in range(len(t_smooth)-1):
                h_s = t_smooth[k+1] - t_smooth[k]
                y_smooth[k+1] = improved_euler_step(y_smooth[k], h_s, c1, c2)

            ax.scatter(t_data, y_data, color='red', s=40, label='Exp')
            ax.plot(t_smooth, y_smooth, color='blue', label='ATDE')

            ax.set_title(f'x={x_val}, y={y_val}', fontsize=10, fontweight='bold')
            ax.legend(loc='upper right', fontsize=8)

            text_str = (f'$c_1$: {c1:.3f}\n'
                        f'$c_2$: {c2:.3f}\n'
                        f'$R^2$: {r_squared:.4f}\n'
                        f'SSE: {sse:.4f}')

            ax.text(0.96, 0.72, text_str, transform=ax.transAxes, fontsize=9,
                    ha='right', va='top',
                    bbox=dict(facecolor='white', edgecolor='gray', alpha=0.9, boxstyle='round'))

            if i == 3: ax.set_xlabel('t (year)')
            if j == 0: ax.set_ylabel(r'MA$^{+}$ (%)')

            ax.grid(True, alpha=0.3)

        except Exception as e:
            ax.text(0.5, 0.5, "Error", ha='center')
            print(f"Error at pair {col_idx}: {e}")

        col_idx += 2

plt.tight_layout(rect=[0, 0.03, 1, 0.95])
plt.savefig("grid_4x4_fit_MA_25C_FixedY0.png")
plt.show()

# Summary Table
print("\n" + "="*70)
print("         PARAMETER & METRICS SUMMARY TABLE (MA+ 25°C)")
print("="*70)
if results_summary:
    print(pd.DataFrame(results_summary)[['x', 'y', 'c1', 'c2', 'y0', 'R2', 'SSE']].to_string(index=False))

import numpy as np
import pandas as pd
from scipy.optimize import least_squares
import matplotlib.pyplot as plt
from google.colab import drive

# ==========================================
# 1. CONFIGURATION & DATA LOADING
# ==========================================

drive.mount('/content/drive')

file_path = '/content/drive/MyDrive/modelsofstability/solution.xlsx'
sheet_name = '25C'

# Define the variations (The Grid)
x_values = [0.25, 0.20, 0.15, 0.10]
y_values = [0.80, 0.85, 0.90, 0.95]

try:
    # UPDATED: skiprows=17 (Skips rows 1-17, starts reading at Row 18)
    # nrows=12 (Reads rows 18 through 29)
    df = pd.read_excel(file_path, sheet_name=sheet_name, usecols="B:AG", skiprows=17, nrows=12, header=None)

    print(f"✅ Data Loaded from Rows 18-29 (Sheet 25C). Shape: {df.shape}")
except Exception as e:
    print(f"❌ Error loading data: {e}")
    raise

# ==========================================
# 2. SOLVER & MODEL DEFINITIONS
# ==========================================

# --- FIXED PARAMETER ---
FIXED_Y0 = 100.0
# -----------------------

def ode_model(y, c1, c2):
    return c1 * y + c2 * y**2

def improved_euler_step(yi, h, c1, c2):
    k1 = ode_model(yi, c1, c2)
    y_predict = yi + h * k1
    k2 = ode_model(y_predict, c1, c2)
    return yi + (h / 2.0) * (k1 + k2)

def solve_ode_numerical(params, t_data):
    # Only unpack c1 and c2. y0 is FIXED at 100.
    c1, c2 = params

    y_solve = np.zeros(len(t_data))
    y_solve[0] = FIXED_Y0

    for i in range(len(t_data) - 1):
        h = t_data[i+1] - t_data[i]
        y_solve[i+1] = improved_euler_step(y_solve[i], h, c1, c2)
    return y_solve

def residuals(params, t_data, y_data):
    y_model = solve_ode_numerical(params, t_data)
    return y_model - y_data

# ==========================================
# 3. LOOPING & PLOTTING
# ==========================================

fig, axes = plt.subplots(4, 4, figsize=(24, 20))
fig.suptitle(r'FA$^{+}$ fitted for 25$^{\circ}$C', fontsize=22) # <--- Updated Title

col_idx = 0
results_summary = []

for i, x_val in enumerate(x_values):
    for j, y_val in enumerate(y_values):

        ax = axes[i, j]

        try:
            # 1. Extract & Clean Data
            t_raw = df.iloc[:, col_idx]
            y_raw = df.iloc[:, col_idx+1]
            pair_df = pd.DataFrame({'t': t_raw, 'y': y_raw}).dropna()

            t_data = pair_df['t'].values.astype(float)
            y_data = pair_df['y'].values.astype(float)

            if len(t_data) < 3:
                ax.text(0.5, 0.5, "Insufficient Data", ha='center')
                col_idx += 2
                continue

            # 2. Optimization (Only optimizing c1, c2)
            # Using smaller initial guess for lower temp kinetics
            initial_guess = [0.05, -0.005]
            res = least_squares(residuals, initial_guess, args=(t_data, y_data), loss='soft_l1')
            c1, c2 = res.x

            # 3. Calculate Metrics
            y_model_at_data = solve_ode_numerical([c1, c2], t_data)
            sse = np.sum((y_data - y_model_at_data) ** 2)
            sst = np.sum((y_data - np.mean(y_data)) ** 2)
            r_squared = 1 - (sse / sst) if sst > 0 else 0

            results_summary.append({
                'x': x_val, 'y': y_val,
                'c1': c1, 'c2': c2,
                'y0': FIXED_Y0,
                'R2': r_squared, 'SSE': sse
            })

            # 4. Generate Smooth Fit
            t_smooth = np.linspace(t_data[0], t_data[-1], 50)
            y_smooth = np.zeros(len(t_smooth))
            y_smooth[0] = FIXED_Y0
            for k in range(len(t_smooth)-1):
                h_s = t_smooth[k+1] - t_smooth[k]
                y_smooth[k+1] = improved_euler_step(y_smooth[k], h_s, c1, c2)

            # 5. Plotting
            ax.scatter(t_data, y_data, color='red', s=40, label='Exp')
            ax.plot(t_smooth, y_smooth, color='blue', label='ATDE')

            ax.set_title(f'x={x_val}, y={y_val}', fontsize=10, fontweight='bold')

            ax.legend(loc='upper right', fontsize=8)

            # Text Box (Metrics)
            text_str = (f'$c_1$: {c1:.3f}\n'
                        f'$c_2$: {c2:.3f}\n'
                        f'$R^2$: {r_squared:.4f}\n'
                        f'SSE: {sse:.4f}')

            ax.text(0.96, 0.72, text_str, transform=ax.transAxes, fontsize=9,
                    ha='right', va='top',
                    bbox=dict(facecolor='white', edgecolor='gray', alpha=0.9, boxstyle='round'))

            if i == 3: ax.set_xlabel('t (year)')

            # --- UPDATED Y-AXIS LABEL ---
            if j == 0: ax.set_ylabel(r'FA$^{+}$ (%)')
            # ----------------------------

            ax.grid(True, alpha=0.3)

        except Exception as e:
            ax.text(0.5, 0.5, "Error", ha='center')
            print(f"Error at pair {col_idx}: {e}")

        col_idx += 2

plt.tight_layout(rect=[0, 0.03, 1, 0.95])
plt.savefig("grid_4x4_fit_FA_25C_FixedY0.png")
plt.show()

# Summary Table
print("\n" + "="*70)
print("         PARAMETER & METRICS SUMMARY TABLE (FA+ 25°C)")
print("="*70)
if results_summary:
    print(pd.DataFrame(results_summary)[['x', 'y', 'c1', 'c2', 'y0', 'R2', 'SSE']].to_string(index=False))

import numpy as np
import pandas as pd
from scipy.optimize import least_squares
from scipy.integrate import odeint
import matplotlib.pyplot as plt
from google.colab import drive

# ==========================================
# 1. CONFIGURATION & DATA LOADING
# ==========================================

drive.mount('/content/drive')

file_path = '/content/drive/MyDrive/modelsofstability/byproduct.xlsx'
sheet_name = '25C'

x_values = [0.25, 0.20, 0.15, 0.10]
y_values = [0.80, 0.85, 0.90, 0.95]

try:
    # Reading Rows 3-14 (MFA+ for 25C)
    # skiprows=2 skips rows 1 & 2. nrows=12 reads rows 3-14.
    df = pd.read_excel(file_path, sheet_name=sheet_name, usecols="B:AG", skiprows=2, nrows=12, header=None)
    print(f"✅ Data Loaded from Rows 3-14 (Sheet 25C - MFA+). Shape: {df.shape}")
except Exception as e:
    print(f"❌ Error loading data: {e}")
    raise

# ==========================================
# 2. ATDE MODEL DEFINITION
# ==========================================

def atde_deriv(y, t, c0, c1, c2):
    y_val = y[0] if isinstance(y, (list, np.ndarray)) else y
    dydt = c0 + c1 * y_val + c2 * (y_val**2)
    return dydt

def solve_atde(params, t_eval, y0):
    c0, c1, c2 = params
    # Integrate ODE
    y_sol = odeint(atde_deriv, y0, t_eval, args=(c0, c1, c2))
    return y_sol.flatten()

def residuals(params, t_data, y_data):
    y0 = y_data[0]
    y_model = solve_atde(params, t_data, y0)
    return y_model - y_data

# ==========================================
# 3. LOOPING & PLOTTING
# ==========================================

fig, axes = plt.subplots(4, 4, figsize=(24, 20))
fig.suptitle(r'MFA$^{+}$ fitted for 25$^{\circ}$C', fontsize=22)

col_idx = 0
results_summary = []

for i, x_val in enumerate(x_values):
    for j, y_val in enumerate(y_values):

        ax = axes[i, j]

        try:
            # 1. Extract & Clean Data
            t_raw = df.iloc[:, col_idx]
            y_raw = df.iloc[:, col_idx+1]
            pair_df = pd.DataFrame({'t': t_raw, 'y': y_raw}).dropna()

            t_data = pair_df['t'].values.astype(float)
            y_data = pair_df['y'].values.astype(float)

            if len(t_data) < 3:
                ax.text(0.5, 0.5, "Insufficient Data", ha='center')
                col_idx += 2
                continue

            # 2. Initial Parameter Guess
            if t_data[1] != t_data[0]:
                slope_est = (y_data[1] - y_data[0]) / (t_data[1] - t_data[0])
            else:
                slope_est = 0.01 # Slower at 25C

            # Adjusted guess for 25C (slower kinetics)
            initial_guess = [slope_est, -0.001, -0.0001]

            # 3. Optimization
            res = least_squares(residuals, initial_guess, args=(t_data, y_data), loss='soft_l1')
            c0, c1, c2 = res.x

            # 4. Calculate Metrics
            y0 = y_data[0]
            y_model_at_data = solve_atde(res.x, t_data, y0)

            sse = np.sum((y_data - y_model_at_data) ** 2)
            sst = np.sum((y_data - np.mean(y_data)) ** 2)
            r_squared = 1 - (sse / sst) if sst > 0 else 0

            results_summary.append({
                'x': x_val, 'y': y_val,
                'c0': c0, 'c1': c1, 'c2': c2,
                'R2': r_squared, 'SSE': sse
            })

            # 5. Generate Smooth Fit
            t_smooth = np.linspace(0, t_data.max() * 1.1, 50)
            y_smooth = solve_atde(res.x, t_smooth, y0)

            # 6. Plotting
            ax.scatter(t_data, y_data, color='purple', s=40, label='MFA Exp')
            ax.plot(t_smooth, y_smooth, color='green', label='ATDE Fit')

            ax.set_title(f'x={x_val}, y={y_val}', fontsize=10, fontweight='bold')
            ax.legend(loc='lower right', fontsize=8)

            # Text Box
            text_str = (f'$c_0$: {c0:.3f}\n'
                        f'$c_1$: {c1:.3f}\n'
                        f'$c_2$: {c2:.3f}\n'
                        f'$R^2$: {r_squared:.4f}\n'
                        f'SSE: {sse:.4f}')

            ax.text(0.96, 0.15, text_str, transform=ax.transAxes, fontsize=9,
                    ha='right', va='bottom',
                    bbox=dict(facecolor='white', edgecolor='gray', alpha=0.9, boxstyle='round'))

            if i == 3: ax.set_xlabel('t (year)')
            if j == 0: ax.set_ylabel(r'MFA$^{+}$ (%)')

            ax.grid(True, alpha=0.3)

        except Exception as e:
            ax.text(0.5, 0.5, "Error", ha='center')
            print(f"Error at pair {col_idx}: {e}")

        col_idx += 2

plt.tight_layout(rect=[0, 0.03, 1, 0.95])
plt.savefig("ATDE_MFA_25C.png")
plt.show()

# Summary Table
print("\n" + "="*80)
print("         PARAMETER & METRICS SUMMARY TABLE (MFA+ 25°C - ATDE)")
print("="*80)
if results_summary:
    cols = ['x', 'y', 'c0', 'c1', 'c2', 'R2', 'SSE']
    print(pd.DataFrame(results_summary)[cols].to_string(index=False))

import numpy as np
import pandas as pd
from scipy.optimize import least_squares
from scipy.integrate import odeint
import matplotlib.pyplot as plt
from google.colab import drive

# ==========================================
# 1. CONFIGURATION & DATA LOADING
# ==========================================

drive.mount('/content/drive')

file_path = '/content/drive/MyDrive/modelsofstability/byproduct.xlsx'
sheet_name = '25C'

x_values = [0.25, 0.20, 0.15, 0.10]
y_values = [0.80, 0.85, 0.90, 0.95]

try:
    # Reading Rows 18-29 (DMFA+ for 25C)
    # skiprows=17 skips rows 1-17. nrows=12 reads rows 18-29.
    df = pd.read_excel(file_path, sheet_name=sheet_name, usecols="B:AG", skiprows=17, nrows=12, header=None)
    print(f"✅ Data Loaded from Rows 18-29 (Sheet 25C - DMFA+). Shape: {df.shape}")
except Exception as e:
    print(f"❌ Error loading data: {e}")
    raise

# ==========================================
# 2. ATDE MODEL DEFINITION
# ==========================================

def atde_deriv(y, t, c0, c1, c2):
    y_val = y[0] if isinstance(y, (list, np.ndarray)) else y
    dydt = c0 + c1 * y_val + c2 * (y_val**2)
    return dydt

def solve_atde(params, t_eval, y0):
    c0, c1, c2 = params
    # Integrate ODE
    y_sol = odeint(atde_deriv, y0, t_eval, args=(c0, c1, c2))
    return y_sol.flatten()

def residuals(params, t_data, y_data):
    y0 = y_data[0]
    y_model = solve_atde(params, t_data, y0)
    return y_model - y_data

# ==========================================
# 3. LOOPING & PLOTTING
# ==========================================

fig, axes = plt.subplots(4, 4, figsize=(24, 20))
fig.suptitle(r'DMFA$^{+}$ fitted for 25$^{\circ}$C', fontsize=22)

col_idx = 0
results_summary = []

for i, x_val in enumerate(x_values):
    for j, y_val in enumerate(y_values):

        ax = axes[i, j]

        try:
            # 1. Extract & Clean Data
            t_raw = df.iloc[:, col_idx]
            y_raw = df.iloc[:, col_idx+1]
            pair_df = pd.DataFrame({'t': t_raw, 'y': y_raw}).dropna()

            t_data = pair_df['t'].values.astype(float)
            y_data = pair_df['y'].values.astype(float)

            if len(t_data) < 3:
                ax.text(0.5, 0.5, "Insufficient Data", ha='center')
                col_idx += 2
                continue

            # 2. Initial Parameter Guess
            if t_data[1] != t_data[0]:
                slope_est = (y_data[1] - y_data[0]) / (t_data[1] - t_data[0])
            else:
                slope_est = 0.01

            # Initial guess for 25C (slower kinetics)
            initial_guess = [slope_est, -0.001, -0.0001]

            # 3. Optimization
            res = least_squares(residuals, initial_guess, args=(t_data, y_data), loss='soft_l1')
            c0, c1, c2 = res.x

            # 4. Calculate Metrics
            y0 = y_data[0]
            y_model_at_data = solve_atde(res.x, t_data, y0)

            sse = np.sum((y_data - y_model_at_data) ** 2)
            sst = np.sum((y_data - np.mean(y_data)) ** 2)
            r_squared = 1 - (sse / sst) if sst > 0 else 0

            results_summary.append({
                'x': x_val, 'y': y_val,
                'c0': c0, 'c1': c1, 'c2': c2,
                'R2': r_squared, 'SSE': sse
            })

            # 5. Generate Smooth Fit
            t_smooth = np.linspace(0, t_data.max() * 1.1, 50)
            y_smooth = solve_atde(res.x, t_smooth, y0)

            # 6. Plotting
            ax.scatter(t_data, y_data, color='purple', s=40, label='DMFA Exp')
            ax.plot(t_smooth, y_smooth, color='green', label='ATDE Fit')

            ax.set_title(f'x={x_val}, y={y_val}', fontsize=10, fontweight='bold')
            ax.legend(loc='lower right', fontsize=8)

            # Text Box
            text_str = (f'$c_0$: {c0:.3f}\n'
                        f'$c_1$: {c1:.3f}\n'
                        f'$c_2$: {c2:.3f}\n'
                        f'$R^2$: {r_squared:.4f}\n'
                        f'SSE: {sse:.4f}')

            ax.text(0.96, 0.15, text_str, transform=ax.transAxes, fontsize=9,
                    ha='right', va='bottom',
                    bbox=dict(facecolor='white', edgecolor='gray', alpha=0.9, boxstyle='round'))

            if i == 3: ax.set_xlabel('t (year)')
            if j == 0: ax.set_ylabel(r'DMFA$^{+}$ (%)')

            ax.grid(True, alpha=0.3)

        except Exception as e:
            ax.text(0.5, 0.5, "Error", ha='center')
            print(f"Error at pair {col_idx}: {e}")

        col_idx += 2

plt.tight_layout(rect=[0, 0.03, 1, 0.95])
plt.savefig("ATDE_DMFA_25C.png")
plt.show()

# Summary Table
print("\n" + "="*80)
print("         PARAMETER & METRICS SUMMARY TABLE (DMFA+ 25°C - ATDE)")
print("="*80)
if results_summary:
    cols = ['x', 'y', 'c0', 'c1', 'c2', 'R2', 'SSE']
    print(pd.DataFrame(results_summary)[cols].to_string(index=False))
