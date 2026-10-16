# ADR 0001: Technologia i metody

- **Status:** Proponowane
- **Data:** 2026-10-16
- **Autorzy:** Max Duda, Monika Duda

## Kontekst

Projekt realizuje temat 5: klasyfikację zdjęć odpadów ze zbioru TrashNet (2 527 zdjęć, 6 klas, niezbalansowanie: `trash` 137 vs `paper` 594). Treść tematu narzuca:

- baseline ResNet-18 z wagami ImageNet na wejściu 224×224,
- co najmniej dwa warianty na tym samym podziale: model trenowany od zera oraz transfer learning z 5-fold cross-validation,
- podział 70/15/15, seed 6102 dla numpy i torch,
- metryki: accuracy, macro-F1 (wiodąca, próg 0,80), macierz pomyłek, recall per klasa,
- własną implementację pętli k-fold (bez `sklearn.KFold`),
- eksperyment: augmentacja × wielkość zbioru (100% / 50% / 25%),
- logowanie do publicznego Weights & Biases, demo w notebooku `.ipynb`,
- budżet: do 2 h GPU lub 4 h CPU na cały projekt,
- repozytorium bez `__pycache__/`, wag `.pt`/`.pth` i surowych danych.

Dokument zbiera potencjalne biblioteki, narzędzia i algorytmy oraz wskazuje, które z nich wybieramy jako domyślne.

## Decyzja

### Stos technologiczny

| Obszar | Wybór | Uzasadnienie |
|--------|-------|--------------|
| Język | Python 3.11+ | Standard w ekosystemie DL, wszystkie wymagane biblioteki dostępne. |
| Framework DL | PyTorch + torchvision | Baseline ResNet-18 i wagi `IMAGENET1K_V1` dostępne bezpośrednio w torchvision; pełna kontrola nad pętlą treningu i k-fold. |
| Akceleracja | CUDA / Apple MPS / CPU | Kod wybiera dostępne urządzenie; pozwala pracować lokalnie na Macu i na GPU w chmurze (Colab / Kaggle Notebooks). |
| Śledzenie eksperymentów | Weights & Biases | Wymóg tematu (publiczny dashboard). |
| Demo | Jupyter Notebook | Wymóg tematu (bez Gradio / Streamlit). |

### Biblioteki

**Podstawowe (planowane do użycia):**

| Biblioteka | Zastosowanie |
|------------|--------------|
| `torch`, `torchvision` | modele (ResNet-18), transformacje i augmentacje, `Dataset` / `DataLoader`, wagi ImageNet |
| `numpy` | operacje na indeksach, losowanie podziałów, własny k-fold |
| `pandas` | tabele metadanych (ścieżka, klasa, fold), agregacja wyników |
| `Pillow` | wczytywanie obrazów, analiza rozmiarów |
| `matplotlib`, `seaborn` | krzywe uczenia, macierze pomyłek, siatki przykładów, wykresy eksperymentu |
| `scikit-learn` | **wyłącznie metryki**: `f1_score`, `confusion_matrix`, `classification_report`; bez `KFold` / `StratifiedKFold` |
| `wandb` | logowanie metryk, konfiguracji, wykresów i macierzy pomyłek |
| `tqdm` | paski postępu |
| `jupyter` / `ipykernel` | notebooki EDA i demo |
| `kagglehub` lub Kaggle CLI | pobieranie zbioru (alternatywa: ręczne pobranie z GitHuba) |

**Opcjonalne (rozważane, użycie zależne od wyników i budżetu):**

| Biblioteka | Zastosowanie |
|------------|--------------|
| `torchmetrics` | alternatywa dla metryk z scikit-learn liczona na tensorach |
| `timm` | lżejsze architektury do porównania (EfficientNet-B0, MobileNetV3) — tylko z wagami ImageNet |
| `albumentations` | bogatsze augmentacje, jeśli torchvision okaże się niewystarczające |
| `pytorch-grad-cam` | mapy Grad-CAM w analizie błędów |
| `umap-learn` / t-SNE z scikit-learn | wizualizacja przestrzeni cech (embeddingów) |

### Narzędzia

- **Git + GitHub**, `.gitignore` wykluczający `__pycache__/`, wagi `*.pt` / `*.pth`, katalog `data/`, lokalne pliki `wandb/`.
- **Środowisko wirtualne:** `venv` lub `uv`, zależności przypięte w `requirements.txt`.
- **Linter / formatter:** `ruff`.
- **Weights & Biases:** publiczny projekt, jeden run na wariant / fold / konfigurację eksperymentu, grupowanie runów k-fold.
- **Środowisko obliczeniowe:** lokalnie (MPS / CPU) do prototypowania; Google Colab lub Kaggle Notebooks (GPU) do pełnych treningów.

### Algorytmy i metody

**Podział danych i walidacja**

- Stratyfikowany podział 70/15/15 (train / val / test) z seedem 6102; zbiór testowy zamrożony i używany tylko do raportu końcowego.
- Własna implementacja stratyfikowanego k-fold (k = 5): indeksy każdej klasy tasowane z seedem i rozdzielane na 5 części, fold i = część i z każdej klasy. K-fold wykonywany na części train + val (85%), test pozostaje nietknięty.
- Wynik raportowany jako średnia ± odchylenie standardowe po foldach.
- Ustalenie ziarna: `random`, `numpy`, `torch` (CPU i CUDA), deterministyczne ustawienia cuDNN, seed workerów `DataLoader`.

**Modele**

