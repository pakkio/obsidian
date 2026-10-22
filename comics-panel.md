# comics-panel

> Rilevamento di vignette in pagine di fumetti scansionate (computer vision classica).

## Concetti
[[Python]] [[OCR]]

## Panoramica
Un solo script (`panel_detector.py`) con tre strategie: contorni (maschere nero/bianco/dual o Canny), watershed e fallback a griglia con proiezioni. De-duplicazione, ordinamento di lettura occidentale, crop JPEG/PNG, JSON di metadati, debug visuale. Niente deep learning.

## Tech Stack
Python, OpenCV, NumPy, SciPy, scikit-image. Repo: https://github.com/pakkio/comics-panel

## Collegamenti
[[ocrlocal]], [[ocr]]
