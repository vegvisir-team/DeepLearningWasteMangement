# Automatyczna klasyfikacja odpadów (segregacja ze zdjęcia)

Projekt zaliczeniowy z Deep Learningu — temat 5. Klasyfikacja zdjęć odpadów do jednej z sześciu frakcji surowcowych na małym, niezbalansowanym zbiorze TrashNet.

## Skład zespołu

| Imię i nazwisko | Nr albumu | E-mail |
|-----------------|-----------|--------|
| Max Duda        | `<TODO>`  | `<TODO>` |
| Monika Duda     | `<TODO>`  | `<TODO>` |

> Praca wykonana samodzielnie; użycie AI (lub jego brak) jest deklarowane w pliku [AI-UZYCIE.md](AI-UZYCIE.md). Uczestnicy: Max Duda, `<nr albumu>`, `<e-mail>`; Monika Duda, `<nr albumu>`, `<e-mail>`.

Deklaracja zakresu użycia AI: [deklaracja.md](deklaracja.md).

## Temat i zakres projektu

**Typ zadania:** klasyfikacja obrazów (jedna etykieta na zdjęcie), mały i niezbalansowany zbiór danych.

**Dataset:** [TrashNet](https://github.com/garythung/trashnet) — 2 527 zdjęć, 6 klas, rozdzielczość 512×384, pojedynczy obiekt na jednolitym tle. Wersja przeskalowana (41 MB) dostępna na [Kaggle](https://www.kaggle.com/datasets/feyzazkefe/trashnet).

| Klasa     | Liczba zdjęć | Udział |
|-----------|-------------:|-------:|
| paper     | 594 | 23,5% |
| glass     | 501 | 19,8% |
| plastic   | 482 | 19,1% |
| metal     | 410 | 16,2% |
| cardboard | 403 | 15,9% |
| trash     | 137 |  5,4% |
| **Razem** | **2 527** | 100% |

**W zakresie projektu:**

- wczytanie danych i analiza eksploracyjna (liczności, rozkład klas, rozmiary obrazów, siatka przykładów),
- baseline: ResNet-18 z wagami ImageNet na wejściu 224×224,
- warianty porównawcze na tym samym podziale danych: ResNet-18 trenowany od zera oraz transfer learning z 5-krotną walidacją krzyżową (własna implementacja pętli k-fold),
- eksperyment: wpływ augmentacji przy 100%, 50% i 25% danych treningowych,
- analiza błędów: macierz pomyłek, recall per klasa, przykłady błędnych predykcji, omówienie przyczyn (szczególnie pary przezroczysty plastik ↔ szkło oraz papier ↔ karton),
- wizualizacje zapisane w repozytorium (krzywe uczenia, macierze pomyłek, przykładowe predykcje),
- logowanie eksperymentów do publicznego dashboardu Weights & Biases,
- notebook demonstracyjny `.ipynb`.

**Poza zakresem:**

- detekcja i segmentacja wielu obiektów na jednym zdjęciu,
- wdrożenie produkcyjne i integracja ze sprzętem sortowni,
- interfejsy Gradio / Streamlit (demo wyłącznie w notebooku),
- gotowe, wytrenowane modele z Kaggle lub innych repozytoriów (dozwolone są wyłącznie wagi ImageNet).

## Charakterystyka problemu

W sortowni każdy błąd klasyfikatora ma konkretny koszt: butelka PET uznana za papier trafia do belki makulatury i obniża wartość całej partii surowca, a w skrajnym przypadku powoduje jej odrzucenie przez odbiorcę. System musi więc dobrze rozpoznawać wszystkie frakcje, a nie tylko te najliczniejsze.

Trzy cechy zbioru TrashNet czynią zadanie trudnym:

1. **Mały zbiór.** 2 527 zdjęć to niewiele dla sieci konwolucyjnej. Pojedynczy podział train/test daje wyniki różniące się o kilka punktów procentowych w zależności od ziarna losowości, dlatego wyniki trzeba uśredniać (k-fold lub wiele seedów).
2. **Niezbalansowanie.** Klasa `trash` ma 137 zdjęć, ponad czterokrotnie mniej niż `paper` (594). Sama accuracy maskuje słabe rozpoznawanie rzadkich klas, więc metryką wiodącą jest macro-F1.
3. **Wizualnie podobne pary klas.** Przezroczysty plastik i szkło mają podobne odbicia i przezierność; papier i karton różnią się głównie grubością i fakturą, często słabo widoczną na zdjęciu.

Taki zbiór dobrze oddaje realne warunki: dane zebrane ręcznie przez operatora instalacji, a nie miliony zdjęć z internetu.

## Główny cel projektu

Zbudować i rzetelnie porównać klasyfikatory zdjęć odpadów, osiągając **macro-F1 ≥ 0,80 na zbiorze testowym** (lub udokumentować, co blokuje osiągnięcie progu), oraz **zmierzyć, w jakim stopniu augmentacja danych kompensuje mniejszą liczbę zdjęć treningowych**.

Cele szczegółowe:

- pokazać różnicę między transfer learningiem a treningiem od zera na małym zbiorze,
- uzyskać wiarygodne liczbowo wyniki dzięki walidacji krzyżowej,
- wyjaśnić, dlaczego model myli konkretne pary klas i czy da się to poprawić.

## Wymagania twarde (z treści tematu)

- [ ] Podział danych 70/15/15 (train / walidacja / test).
- [ ] Seed treningu 6102 (numpy i torch), identyczny dla wszystkich wariantów.
- [ ] Próg metryki głównej (macro-F1): 0,80 lub udokumentowany wniosek, dlaczego jest nieosiągalny.
- [ ] Własnoręcznie zaimplementowana pętla k-fold cross-validation (bez `sklearn.KFold`).
- [ ] Wagi wyłącznie ImageNet; brak gotowych modeli z Kaggle / repozytoriów.
- [ ] Budżet obliczeniowy: do 2 h GPU lub 4 h CPU na cały projekt.
- [ ] Metryki: accuracy, macro-F1, macierz pomyłek, recall per klasa.
- [ ] Wandb: publiczny dashboard, link do run-u w README.
- [ ] Demo w notebooku `.ipynb`.
- [ ] Repozytorium bez `__pycache__/`, wag `.pt`/`.pth` i surowych danych.
- [ ] Wnioski: co działało i dlaczego, ograniczenia, co zrobilibyśmy inaczej, odniesienie do praktyki.

## Dokumentacja

- [ADR 0001 — technologia i metody](docs/adr/0001-technologia-i-metody.md)
- [Przypadki użycia](docs/use-cases.md)
- [Treść tematu (PDF)](docs/temat_05_klasyfikacja-odpadow.pdf)

## Planowana struktura repozytorium

```
.
├── README.md
├── AI-UZYCIE.md
├── deklaracja.md
├── docs/            # dokumentacja, ADR, use case'y, treść tematu
├── src/             # kod: dane, modele, trening, k-fold, metryki
├── notebooks/       # EDA i notebook demonstracyjny
├── figures/         # wykresy zapisywane do repozytorium
└── data/            # surowe dane (ignorowane przez git)
```

## Uruchomienie

`<TODO>` — instrukcja krok po kroku dla GPU i CPU (instalacja zależności, pobranie danych, trening, ewaluacja).

## Wyniki

- Dashboard Weights & Biases: `<TODO: link>`
- Tabela wyników i wnioski: `<TODO>`
