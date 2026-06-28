# `feature/preprocessing` — CT preprocessing & lesion segmentation

Traditional (non-learning) image-processing pipeline that turns the CIRS Model 073
breast-phantom CT into a normalized volume plus a binary mask of the bright
inclusions (candidate lesions).

---

## Pipeline (notebook `03_preprocessing_phantom_paper_based.ipynb`)

```
DICOM CT
  -> HU conversion              (pixel * RescaleSlope + RescaleIntercept)
  -> fat-based normalization    (subtract mean HU of fat voxels, HU in [-200,-50])
  -> windowing                  (WW=350, WL=25  ->  rescale to [0,1])    [Prionas 2010]
  -> 5-point median filter      (slice-by-slice, 2-D per axial slice)    [Nelson 2008]
  -> Gaussian-fit threshold     (t = mu + 3*sigma on glandular peak)     [Nelson 2008, Caballo 2018]
  -> NRRD output (+ LPS spatial metadata via SimpleITK)
```

The threshold is data-driven, not arbitrary: the glandular intensity peak is fitted with a Gaussian on the
phantom-interior band `[0.30, 0.85]`, using the left side of the peak only to
avoid bias from the inclusion tail, and the inclusion threshold is set at
`mu + K*sigma` with `K_DEFAULT = 3`.

---

## Key outputs

Two phantom series are processed: S2010 and S3010.

| | S2010 | S3010 |
|---|---|---|
| shape (z,y,x) | 571 × 512 × 512 | 577 × 512 × 512 |
| in-plane spacing | 0.378906 mm | 0.333984 mm |
| slice spacing | 0.335 mm | 0.335 mm |

Per series: `filtered_volume.nrrd` (values in `[0,1]`),
`filtered_volume_hu.nrrd`, and the three binary masks `mask_inclusions.nrrd`,
`mask_fat.nrrd`, `mask_glandular.nrrd` (uint8, values `{0,1}`), plus `.npy`
mirrors and `spacing.npy`. `mask_inclusions.nrrd` is a single binary mask.

---

## Process of the data

1. **First opening of the phantom in 3D Slicer** (Dec 2025). Visual inspection,
   manual thresholding trials, first 3D reconstruction. Noted the metal seed in
   the caudal-right position used as anatomical fiducial (supervisor remark).
2. **TCIA CMB-BRCA download** (`dataset/CMB-BRCA/`). Patient set curated down to
   4 female subjects in `dataset/consider_cases/` (MSB-00727, MSB-01799,
   MSB-02599, MSB-08876). Finding: the CT acquisitions are chest/CAP, **no
   dedicated breast-lesion annotations** are present — relevant breast imaging in
   this collection is MRI/MG, not CT.
3. **Notebook `02_preprocessing_phantom_test.ipynb`** — first pipeline variant
   using CLAHE contrast enhancement. Kept as a test; outputs live under
   `output/preprocessing_data/test/`. Not used downstream.
4. **Notebook `03_..._paper_based.ipynb`** — the canonical pipeline above,
   grounded in cited references for each stage (windowing, median filter,
   Gaussian-fit threshold).
5. **Data-driven threshold** added: Gaussian fit on the glandular peak, `k=3`,
   restricted to the phantom band to exclude air dominance.

---

## Repository layout

```
notebooks/
  01_extraction_CT_patient.ipynb        patient CT extraction (scaffold)
  02_preprocessing_phantom_test.ipynb   CLAHE variant (test)
  03_preprocessing_phantom_paper_based.ipynb   canonical pipeline
  04_analyzed_phantom.ipynb             inclusion analysis
inclusions_stats.py                     standalone connected-components
output/
  figures/paper_based/                  pipeline / histogram / segmentation PNGs
  preprocessing_data/paper_based/<S2010|S3010>/   NRRD + NPY + masks
  preprocessing_data/test/              CLAHE-variant outputs (unused)
dataset/
  FANTOCCIO/S72990/{S1000,S2010,S3010,S5010}/     CIRS 073 phantom DICOM
  CMB-BRCA/ , consider_cases/           TCIA patient data
  metadata.csv
```
