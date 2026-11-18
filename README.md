# Stacking Regressor – Gross Emissions Alih Fungsi Lahan Sawit

Prediksi *gross emissions* dari alih fungsi lahan (tonnes CO₂-equivalent) per kabupaten menggunakan **Stacking Regressor** (Gradient Boosting + KNN + SVR, meta-learner Ridge).

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/nasythryn/stack-regression-gross-emissions-luc-palm-oil/blob/main/stack_clip.ipynb)

## Metode

- **Target:** `gross_emissions_from_land-use_change_tonnes_co₂-equivalent`
- **Fitur prediktor:**
  - `annual_palm_deforestation_(industrial)_hectares`
  - `oil_palm_planted_on_peat_hectares`
  - `oil_palm_planted_area_hectares`
- **Pembagian data:** latih = tahun ≤ 2022 (1.560 baris), uji = tahun ≥ 2023 (390 baris). Kabupaten yang tidak aktif pada data latih dihapus.
- **Preprocessing:** `log1p` lalu `RobustScaler` pada fitur dan target (target lewat `TransformedTargetRegressor`).
- **Model dasar:** Gradient Boosting, SVR, KNN, dituning dengan GridSearchCV dan `TimeSeriesSplit` (3 lipatan), skor RMSE.
- **Stacking:** prediksi out-of-fold dari model dasar (3 lipatan, tanpa pengacakan), meta-learner Ridge.
- **Evaluasi:** RMSE, MAE, R² pada data latih dan uji.

```
Data mentah -> split latih/uji -> log1p -> RobustScaler -> [GB | KNN | SVR] -> Ridge -> invers (RobustScaler -> expm1) -> prediksi
```

## Ringkasan Hasil

Model stacking memberi galat terendah dan R² tertinggi pada data uji (tahun 2023 dan seterusnya), dibanding tiga model dasarnya.

| Model | RMSE Uji | MAE Uji | R² Uji |
|---|---:|---:|---:|
| **Stacking GB + KNN + SVR** | **308.104** | **106.606** | **0,758** |
| Gradient Boosting | 318.583 | 119.263 | 0,741 |
| KNN | 319.952 | 113.760 | 0,739 |
| SVR | 330.108 | 146.499 | 0,722 |

*(RMSE dan MAE dalam tonnes CO₂-eq. Nilai diambil dari `hasil_excel/hasil_stacking.xlsx`, sheet `evaluasi_model`.)*

- RMSE uji stacking sekitar **3,3% lebih rendah** dari Gradient Boosting, model dasar terbaik.

### Kontribusi tiap model dasar (koefisien meta-learner Ridge)

| Model dasar | Koefisien |
|---|---:|
| Gradient Boosting | 0,668 |
| KNN | 0,256 |
| SVR | 0,068 |

Gradient Boosting paling berpengaruh pada prediksi akhir, KNN menyumbang sedang, dan SVR paling kecil. Intercept 0,008 dan alpha Ridge 1,0.

### Waktu komputasi

| Model | Total tuning + fit (detik) | Inferensi 390 baris (ms) |
|---|---:|---:|
| KNN | 2,47 | 7,8 |
| SVR | 18,35 | 51,7 |
| Gradient Boosting | 37,53 | 5,9 |
| Stacking | 60,04 | 71,1 |

Stacking paling lambat karena menjalankan ketiga model dasar sekaligus, tetapi inferensinya tetap sekitar 0,18 ms per sampel.

## Struktur Repositori

```
NAMA-REPO/
├── README.md
├── stack.ipynb
├── data/
│   └── Gross_emissions.csv
├── hasil/
│   ├── hasil_preprocessing.xlsx          # data skala log1p & RobustScaler, parameter scaler
│   ├── hasil_stacking.xlsx               # OOF, koefisien meta-learner, evaluasi, waktu komputasi
│   ├── hasil_invers_prediksi.xlsx        # aktual & prediksi di tiap skala, per model
│   ├── hasil_aktual_vs_prediksi.xlsx     # fitur, aktual, prediksi, selisih (data uji)
└── visualisasi/
    └── (grafik .png)
```

## Cara Menjalankan

1. Klik tombol **Open In Colab** di atas, atau unduh `stack_clip.ipynb`.
2. Siapkan `Gross_emissions.csv`. Jika notebook dijalankan dari folder repositori, ubah baris pembacaan data menjadi:
   ```python
   df = pd.read_csv('data/Gross_emissions.csv')
   ```
   Di Colab, unggah CSV ke panel Files lalu pakai `pd.read_csv('Gross_emissions.csv')` seperti semula.
3. Jalankan seluruh sel berurutan (**Runtime → Run all**).
4. File Excel dan gambar tersimpan di folder kerja. Seed acak dikunci (`seed=42`) agar hasil bisa direproduksi.

## Keterbatasan

- Data uji hanya mencakup periode tahun 2023 ke atas dan sebagian baris bernilai nol, sehingga galat bisa berbeda antar kabupaten dan antar tahun (lihat grafik error per kabupaten dan per tahun).
- Model hanya memakai tiga fitur luas lahan, tanpa faktor lain seperti tutupan lahan atau curah hujan.
- Hasil berlaku untuk dataset ini dan belum diuji pada data wilayah lain.

## Library

Python 3, numpy, pandas, scikit-learn, scipy, matplotlib, seaborn, openpyxl.
