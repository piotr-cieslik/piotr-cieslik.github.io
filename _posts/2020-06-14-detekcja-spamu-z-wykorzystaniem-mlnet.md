---
title: "Detekcja spamu z wykorzystaniem ML.NET"
date: 2020-06-14
categories: [C#, ML.NET]
---
Problem detekcji oraz automatycznego usuwania spamu dotyczy coraz większej liczby aplikacji. Jeszcze do niedawna stwierdzenia typu "aplikacja wykorzystuje sztuczną inteligencję do detekcji spamu" lub "program uczy się czym jest spam na podstawie decyzji użytkownika" brzmiały dla mnie bardzo egzotycznie. W praktyce okazuje się, że stworzenie "inteligentnego" algorytmu detekcji spamu nie jest wcale takie trudne. W poniższym artykule przedstawię w jaki sposób napisać taki algorytm z wykorzystaniem biblioteki ML.NET oraz zaprezentuję wyniki jego działania na przykładowym zbiorze dany.

Zanim zaczniemy chciałbym zwrócić uwagę, że przedstawione poniżej rozwiązanie wymaga posiadania zbioru danych, na którym przeprowadzone zostanie uczenie naszego klasyfikatora. Jeśli nie dysponujesz własnymi danymi możesz spróbować wykorzystać jedne z publicznie dostępnych zbiorów, jednak uzyskane w ten sposób wyniki mogą być niezadowalające.

Niniejszy artykuł powstał na bazie biblioteki ML.NET w wersji 1.5.0.

Całość omawianego kodu dostępna jest na GitHub [pod tym adresem](https://github.com/piotr-cieslik/Blog.SpamDetection).
## Dane
W tym artykule posługuję się zbiorem "YouTube Spam Collection" zawierającym prawie 2000 tysięcy komentarzy pozostawionych w serwisie YouTube pod popularnymi teledyskami.

Pojedynczy komentarz reprezentowany jest przez wartości:
- id,
- autor,
- czas utworzenia,
- treść,
- etykieta (spam / nie spam).

Autorami zbioru są Tiago A. Almeida oraz Tulio C. Alberto.
Linki do zbioru danych:
- [https://archive.ics.uci.edu/ml/datasets/YouTube+Spam+Collection](https://archive.ics.uci.edu/ml/datasets/YouTube+Spam+Collection)
- [http://dcomp.sor.ufscar.br/talmeida/youtubespamcollection/](http://dcomp.sor.ufscar.br/talmeida/youtubespamcollection/)

## Krok 1: wczytanie danych
Podstawą pracy z biblioteką ML.NET jest utworzenie instancji kontekstu.

```csharp
var mlContext = new MLContext(seed: 0);
```

Mając utworzony kontekst zajmijmy się wczytaniem danych. Do tego zadania posłużymy się klasą `TextLoader`, której metoda `Load` pozwala na równoczesne wczytanie danych z kilku plików.

```csharp
var files =
    new[]
    {
        @"Data/youtube_01_psy.csv",
        @"Data/youtube_02_katy_perry.csv",
        @"Data/youtube_03_lmfao.csv",
        @"Data/youtube_04_eminem.csv",
        @"Data/youtube_05_shakira.csv",
    };
var textLoader =
    mlContext.Data.CreateTextLoader<Comment>(
        separatorChar: ',',
        hasHeader: true,
        allowQuoting: true,
        trimWhitespace: true);
var data =
    textLoader.Load(files);
```

Ponieważ do oceny jakości modelu posłużymy się [k-krotną walidacją krzyżową](https://en.wikipedia.org/wiki/Cross-validation_(statistics)), nie musimy dzielić w tym momencie zbioru danych na podzbiór uczący oraz testowy.

## Krok 2: definicja modelu
Podobnie jak w przypadku poprzednich artykułów nasz zbiór przekształceń na danych nazwiemy `pipeline`. Zaczniemy od pustego zbioru, do którego dodawać będziemy kolejne przekształcenia.

```csharp
var pipeline =
    new EstimatorChain<ITransformer>();
```

Naszym głównym wyzwaniem podczas tworzenia modelu jest obliczenie liczbowego wektora cech na podstawie treści komentarza. Do tego zadania posłuży nam metoda `FeaturizeText` znajdująca się w katalogu transformacji tekstu. Pod tą dość ogólną nazwą metody kryje się prawdziwy kombajn przekształceń.

Zaglądając do [dokumentacji](https://docs.microsoft.com/en-us/dotnet/api/microsoft.ml.transforms.text.textfeaturizingestimator?view=ml-dotnet) widzimy listę operacji, które mogą być wykonane w ramach jej działania:
- automatyczna detekcja języka;
- [tokenizacja tekstu](https://en.wikipedia.org/wiki/Lexical_analysis#Tokenization);
- [normalizacja tekstu](https://en.wikipedia.org/wiki/Text_normalization);
- usuwanie [stop words](https://en.wikipedia.org/wiki/Stop_words);
- obliczenie [n-gramów](https://en.wikipedia.org/wiki/N-gram) na bazie słów,
- obliczenie n-gramów na bazie znaków;
- obliczenie wag poszczególnych słów korzystając z miar [TF, IDF, TF-IDF](https://en.wikipedia.org/wiki/Tf%E2%80%93idf);
- [normalizacja wektora wyjściowego](https://en.wikipedia.org/wiki/Norm_(mathematics)#Euclidean_norm).

Jeśli powyższe przekształcenia niewiele Ci mówią, nie przejmuj się, jeszcze do niedawna były one czarną magią również dla mnie. Na szczęście domyśle parametry metody `FeaturizeText` dobrane są na tyle uniwersalnie, że być może wcale nie musisz ich zmieniać. W przypadku analizowanego zbioru danych, pozostawienie domyślnych parametrów skutkuje praktycznie identycznymi wynikami działania (średnia dokładność na poziomie 0.95). Jeśli więc nie interesują cię tajniki obliczania wektora cech z tekstu twój *pipeline* może wyglądać następująco:

```csharp
var pipeline =
    new EstimatorChain<ITransformer>()
        .Append(
            mlContext.Transforms.Text.FeaturizeText(
                outputColumnName: "Features",
                inputColumnName: nameof(Comment.Content));
```

Zachęcam jednak do przyjrzenia się możliwościom tej metody oraz sprawdzenie rezultatów klasyfikacji dla różnych konfiguracji. Zmiana domyślnych ustawień `FeaturizeText` obywa się przez przekazanie obiektu klasy `TextFeaturizingEstimator.Options` jako drugi. W moim przypadku zdecydowałem się na następującą konfigurację:

```csharp
var pipeline =
    new EstimatorChain<ITransformer>()
        .Append(
            mlContext.Transforms.Text.FeaturizeText(
                outputColumnName: "Features",
                options: new TextFeaturizingEstimator.Options
                {
                    CaseMode = TextNormalizingEstimator.CaseMode.Lower,
                    KeepDiacritics = false,
                    KeepNumbers = true,
                    KeepPunctuations = true,
                    StopWordsRemoverOptions = null,
                    CharFeatureExtractor = new WordBagEstimator.Options
                    {
                        NgramLength = 3,
                        UseAllLengths = false,
                        SkipLength = 0,
                        MaximumNgramsCount = null,
                        Weighting = NgramExtractingEstimator.WeightingCriteria.TfIdf,
                    },
                    WordFeatureExtractor = null,
                    Norm = TextFeaturizingEstimator.NormFunction.L2,
                    OutputTokensColumnName = null
                },
                inputColumnNames: nameof(Comment.Content));
```

Myślę, że komentarza wymagają następujące właściwości:
- `StopWordsRemoverOptions`: nie zdecydowałem się na usuwanie *stop words*, ponieważ nie jestem pewien języka w jakim napisany został komentarz.
- `CharFeatureExtractor`: okazuje się, że analiza wystąpień sekwencji znaków pozwoliła mi na osiągnięcie lepszych wyników klasyfikacji niż analiza sekwencji słów. Prawdopodobnie spam zawiera często linki do innych stron, które łatwo mogą być wykryte przez tego typu analizę.
- `WordFeatureExtractor`: ustawienie wartości `null` wyłącza ekstrakcję cech na podstawie sekwencji słów.

Mając zdefiniowane w jaki sposób obliczyć wektor cech, możemy zająć się uczeniem naszego modelu. Zanim to jednak zrobimy chciałbym zwrócić uwagę na pewną konwencję stosowaną w bibliotece ML.NET. Wszystkie algorytmy binarnej klasyfikacji zakładają, że wektor cech wejściowych znajduje się w kolumnie *Features*, natomiast parametr określający przynależność do danej klas określony jest w kolumnie *Label*. Trzymając się tej konwencji dodajmy do naszego *pipeline* przekształcenie, które skopiuje nam informację o klasie komentarza do kolumny *Label*:

```csharp
...
.Append(
    mlContext.Transforms.CopyColumns(
        outputColumnName: "Label",
        inputColumnName: nameof(Comment.Class)));
```

Następnie możemy przejść do definicji samego klasyfikatora. Ja posłużę się modelem regresji logistycznej, zachęcam jednak do poeksperymentowania i sprawdzenia również innych algorytmów. Ponieważ trzymam się standardowej konwencji nazewnictwa kolumn *Features*/*Label* nie muszę przekazywać żadnych dodatkowych parametrów do klasyfikatora.

```csharp
...
.Append(
    mlContext.BinaryClassification.Trainers.SdcaLogisticRegression()));
```

Zmiana metody `SdcaLogisticRegression` na `LinearSvm` pozwoli na przeprowadzenie klasyfikacji z wykorzystaniem algorytmu SVM. Zestawienie wyników jakie uzyskałem dla różnych algorytmów znajduję się w dalszej części artykułu.

## Krok 3: ocena modelu
Jak wspomniałem już wcześniej, do oceny modelu wykorzystamy k-krotną walidację krzyżową. Ja zdecydowałem się na przeprowadzenie 5-krotnej walidacji.

```csharp
var crossValidationResults =
    mlContext.BinaryClassification.CrossValidateNonCalibrated(
        data,
        pipeline,
        numberOfFolds: 5);
```

Uzyskane wyniki na każdym z etapów możemy wyświetlić bezpośrednio w konsoli:

```csharp
foreach (var result in crossValidationResults)
{
    Console.WriteLine($"Fold {result.Fold}, accuracy: {result.Metrics.Accuracy}");
}
```

Możemy też obliczyć ich średnią wartość:

```csharp
var meanAccuracy =
    crossValidationResults.Sum(x => x.Metrics.Accuracy) / crossValidationResults.Count();
```

W przypadku badanych przeze mnie klasyfikatorów otrzymałem następujące wyniki:

|   |  Iteracja 1 | Iteracja 2 | Iteracja 3 | Iteracja 4 | Iteracja 5 | Średnia |
| :--- | :--- | :--- | :--- | :--- | :--- | :---|
| Regresja logistyczna | 0,96 | 0,96 | 0.95 | 0.95 | 0.95 | 0.96 |
| SVM | 0,95 | 0,93 | 0.86 | 0.94 | 0.86 | 0.91 |
| Perceptron | 0,95 | 0,96 | 0.95 | 0.94 | 0.94 | 0.94 |

## Wykorzystanie modelu w praktyce
Skoro wiemy już w jaki sposób obliczyć wektor cech na podstawie treści komentarza oraz wybraliśmy algorytm klasyfikacji, przyjrzyjmy się w jaki wykorzystać nasz model w praktyce.

Załóżmy następujący scenariusz:
1. Dysponujemy komentarzami zebranymi jedynie dla czterech z pięciu artystów. Jest to nasz zbiór uczący.
1. Na podstawie zbioru uczącego tworzymy model wykrywający spam, następnie zapisujemy go do pliku.
2. Otrzymujemy nowe komentarze pozostawione pod teledyskami piątego z artystów. Jest to nasz zbiór testowy.
3. Wczytujemy stworzony wcześniej model oraz dokonujemy klasyfikacji nowych komentarzy.

Wczytujemy zbiór uczący:
```csharp
var trainFiles =
    files.Take(4).ToArray();
var trainData =
    textLoader.Load(trainFiles);
```

Tworzymy model predykcji na podstawie zdefiniowanego wcześniej *pipeline*:
```csharp
var model =
    pipeline.Fit(trainData);
```

Zapisujemy nasz model do pliku:
```csharp
mlContext.Model.Save(model, trainData.Schema, "spam-detection-model.zip");
```

Tworzymy zbiór danych testowych:
```csharp
var testFiles =
    files.Skip(4).Take(1).ToArray();
var testData =
    textLoader.Load(testFiles);
```

Wczytujemy model z pliku:
```csharp
var loadedModel =
    mlContext.Model.Load("spam-detection-model.zip", out var loadedInputSchema);
```

Przekształcamy nasz model w *prediction engine*, a następnie dokonujemy klasyfikacji 5 losowych komentarzy pochodzących z zbioru danych testowych.

```csharp
var predictionEngine =
    mlContext.Model.CreatePredictionEngine<Comment, Prediction>(loadedModel);
var random = new Random();
var samples =
    mlContext.Data.CreateEnumerable<Comment>(testData, false)
    .OrderBy(x => random.Next())
    .Take(5);
foreach (var sample in samples)
{
    var prediction = predictionEngine.Predict(sample);
    Console.Write("Comment:");
    Console.WriteLine(sample.Content);
    Console.WriteLine("Label: " + (prediction.Label ? "SPAM" : "OK"));
    Console.WriteLine("");
}
```

Przykładowe wyniki działania:

| Komentarz     | Kategoria |
| :--- | :--- |
| Subscribe and Win a CAP | SPAM |
| Hi. Check out and share our songs. | SPAM | 
| Beautiful song beautiful girl it works | OK | 
| ....I stil lisening this :) | OK |