| Wariant | Opis | Rola |
|---------|------|------|
| A — baseline | ResNet-18, wagi ImageNet, pełny fine-tuning, 224×224 | punkt odniesienia |
| B — od zera | ResNet-18 bez wag (losowa inicjalizacja) | pokazanie kosztu małego zbioru |
| C — transfer + 5-fold | jak A, oceniany w 5-fold CV | wiarygodność wyniku |
| (opcj.) D | feature extraction (zamrożony backbone, trenowana tylko głowa) lub EfficientNet-B0 / MobileNetV3 | porównanie kosztu i jakości |

Strategia fine-tuningu do sprawdzenia: najpierw tylko głowa klasyfikatora (kilka epok), potem odmrożenie całej sieci z mniejszym learning rate dla backbone'u.

**Obsługa niezbalansowania**

- Wagi klas w `CrossEntropyLoss` (odwrotność częstości).
- `WeightedRandomSampler` jako alternatywa dla ważenia straty.
- Opcjonalnie: focal loss, label smoothing.
- Wybór najlepszego checkpointu według macro-F1 na walidacji, nie według accuracy.

**Augmentacja danych**

- Bazowa: `RandomResizedCrop(224)`, poziome i pionowe odbicia, rotacja, `ColorJitter`.
- Automatyczne polityki: `TrivialAugmentWide` lub `RandAugment`.
- Opcjonalnie: MixUp / CutMix.
- Walidacja i test: tylko `Resize` + `CenterCrop` + normalizacja ImageNet.

**Optymalizacja**

- AdamW, harmonogram cosine annealing (opcjonalnie z warmupem).
- Early stopping po macro-F1 na walidacji.
- Rozmiar batcha i liczba epok dobrane pod budżet czasowy.

**Analiza błędów**

- Macierz pomyłek (znormalizowana i liczbowa), recall i F1 per klasa.
- Galeria błędnych predykcji z pewnością modelu, ze szczególnym uwzględnieniem par plastik ↔ szkło i papier ↔ karton.
- Grad-CAM: na co patrzy model przy pomyłkach (obiekt, tło, odbicia).
- t-SNE / UMAP embeddingów z przedostatniej warstwy: czy problematyczne klasy nakładają się w przestrzeni cech.
- Opcjonalnie: próg pewności (softmax) i odrzucanie niepewnych predykcji do ręcznej weryfikacji.

### Plan eksperymentu: augmentacja × wielkość zbioru

Macierz 2 × 3 konfiguracji na modelu transferowym (wariant A):

| | 100% train | 50% train | 25% train |
|---|---|---|---|
| bez augmentacji | ✓ | ✓ | ✓ |
| z augmentacją | ✓ | ✓ | ✓ |

- Podzbiory 50% i 25% losowane stratyfikowanie wyłącznie z części treningowej, walidacja i test bez zmian.
- Pytanie badawcze: czy model z augmentacją na 50% (lub 25%) danych dorównuje modelowi bez augmentacji na 100%?

### Szacunek budżetu obliczeniowego

Około 13 treningów (A, B, 5 foldów C, 6 konfiguracji eksperymentu) po kilkanaście–kilkadziesiąt epok na ~1 770 obrazach 224×224. Na GPU klasy T4 jedna epoka ResNet-18 to rząd kilkunastu sekund, więc całość mieści się w limicie 2 h GPU. Przy pracy na CPU ograniczamy liczbę epok i rozważamy feature extraction z buforowaniem cech, aby zmieścić się w 4 h. Faktyczne czasy będą mierzone i logowane w W&B.

## Rozważane alternatywy

| Alternatywa | Dlaczego nie jako domyślna |
|-------------|----------------------------|
| TensorFlow / Keras | Brak przewagi dla tego zadania; zespół planuje pracę w PyTorch, a temat wskazuje ResNet-18 dostępny w torchvision. |
| PyTorch Lightning / fastai | Gotowe pętle treningowe ukrywają k-fold i logikę treningu; temat wymaga własnej implementacji i zakazuje gotowych pipeline'ów pokrywających cały etap. |
| `sklearn.KFold` / `StratifiedKFold` | Wprost zakazane w treści tematu. |
| Większe modele (ResNet-50, ViT) | Większe ryzyko przeuczenia na 2,5 tys. zdjęć i przekroczenia budżetu obliczeniowego. |
| Gotowe wagi wytrenowane na TrashNet | Zakazane; dozwolone wyłącznie wagi ImageNet. |
| Gradio / Streamlit do demo | Temat wymaga notebooka `.ipynb`. |

## Konsekwencje

**Pozytywne**

- Spójny, prosty stos (PyTorch + torchvision) pokrywa wszystkie wymagania bez ciężkich zależności.
- Własna pętla treningu i k-fold daje pełną kontrolę i spełnia wymóg samodzielnej implementacji.
- W&B zapewnia porównywalność runów i wymagany publiczny dashboard.

**Negatywne i ryzyka**

- **Wariancja na małym zbiorze:** wyniki pojedynczego podziału mogą być mylące; łagodzimy przez k-fold i raportowanie odchylenia standardowego.
- **Przeuczenie modelu od zera:** wariant B prawdopodobnie nie osiągnie progu 0,80; to zamierzony wynik, który trzeba opisać i wyjaśnić.
- **Przesunięcie domeny:** TrashNet ma jednolite tło i pojedynczy obiekt, więc wysoki wynik nie przenosi się automatycznie na zdjęcia z taśmy sortowniczej.
- **Rzadka klasa `trash`:** niski recall tej klasy może samodzielnie obniżyć macro-F1 poniżej progu.
- **Determinizm na MPS / CUDA:** pełna powtarzalność nie zawsze jest gwarantowana; dokumentujemy środowisko i wersje bibliotek.
