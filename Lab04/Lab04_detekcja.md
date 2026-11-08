### Laboratorium 4, część 1: Detekcja obiektów metodami klasycznymi

#### Zadania

##### Wykonanie detekcji i segmentacji

- potok przetwarzania przyjmujący obraz kolorowy w przestrzeni BGR/RGB a zwracający kolekcję obiektów opisanych 
  - (1) prostokątem otaczającym (_bounding box_) - na potrzeby detekcji, oraz 
  - (2) maską binarną **lub** konturem (tj. zbiorem koordynat punktów brzegowych).
- Potok przetwarzania powinien obejmować binaryzację obrazu kolorowego (znanymi Ci już metodami), 
- detekcję komponentów spójności ([`cv2.findContours`](https://docs.opencv.org/4.x/d3/dc0/group__imgproc__shape.html#gadf1ad6a0b82947fa1fe3c3d497f260e0)), oraz 
- analizę blobów binarnych (obiektów).
- Wykryte obiekty należy przefiltrować tak, aby pozostawić wyłącznie _obiekty zainteresowania_, tj. pojedyncze larwy. Należy więc usunąć:
  * drobne obiekty niebędące larwami,
  * instancje złączone,
  * wszelkie inne zakłócenia.
- Zaimplementuj również funkcję pozwalającą na zaprezentowanie wyników detekcji w formie 
  - rysowania prostokątów odpowiadających instancjom
  - segmentacji w formie zaznaczania pikseli należących do instancji (może być w jednej funkcji z opcją wyboru).
- **Następnie** opracujesz moduł przetwarzania poszczególnych instancji na poziomie masek binarnych lub konturów. Moduł ten ma przyjmować pojedynczą instancję i wyznaczać dla niej podstawowe informacje, takie jak:
  * długość (skorzystaj z [`skimage.morphology.skeletonize`](https://scikit-image.org/docs/stable/api/skimage.morphology.html#skimage.morphology.skeletonize)) - relatywnie łatwa do _estymacji_, ale dość trudna do precyzyjnego _pomiaru_,
  * grubość - może stanowić wyzwanie,
  * klasyfikację kształtu (I-kształtne, S-kształtne, C-kształtne) -- potencjalnie trudne zadanie,
  * ...

#### Detekcja
##### Etap 1: segmentacja zgrubna (Fig. 4a)
- Konwersja obrazu BGR → HSV.
- **Globalny próg na kanale S.** Wartość progu dobrana **eksperymentalnie** (ręcznie, jedna na cały zbiór).
- Wynik: binarny obraz z grubymi blobami, zawierający larwy, ale też drobne zanieczyszczenia.

##### Etap 2: filtracja po polu i wyznaczenie ROI
- Odrzucenie obiektów o polu **mniejszym niż pojedyncza larwa** (śmieci, drobiny, resztki odchodów).
- Dla każdego pozostałego obiektu wyznaczany jest **prostokątny obszar zainteresowania (ROI)**, czyli bounding box (Fig. 4a pokazuje te prostokąty).

Ten krok to odpowiednik `findContours` + `boundingRect` + filtr po `contourArea` w Twoim zadaniu.

##### Etap 3: segmentacja lokalna w ROI (Fig. 4b)
- W **każdym ROI osobno** ponownie progowanie kanału S, ale teraz próg jest **adaptacyjny, wyznaczany z lokalnego histogramu** tego fragmentu.
- Sensowność: w małym wycinku histogram jest wyraźnie dwumodalny (tło vs larwa), więc próg można dobrać dokładnie dla tej konkretnej larwy, zamiast jednego kompromisowego progu globalnego. Typowy kandydat to metoda **Otsu** (artykuł nie podaje nazwy algorytmu, tylko ogólnie "na podstawie lokalnego histogramu").
- Wynik: **wstępne maski binarne** (Fig. 4b), dokładniejsze niż zgrubne, ale wciąż z resztkami cienia na brzegach.

##### Etap 4: adaptacyjna erozja krawędzi (Fig. 4c)
To najbardziej nietypowy element potoku.
- Zamiast stałej erozji (która skróciłaby też i ścieńczyła larwę wszędzie jednakowo), **głębokość erozji jest różna dla różnych punktów brzegu**.
- Wykorzystany jest kanał **B**.
- Dla każdego punktu na krawędzi maski liczy się **różnicę między**:
  - średnią wartością kanału B w jądrze (oknie) obejmującym najbliższe sąsiedztwo punktu brzegowego,
  - średnią wartością kanału B w całym obiekcie.
- Jeśli sąsiedztwo punktu brzegowego jest wyraźnie jaśniejsze od wnętrza larwy (jest to piksel cienia/tła, a nie larwy), erozja w tym miejscu jest głębsza; jeśli zbliżone do wnętrza, erozja jest płytka lub zerowa.
- Efekt: usunięcie "obwódki" cienia przy zachowaniu rzeczywistego kształtu, w tym końców larwy (Fig. 4c).

##### Etap 5: kontrola wizualna i korekta ręczna
- Końcowe maski były **przeglądane wzrokowo** i w razie potrzeby poprawiane ręcznie.
- Przykład podany w tekście: **rzadkie przypadki nakładających się larw usuwano ręcznie** ze zbioru.

To ważne: autorzy **nie** mają w potoku automatycznego wykrywania złączonych instancji, robią to człowiekiem. Twoje zadanie wymaga, żebyś to zautomatyzował.

#### Oczekiwane rezultaty
1. Zaprojektowanie potoku przetwarzania do detekcji i segmentacji - skonstruuj go najlepiej jako pojedynczą funkcję, ale pozostaw ślad z dokonywanych po drodze eksperymentów, tak aby możliwe było przeanalizowanie przebiegu operacji krok po kroku.
2. Zadbaj o możliwość przetworzenia tylko wybranego RoI z obrazu, aby można było przyjrzeć się "z bliska" wybranym rejonom (np. trudniejszym) -- rekomenduję zacząć pracę od wybrania mniejszego, wygodniejszego w przetwarzaniu rejonu.
3. Działanie modułu analizy zademonstruj na kilku wybranych przykładach, także umożliwiając podgląd operacji krok po kroku.
4. Debugging - jeśli na którymkolwiek etapie pojawią się problemy, których nie potrafisz rozwiązać, postaraj się przynajmniej przeanalizować, co je powoduje.

#### Literatura
1. [`cv2.findContours`](https://docs.opencv.org/4.x/d3/dc0/group__imgproc__shape.html#gadf1ad6a0b82947fa1fe3c3d497f260e0), [OpenCV Structural Analysis and Shape Descriptors](https://docs.opencv.org/4.x/d3/dc0/group__imgproc__shape.html), [OpenCV Drawing Functions](https://docs.opencv.org/4.x/d6/d6e/group__imgproc__draw.html)
2. Publikacja, której częścią był potok przetwarzania podobny do tego, który zaimplementujesz: https://ieeexplore.ieee.org/document/10924163