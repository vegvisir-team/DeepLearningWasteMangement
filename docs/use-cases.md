# Przypadki użycia

Scenariusze, które ma wspierać klasyfikator zdjęć odpadów. Każdy opisuje aktora, sytuację, oczekiwany wynik, miarę sukcesu oraz wymaganie projektu, które go weryfikuje. Klasy: `glass`, `paper`, `cardboard`, `plastic`, `metal`, `trash`.

## UC1. Klasyfikacja przedmiotu na linii sortowniczej

- **Aktor:** system sortowni (MRF), kamera nad taśmą, operator linii.
- **Kontekst:** przedmiot przejeżdża pod kamerą; system musi zdecydować, do którego strumienia surowca go skierować.
- **Przebieg:**
  1. Kamera wykonuje zdjęcie pojedynczego przedmiotu.
  2. Model zwraca klasę i pewność predykcji.
  3. Sterownik kieruje przedmiot do odpowiedniego strumienia.
- **Oczekiwany wynik:** przedmiot trafia do właściwej frakcji; plastik nie trafia do papieru (i odwrotnie).
- **Miara sukcesu:** macro-F1 ≥ 0,80; wysoki recall i precyzja dla `plastic` i `paper`; niski udział pomyłek plastik → papier w macierzy pomyłek.
- **Powiązanie z projektem:** baseline vs warianty, macierz pomyłek, recall per klasa.

## UC2. Kontrola jakości partii surowca

- **Aktor:** kontroler jakości w sortowni lub u odbiorcy surowca.
- **Kontekst:** przed sprzedażą belki makulatury lub szkła trzeba ocenić poziom zanieczyszczeń.
- **Przebieg:**
  1. Kontroler fotografuje losową próbkę przedmiotów z partii.
  2. Model klasyfikuje każde zdjęcie.
  3. System raportuje udział przedmiotów spoza deklarowanej frakcji.
- **Oczekiwany wynik:** szacunek czystości partii i wskazanie, które frakcje ją zanieczyszczają.
- **Miara sukcesu:** wysoka precyzja per klasa (fałszywe alarmy obniżają zaufanie do raportu); stabilność wyniku między foldami.
- **Powiązanie z projektem:** k-fold i raport średnia ± odchylenie, precyzja i recall per klasa.

## UC3. Podpowiedź segregacji dla mieszkańca

- **Aktor:** mieszkaniec korzystający z aplikacji edukacyjnej.
- **Kontekst:** mieszkaniec nie wie, do którego pojemnika wrzucić odpad (np. karton po soku, słoik, folia).
- **Przebieg:**
  1. Użytkownik robi zdjęcie odpadu.
  2. Model rozpoznaje frakcję.
  3. Aplikacja wskazuje pojemnik zgodny z lokalnymi zasadami segregacji.
- **Oczekiwany wynik:** poprawna podpowiedź pojemnika i krótkie wyjaśnienie.
- **Miara sukcesu:** wysoka accuracy na typowych przedmiotach; czytelna informacja przy niskiej pewności zamiast błędnej podpowiedzi.
- **Powiązanie z projektem:** notebook demonstracyjny z przykładowymi predykcjami i pewnością modelu.

## UC4. Inteligentny kosz z lokalną inferencją

- **Aktor:** kosz z kamerą i prostym mikrokomputerem (np. Raspberry Pi), zarządca budynku.
- **Kontekst:** kosz automatycznie otwiera właściwą komorę po rozpoznaniu odpadu; brak stałego dostępu do serwera.
- **Przebieg:**
  1. Kamera w koszu robi zdjęcie wrzucanego przedmiotu.
  2. Lekki model działa lokalnie na CPU.
  3. Kosz otwiera komorę odpowiadającą rozpoznanej frakcji.
- **Oczekiwany wynik:** decyzja w czasie poniżej sekundy przy akceptowalnej jakości.
- **Miara sukcesu:** macro-F1 bliskie wariantowi bazowemu przy mniejszym modelu; czas inferencji na CPU.
- **Powiązanie z projektem:** opcjonalne porównanie z lżejszą architekturą (MobileNetV3 / EfficientNet-B0), budżet CPU.

## UC5. Odrzucenie niepewnej predykcji do ręcznej weryfikacji

- **Aktor:** system sortowni, pracownik stanowiska ręcznego sortowania.
- **Kontekst:** dla trudnych przedmiotów (przezroczysty plastik vs szkło, cienki karton vs papier) model jest niepewny.
- **Przebieg:**
  1. Model zwraca rozkład prawdopodobieństw.
  2. Jeśli najwyższa pewność jest poniżej progu, przedmiot trafia na stanowisko ręczne.
  3. Pracownik sortuje przedmiot; zdjęcie i etykieta mogą zasilić przyszły zbiór treningowy.
- **Oczekiwany wynik:** mniej błędów w strumieniach surowca kosztem niewielkiej części przedmiotów sortowanych ręcznie.
- **Miara sukcesu:** wzrost precyzji na przedmiotach zaakceptowanych przy kontrolowanym odsetku odrzuceń.
- **Powiązanie z projektem:** analiza błędów (pewność modelu przy pomyłkach, trudne pary klas), opcjonalny próg odrzucenia.

## UC6. Uruchomienie klasyfikatora w nowej sortowni z małym zbiorem

- **Aktor:** zespół wdrożeniowy, operator nowej instalacji.
- **Kontekst:** nowa sortownia ma inne oświetlenie i kamerę; operator może ręcznie oznaczyć tylko kilkaset zdjęć.
- **Przebieg:**
  1. Operator zbiera i etykietuje niewielki zbiór lokalnych zdjęć.
  2. Zespół dostraja model wstępnie wytrenowany na ImageNet z augmentacją.
  3. Model jest oceniany walidacją krzyżową przed uruchomieniem.
- **Oczekiwany wynik:** użyteczny klasyfikator bez zbierania tysięcy zdjęć.
- **Miara sukcesu:** jak blisko jest wynik przy 25% / 50% danych z augmentacją do wyniku na 100% danych.
- **Powiązanie z projektem:** eksperyment augmentacja × wielkość zbioru (100% / 50% / 25%), porównanie transfer vs od zera.

## Ograniczenia przypadków użycia

- **Jednolite tło:** zdjęcia TrashNet mają czyste, jednolite tło. Na taśmie sortowniczej tło jest zabrudzone, a oświetlenie zmienne, więc wyniki na TrashNet są górnym oszacowaniem jakości w praktyce.
- **Jeden obiekt na zdjęciu:** projekt nie obejmuje detekcji wielu obiektów; UC1 i UC4 zakładają, że przedmioty są rozdzielone.
- **Sześć klas:** brak m.in. bioodpadów, elektroodpadów i odpadów niebezpiecznych; klasa `trash` jest zbiorcza i mało liczna.
- **Lokalne zasady segregacji:** mapowanie klas na pojemniki (UC3) zależy od gminy i nie jest częścią modelu.
- **Brak walidacji w terenie:** wszystkie miary sukcesu są liczone na zbiorze TrashNet, nie na danych z rzeczywistej instalacji.
