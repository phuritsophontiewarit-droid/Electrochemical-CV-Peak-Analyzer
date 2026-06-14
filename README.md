# Electrochemical CV Peak Analyzer

Python tool for analyzing cyclic voltammetry (CV) data from electrochemical sensor experiments.

## Features
- Reads multiple CV datasets from CSV files
- Detects oxidation peak current within 0.1–0.3 V
- Applies smoothing using Savitzky–Golay filtering
- Generates calibration curve
- Calculates sensitivity, intercept, and R²
- Exports peak results to CSV and Excel
- Saves diagnostic CV plots

## Skills Demonstrated
- Python
- Pandas
- NumPy
- Matplotlib
- SciPy
- Scikit-learn
- Electrochemical data analysis
- Sensor calibration

## Input Format

CSV files should contain:

```csv
Voltage,Current
0.10,0.0000012
0.11,0.0000015
