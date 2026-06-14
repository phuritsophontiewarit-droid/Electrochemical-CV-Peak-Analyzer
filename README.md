import pandas as pd
import matplotlib.pyplot as plt
import numpy as np
from scipy.signal import find_peaks, savgol_filter
from sklearn.linear_model import LinearRegression
import os


def find_peak_in_window(
    data,
    start_voltage=0.1,
    end_voltage=0.3,
    peak_type="oxidation",
    prominence=0.0000005
):
    window_data = data[
        (data["Voltage"] >= start_voltage) &
        (data["Voltage"] <= end_voltage)
    ].copy()

    voltage = window_data["Voltage"].values
    current = window_data["Current"].values

    baseline = np.linspace(current[0], current[-1], len(current))
    corrected_current = current - baseline

    window_data["Baseline_Current"] = baseline
    window_data["Corrected_Current"] = corrected_current

    smooth_corrected = savgol_filter(
        corrected_current,
        window_length=5,
        polyorder=2
    )

    if peak_type == "oxidation":
        peaks, _ = find_peaks(smooth_corrected, prominence=prominence)
    elif peak_type == "reduction":
        peaks, _ = find_peaks(-smooth_corrected, prominence=prominence)
    else:
        raise ValueError("peak_type must be 'oxidation' or 'reduction'")

    if len(peaks) == 0:
        peak_idx = smooth_corrected.argmax()
    else:
        peak_idx = peaks[smooth_corrected[peaks].argmax()]

    peak_voltage = voltage[peak_idx]
    raw_peak_current = current[peak_idx]
    corrected_peak_current = corrected_current[peak_idx]

    return (
        window_data,
        peak_voltage,
        raw_peak_current,
        corrected_peak_current,
        smooth_corrected
    )


files = {
    0: "CV_0_µM.csv",
    2.5: "CV_2.5_µM.csv",
    5: "CV_5_µM.csv",
    7.5: "CV_7.5_µM.csv",
    10: "CV_10_µM.csv",
    12.5: "CV_12.5_µM.csv",
    15: "CV_15_µM.csv"
}

results = []

base_dir = os.path.dirname(__file__)
output_folder = os.path.join(base_dir, "outputs")

if not os.path.exists(output_folder):
    os.makedirs(output_folder)


for concentration, filename in files.items():
    file_path = os.path.join(base_dir, filename)
    data = pd.read_csv(file_path)

    window_data, peak_voltage, raw_peak_current, corrected_peak_current, smooth_corrected = find_peak_in_window(
        data,
        start_voltage=0.1,
        end_voltage=0.3,
        peak_type="oxidation",
        prominence=0.0000005
    )

    results.append({
    "Concentration_uM": concentration,
    "Peak_Voltage_V": peak_voltage,
    "Raw_Peak_Current_A": raw_peak_current,
    "Corrected_Peak_Current_A": corrected_peak_current
})

    plt.figure(figsize=(7, 5))

    plt.plot(data["Voltage"], data["Current"], label="Raw CV Curve")

    plt.axvspan(
    0.1,
    0.3,
    alpha=0.15,
    label="Search Window"
    )

    plt.scatter(
    peak_voltage,
    raw_peak_current,
    s=90,
    label="Detected Peak"
    )

    plt.xlabel("Voltage (V)")
    plt.ylabel("Current (A)")
    plt.title(f"CV Curve with Detected Peak - {concentration} µM")
    plt.legend()
    plt.tight_layout()

    diagnostic_filename = f"{os.path.splitext(filename)[0]}_cv_peak_detection.png"

    plt.savefig(
        os.path.join(output_folder, diagnostic_filename),
        dpi=300
    )

    plt.close()


results_table = pd.DataFrame(results)

print(results_table)

results_table.to_csv(
    os.path.join(output_folder, "peak_results.csv"),
    index=False
)

X = results_table[["Concentration_uM"]]
y = results_table["Corrected_Peak_Current_A"]

model = LinearRegression()
model.fit(X, y)

slope = model.coef_[0]
intercept = model.intercept_
r2 = model.score(X, y)

print("Sensitivity / Slope:", slope, "A/µM")
print("Intercept:", intercept, "A")
print("R²:", r2)

predicted_current = model.predict(X)

plt.figure(figsize=(7, 5))

plt.scatter(
    results_table["Concentration_uM"],
    results_table["Corrected_Peak_Current_A"],
    label="Experimental Peak Current"
)

plt.plot(
    results_table["Concentration_uM"],
    predicted_current,
    label="Linear Fit"
)

equation = f"y = {slope:.2e}x + {intercept:.2e}"
r_squared = f"R² = {r2:.4f}"

plt.text(
    0.50,
    0.45,
    f"{equation}\n{r_squared}",
    transform=plt.gca().transAxes,
    fontsize=10,
    bbox=dict(boxstyle="round", facecolor="white", alpha=0.5)
)

plt.xlabel("Concentration (µM)")
plt.ylabel("Peak Current (A)")
plt.title("Calibration Curve")
plt.legend()
plt.tight_layout()

plt.savefig(
    os.path.join(output_folder, "calibration_curve.png"),
    dpi=300
)

plt.show()


calibration_summary = pd.DataFrame({
    "Parameter": ["Sensitivity / Slope", "Intercept", "R²"],
    "Value": [slope, intercept, r2],
    "Unit": ["A/µM", "A", "-"]
})

excel_file = os.path.join(
    output_folder,
    "electrochemical_analysis_results.xlsx"
)

with pd.ExcelWriter(excel_file) as writer:
    results_table.to_excel(writer, sheet_name="Peak Results", index=False)
    calibration_summary.to_excel(writer, sheet_name="Calibration Summary", index=False)
