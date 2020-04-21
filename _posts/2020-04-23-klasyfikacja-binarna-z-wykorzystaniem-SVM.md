---
title: "Klasyfikacja binarna z wykorzystaniem SVM"
date: 2020-04-23
categories: [C#, ML.NET]
---
Klasyfikacja danych jest jednym z podstawowych problemów, do rozwiązania których wykorzystać możemy Machine Learning. W poniższym artykule przedstawię w jaki sposób wykorzystać algorytm SVM do przeprowadzenia binarnej klasyfikacji danych z użyciem biblioteki ML.NET.

Na potrzeby tego artykułu zakładam, że posiadasz minimalną wiedzę na temat klasyfikacji danych oraz zasady działania algorytmu SVM. Jeśli pojęcia "klasyfikacja binarna" oraz "dane liniowo separowalne" nie są Ci obce, możesz śmiało czytać dalej. Jeśli tak nie jest, zachęcam do zapoznania się z dostępnym na YouTube filmem [Support Vector Machines, Clearly Explained!!!](https://www.youtube.com/watch?v=efR1C6CvhmE) z kanału StatQuest lub zerknięcie do [Wikipedii](https://en.wikipedia.org/wiki/Support-vector_machine), a następnie powrotu do artykułu.

*Całość omawianego kodu dostępna jest na GitHub [pod tym adresem](https://github.com/piotr-cieslik/MachineLearning/tree/master/MachineLearning.BinaryClassification.LinearSvm).*

## Dane
Wykorzystamy popularny zbiór danych Iris Dataset zawierający wymiary kielichów oraz płatków 3 różnych gatunków irysów. Nie jest to może najbardziej porywający zbiór danych, jednak idealnie nadaje się do omówienia problemu klasyfikacji.

*Omawiany zbiór danych pobrany został ze strony [https://archive.ics.uci.edu/ml/datasets/Iris](https://archive.ics.uci.edu/ml/datasets/Iris).*

Kolumny w naszym zbiorze to:
- sepal length (długość kielicha w cm)
- sepal width (szerokość kielicha w cm)
- petal length (długość płatka w cm)
- petal width (szerokość płatka w cm)
- class (informacja o gatunku kwiatu)

Zbiór ten ma kilka istotnych dla nas cech:
- Jest prosty, składa się z 150 obserwacji w 5 kolumnach.
- Wszystkie dane są kompletne, odpada problem z uzupełnianiem braków danych.
- Nie wymaga normalizacji danych.
- Każdy gatunek kwiatu reprezentowany jest przez dokładnie 50 pomiarów.
- Gatunek *setosa* jest liniowo separowalny od reszty.
- Gatunki *versicolor* oraz *virginica* nie są liniowo separowalne.

Poniżej zamieszczam wizualizacje relacji pomiędzy parami poszczególnych wartości w naszym zbiorze. W zależności od wiersza i kolumny, w którym znajduje się wykres, osie X i Y zawierają różne kombinację cech kwiatu: sepal length, sepal width, petal length oraz petal width. Kolorem oznaczone zostały gatunki kwiatów. Pozwoliłem sobie tutaj na delikatne oszustwo, wykresy stworzone zostały z wykorzystaniem R Studio, a nie platformy .NET.

![Wykresy uzyskane za pomocą funkcji pairs.](/assets/images/2020/iris_data_pairs.png)

Przyglądając się powyższym wykresom widać wyraźnie, że na kilku z nich kółka czarne nie mieszają się z kółkami o innym kolorze. Przyjrzyjmy się bliżej wykresowi pokazującemu zależność pomiędzy petal length a sepal length.

![Wykres zależności długości płatka do długości kielicha.](/assets/images/2020/iris_data_petal_to_sepal.png)

Na powyższym wykresie widać jak na dłoni, że kółka czarne, reprezentujące gatunek kwiatu *setosa*, nie mieszają się z kółkami o innych kolorach. O danych takich mówimy, że są liniowo separowalne w danej przestrzeni.

Ponieważ SVM jest klasyfikatorem liniowym, to właśnie na gatunków kwiatu "setosa" skupimy się w dalszej części tego artykułu.

## Krok 1: wczytanie danych
Na samym początku stwórzmy instancję kontekstu, którą wykorzystywać będziemy do definiowania dalszych operacji na danych.

```csharp
var mlContext = new MLContext(seed: 0);
```

W dalszej części artykułu szczególnie interesować będą nas trzy katalogi operacji udostępnione przez kontekst:
- katalog danych (`mlContext.Data`),
- katalog transformacji danych (`mlContext.Transforms`),
- katalog klasyfikacji binarnej (`mlContext.BinaryClassification`).

Jeśli nie wiesz czym jest klasa `MLContext` zachęcam do zapoznania się najpierw z [moim wcześniejszym artykułem](/2020/MLContext/) na jego temat.

Zajmijmy się teraz wczytaniem danych. W tym celu zdefiniujmy klasę `IrisData` reprezentującą pojedynczy wiersz z naszego zbioru danych, oraz  określmy za pomocą atrybutu `LoadColumn` w jaki sposób ML.NET powinien przeprowadzić mapowanie pomiędzy kolumnami pliku tekstowego a właściwościami naszej klasy. 

```csharp
public class IrisData
{
    [LoadColumn(0)]
    public float SepalLength { get; set; }

    [LoadColumn(1)]
    public float SepalWidth { get; set; }

    [LoadColumn(2)]
    public float PetalLength { get; set; }

    [LoadColumn(3)]
    public float PetalWidth { get; set; }

    [LoadColumn(4)]
    public string Class { get; set; }
}
```

Mając zdefiniowaną klasę `IrisData` skorzystajmy z funkcji `LoadFromTextFile`, aby wczytać dane z pliku.

```csharp
var irisData =
    mlContext.Data.LoadFromTextFile<IrisData>(
        "iris.data",
        separatorChar: ',',
        hasHeader: false);
```

Funkcja ta oprócz ścieżki do pliku pozwala na przekazanie dodatkowych parametrów niezbędnych do prawidłowego odczytu plików. W moim przypadku zmieniłem separator wartości na przecinek (domyślnie jest to tabulator), oraz doprecyzowałem, że mój plik nie posiada nagłówków kolumn. W wyniku otrzymamy obiekt typu `IDataView`, który jest podstawowym typem reprezentującym dane w bibliotece ML.NET.

Mając wczytane dane podzielmy je w proporcji 80/20 na dane uczące oraz testowe. Możemy to zrobić z wykorzystaniem metody `TrainTestSplit`. Jako rezultat otrzymamy prostą strukturę danych, składającą się jedynie z dwóch właściwości `TrainSet` oraz `TestSet`, obie typu `IDataView`.

```csharp
var data =
    mlContext.Data.TrainTestSplit(
        irisData,
        testFraction: 0.2);
```

## Krok 2: obróbka danych
W dokumentacji biblioteki ML.NET znajdziemy informacje, że do nauki binarnego klasyfikatora SVM zbiór danych musi zawierać następujące kolumny:
- Kolumnę z danymi wejściowymi, której każdy wiersz zawiera wektor elementów typu `Single` (czyli `float`). Kolumna ta zazwyczaj nosi nazwę "Features".
- Kolumnę etykietę danych typu `bool`, określającą do której klasy przynależy dany wiersz. Kolumna ta zazwyczaj nosi nazwę "Label".

Myślę, że warto w tym miejscu zauważyć, że nazwy kolumn "Features" oraz "Label" są jedynie sugerowanymi nazwami kolumn i nie musimy ich używać. Jednak trzymając się tej dość prostej i sensownej nomenklatury oszczędzimy sobie kłopotu (oraz kodu) z przekazywaniem informacji na temat nazw kolumn, na których mają być przeprowadzane dalsze operacje. 

Ponieważ nasze dane są kompletne, proces ich obróbki ograniczać będzie się jedynie do utworzenia dwóch wspomnianych kolumn.

W celu stworzenia kolumny "Features", zawierającej wektor cech wejściowych, wykorzystamy metodę `Concatenate` z katalogu transformacji danych. Metoda ta służy do połączenia wartości kilku kolumn tego samego typu w jedną kolumnę. Jej parametry to nazwa nowej kolumny oraz tablica kolumn, które chcemy ze sobą połączyć.

```csharp
var pipeline =
    mlContext.Transforms.Concatenate(
        "Features",
        nameof(IrisData.SepalLength),
        nameof(IrisData.SepalWidth),
        nameof(IrisData.PetalLength),
        nameof(IrisData.PetalWidth));
```

Zwróć proszę uwagę na nazwę zmiennej *pipeline*. Biblioteka ML.NET pozwala na łączenie następujących po sobie operacji na danych i definiować w ten sposób łańcuch przekształceń nazywany *pipeline*. W dalszej części artykułu zdefiniujemy kolejne przekształcenia, które dodawane będą do naszego łańcucha z wykorzystaniem metody `Append`.

Mając zdefiniowany wektor cech wejściowych zajmijmy się przekształceniem kolumny `Class`, zawierającej dane typu `string`, do postaci binarnych etykiet wymaganych przez nasz algorytm. Wiedząc, że nasz zestaw danych zawiera informacje jedynie o trzech gatunkach kwiatów, dodajmy trzy nowe kolumny typu `bool` do naszego naszego zbioru danych, z których każda reprezentować będzie jeden gatunek kwiatu. Następnie wybierzmy jedną z nich i przekopiujmy jej wartości do kolumny "Features".

Dodanie nowych kolumn zrealizujemy za pomocą metody `CustomMapping` z katalogu transformacji danych. Przyjmuje ona dwa generyczne parametry określające wejściowy i wyjściowy typ danych. Nie wiem dlaczego Microsoft nie zdecydował się w tym miejscu na wykorzystanie anonimowych typów danych. Mam jedynie nadzieje, że opcja ta zostanie wprowadzona w kolejnych wersjach biblioteki.

Naszym wejściowym typem danych będzie klasa `IrisData`, natomiast typem wyjściowym (reprezentującym nowe kolumny) będzie nowa klasa `IrisDataCalculated` zdefiniowana w następujący sposób:

```csharp
public sealed class IrisDataCalculated
{
    public bool Setosa { get; set; }

    public bool Versicolor { get; set; }

    public bool Virginica { get; set; }
}
```

Mając zdefiniowaną nową klasę, dodajmy kolejne przekształcenie do naszego *pipeline* wykorzystując do tego metodę `Append`.

```csharp
var pipeline =
    mlContext.Transforms.Concatenate(
        "Features",
        nameof(IrisData.SepalLength),
        nameof(IrisData.SepalWidth),
        nameof(IrisData.PetalLength),
        nameof(IrisData.PetalWidth))
    .Append(
        mlContext.Transforms.CustomMapping<IrisData, IrisDataCalculated>(
            (x, y) =>
            {
                y.Setosa = x.Class.Contains("setosa");
                y.Versicolor = x.Class.Contains("versicolor");
                y.Virginica = x.Class.Contains("virginica");
            },
            contractName: default));
```

Następnie wybierzmy jedną z kolumn i przekopiujmy jej wartości do kolumn "Label" korzystając z metody `CopyColumns` z katalogu transformacji danych. W naszym przypadku za etykiety danych posłuży zawartość kolumny "Setosa". Moglibyśmy w prawdzie połączyć to przekształcenie z przekształceniem mapowania i zdefiniować właściwość "Label" bezpośrednio w typie `IrisDataCalculated`. Z powodów dydaktycznych zdecydowałem się jednak na rozbicie tej operacji na dwa etapy.

Ostatecznie nasz *pipeline* wygląda następująco:
```csharp
var pipeline =
    mlContext.Transforms.Concatenate(
        "Features",
        nameof(IrisData.SepalLength),
        nameof(IrisData.SepalWidth),
        nameof(IrisData.PetalLength),
        nameof(IrisData.PetalWidth))
    .Append(
        mlContext.Transforms.CustomMapping<IrisData, IrisDataCalculated>(
            (x, y) =>
            {
                y.Setosa = x.Class.Contains("setosa");
                y.Versicolor = x.Class.Contains("versicolor");
                y.Virginica = x.Class.Contains("virginica");
            },
            contractName: default))
    .Append(
        mlContext.Transforms.CopyColumns("Label", nameof(IrisDataCalculated.Setosa)));
```

Podsumowując, po zastosowaniu powyższych przekształceń nasze dane reprezentowane są w postaci tabeli (typu `IDataView`) zawierającej następujące kolumny:
- SepalLength: `float`
- SepalWidth: `float`
- PetalLength: `float`
- PetalWidth: `float`
- Class: `string`
- Setosa: `bool`
- Versicolor: `bool`
- Virginica: `bool`
- Features: `Vector<float>`
- Label: `bool`

## Krok 3: nauka modelu
Mając przygotowane dane przejdźmy do zdefiniowania oraz nauki naszego klasyfikatora. Definiujemy go poprzez dodanie kolejnego etapu przetwarzania do naszego *pipeline*. Tym razem posłużymy się metodą `LinearSvm` dostępną w katalogu klasyfikacji binarnej. Wspomniana metoda pozwala nam określić dodatkowe parametry, na podstawie których odbywać będzie się nauka naszego algorytmu. Specjalnie zwracam na to uwagę, gdyż wrócimy jeszcze do parametrów tej metody w dalszej części artykułu. Zwróć uwagę, że trzymając się domyślnego nazewnictwa kolumn ("Features" oraz "Label") nie musimy ręcznie przekazywać ich nazw. Gdybyśmy zdecydowali się na inne nazwy kolumn, musielibyśmy podać jako parametry metody `LinearSvm`.

```csharp
var pipeline =
    mlContext.Transforms.Concatenate(
        "Features",
        nameof(IrisData.SepalLength),
        nameof(IrisData.SepalWidth),
        nameof(IrisData.PetalLength),
        nameof(IrisData.PetalWidth))
    .Append(
        mlContext.Transforms.CustomMapping<IrisData, IrisDataCalculated>(
            (x, y) =>
            {
                y.Setosa = x.Class.Contains("setosa");
                y.Versicolor = x.Class.Contains("versicolor");
                y.Virginica = x.Class.Contains("virginica");
            },
            contractName: default))
    .Append(
        mlContext.Transforms.CopyColumns("Label", "Setosa"))
    .Append(
        mlContext.Transforms.ApproximatedKernelMap("Features"))
    .Append(
        mlContext.BinaryClassification.Trainers.LinearSvm());
```

Mając tak zdefiniowany *pipeline* możemy rozpocząć trening naszego modelu. Do tego zadania posłuży nam metoda `Fit`, której parametrem jest zbiór danych uczących (typu `IDataView`). Przypomnę, że w naszym przypadku składa się on z 80% wszystkich danych.

```csharp
var model = pipeline.Fit(data.TrainSet);
```

Wynikiem działania metody `Fit` jest gotowy, nauczony model, który wykorzystać możemy do klasyfikacji naszych danych. Zanim jednak to zrobimy oceńmy skuteczność naszego modelu.

## Krok 4: ocena modelu
Do oceny jakości modelu posłużymy się [współczynnikiem dokładności (ang. accuracy)](https://pl.wikipedia.org/wiki/Dok%C5%82adno%C5%9B%C4%87_i_precyzja_metody_pomiaru) obliczonym na podstawie wyników klasyfikacji uzyskanych dla danych testowych. Aby tego dokonać posłużmy się metodą `Transform`. W wyniku jej działania otrzymamy nowy zbiór danych, który zawierać będzie dodatkową kolumnę o nazwie "PredictedLabel".

```csharp
 var prediction = model.Transform(data.TestSet);
```

Mając pojedynczą tabelę zawierającą zarówno wzorcowe klasy dla danych (kolumna "Label"), jak i te przypisane w wyniku działania naszego modelu (kolumna "PredicatedLabel"), możemy ocenić jakość naszego modelu. Wykorzystajmy w tym celu wybudowaną funkcję `EvaluateNonCalibrated`. Domyślnie funkcja ta porównuje wartości kolumny "PredictedLabel" z wartościami kolumny "Label" oraz zwraca obiekt z wynikami tego porównania. My skupimy się jedynie na właściwości `Accuracy` tego obiektu.

```csharp
var metrics = mlContext.BinaryClassification.EvaluateNonCalibrated(prediction);
Console.WriteLine($"Accuracy: {metrics.Accuracy}");
// Output:
// Accuracy: 0,6190476190476191
```

Aktualnie, nasz model cechuje się około 62% dokładnością, co nie jest wynikiem zadowalającym. Wiemy, że zbiór danych opisujący gatunek kwiatu Setosa jest liniowo separowalny, co oznacza że możemy oczekiwać zdecydowanie wyższej dokładności.

## Krok 5: dostosowania parametrów i ocena wyników
Wspomniałem wcześniej, że metoda `LinearSvm` pozwala na przekazanie kilku dodatkowych parametrów. Jednym z nich jest liczba iteracji procesu nauki. Domyślnie wartość ta ustawiona jest na 1. Sprawdźmy jak zachowa się nasz model, gdy zmienimy jej wartość na 10.

```csharp
mlContext.BinaryClassification.Trainers.LinearSv(numberOfIterations: 10)
```

Odpalamy ponownie nasz program i widzimy, że tym razem...
```csharp
Console.WriteLine($"Accuracy: {metrics.Accuracy}");
// Output:
// Accuracy: 1
```
... udało się uzyskać 100% dokładności klasyfikacji :)

Zanim przejdziemy do podsumowania wyników przedstawię jeszcze zestawienie, w jaki sposób nasz klasyfikator poradził sobie z klasyfikacją innych gatunków kwiatów. Pamiętajmy, że w przypadku pozostałych gatunków nie mamy pewności czy są one liniowo separowalne, przez co nie wiemy czy możliwym jest osiągnięcie 100% dokładności.

| Gatunek \ Iteracje | 1 | 10 | 100 | 1000 |
| --- | --: | --: | --: | --: |
| Setosa | 0,62 | 1,00 | 1,00 | 1,00 |
| Versicolor | 0,67 | 0,67 | 0,67 | 0,67 |
| Virginica | 0,28 | 0,95 | 0,81 | 0,95 |

## Podsumowanie
Udało nam się zdefiniować oraz wytrenować przyzwoicie działający klasyfikator. Mam nadzieję, że artykuł ten pomógł Ci chociaż w minimalnym stopniu zrozumieć sposób wykorzystania biblioteki ML.NET.

Jeszcze dwie uwagi na sam koniec:

1. Nazwy kolumn "Features" oraz "Label" są standardowymi nazwami kolumn również dla innych algorytmów klasyfikacji binarnej. Możesz z powodzeniem podmienić metodę `LinearSvm` na inną, przykładowo `AveragedPerceptron` i uzyskać w ten sposób inny klasyfikator.
1. W przyszłości pokażę w jaki sposób możemy "przeskoczyć" problem danych, które nie są liniowo separowalne i poprawić dokładność naszego klasyfikatora poprzez zwiększenie liczby wymiarów naszej przestrzeni cech.