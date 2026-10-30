# Sleep-Quality (WASO) Prediction, from notebook to microcontroller

Predicts whether a person will have a night of fragmented sleep (high **WASO**, wake after sleep onset) from the previous 7 days of wearable and questionnaire data, then runs a compressed version of the model on an **Arduino Nano 33 BLE**.

Research project at IIIT Vadodara, replicating and extending *"Predicting sleep quality with digital biomarkers and ANN"* (Lee et al., 2025). Team project; contributors are listed in the project report.

## About the data

The paper's raw dataset is not public, so every notebook here **generates a synthetic dataset** built to match the paper's published statistics: 82 participants × 27 days = 2,214 daily records. A latent stress variable ties the features together the way they relate in real physiology (higher LF/HF, lower RMSSD, worse insomnia score). Results below are on this simulated cohort, not on real patients.

Features per day (7): LF/HF ratio, RMSSD (HRV), steps, accelerometer, gyroscope, ISI (insomnia) and WHOQOL (quality of life). Target: next-night WASO above the cohort threshold (binary).

## Results

Subject-wise split: 66 participants train, 16 test. No person appears in both.

| Model | Accuracy | Precision | Recall | AUROC |
|---|---|---|---|---|
| **LSTM + XGBoost ensemble** | 0.809 | 0.858 | 0.855 | **0.870** |
| XGBoost | 0.782 | 0.881 | 0.780 | 0.866 |
| LSTM | 0.718 | 0.923 | 0.630 | 0.856 |
| Transformer | 0.747 | 0.877 | 0.723 | 0.854 |
| TCN | 0.771 | 0.840 | 0.811 | 0.854 |
| GRU | 0.709 | 0.951 | 0.604 | 0.850 |
| Random Forest | 0.762 | 0.865 | 0.762 | 0.845 |
| ARIMA | 0.613 | 0.705 | 0.756 | 0.485 |

Checks that the numbers are not leakage:
- **Label shuffle:** training on shuffled labels gives AUROC ≈ 0.50 (chance).
- **Questionnaire ablation:** a sensor-only model is compared against the full feature set.
- **Explainability:** SHAP (tree model) and LIME (LSTM) agree that HRV and questionnaire features drive predictions.
- Decision thresholds are tuned per model with Youden's J.

## On-chip deployment (TinyML)

LSTM layers are poorly supported by TensorFlow Lite Micro, so the on-chip model is a small 1D-CNN.

| | FP32 CNN | INT8 CNN (on chip) |
|---|---|---|
| Parameters | 1,425 | 1,425 |
| AUROC | 0.813 | 0.803 |
| Accuracy | 73.1% | 72.8% |
| Model size | – | 7.3 KB, 7,312 bytes (`waso_model_int8.tflite`) |

- Full-integer post-training quantization (weights and activations) with a 100-sample calibration set.
- The model and the scaler parameters are exported as a C array in `waso_model.h`.
- `WASO_Inference.ino` runs inference with a 12 KB tensor arena. On the Nano 33 BLE: about 8 ms per prediction, about 25 KB of the 256 KB RAM in use.

## Repository layout

| Path | What it is |
|---|---|
| `Sleep_Quality_Prediction_FINAL.ipynb` | Main pipeline: data generation, subject-wise split, 7 models, ensemble, SHAP/LIME, leakage tests |
| `Sleep_WASO_Prediction_Clean.ipynb` | Alternate leak-free version with the questionnaire ablation |
| `Chip_Imp/WASO_TinyML_Conversion.ipynb` | 1D-CNN training, INT8 quantization, C header export |
| `Chip_Imp/waso_model_int8.tflite`, `waso_model.h` | The quantized model |
| `WASO_Inference/WASO_Inference.ino` | Arduino firmware |
| `Implementation_Guide/WASO_TinyML_Setup_Guide.pdf` | Step-by-step board setup |
| `Sleep_Quality_Project_Report.pdf` | Full project report |
| `Chip_Imp/WASO_Poster_A2.pdf` | Poster |

## Run it

1. Open `Sleep_Quality_Prediction_FINAL.ipynb` in Google Colab and run all cells. The data is generated inside the notebook, so nothing needs downloading.
2. For the chip: run `Chip_Imp/WASO_TinyML_Conversion.ipynb`, then follow `Implementation_Guide/WASO_TinyML_Setup_Guide.pdf` to flash `WASO_Inference.ino` from the Arduino IDE (board: Arduino Mbed OS Nano Boards → Arduino Nano 33 BLE).

Stack: Python, TensorFlow/Keras, TensorFlow Lite Micro, scikit-learn, XGBoost, statsmodels, SHAP, LIME, Arduino C++.
