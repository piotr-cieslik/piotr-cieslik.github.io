---
title: "MLContext"
date: 2020-03-09
categories: [C#, ML.NET]
---

Aby zrozumieć w jaki sposób efektywnie wykorzystywać bibliotekę *ML.NET* niezbędne jest zapoznanie się z klasą `MLContext` stanowiącą punkt wyjścia dla operacji na danych.

*Wpis został stworzony na bazie ML.NET w wersji 1.4.*

Cechą, która na wstępie odróżnia bibliotekę ML.NET od innych, znanych mi, narzędzi pozwalających na wykorzystanie algorytmów uczenia maszynowego jest konieczność stworzenia instancji kontekstu.

```csharp
var mlContext = new MlContext();
```

Stworzony kontekst staje się dla nas punktem wyjścia przy definiowaniu dalszych operacji na danych. Przyznam, że początkowo miałem problem ze zrozumieniem dlaczego Microsoft zdecydował się na takie rozwiązanie. Jednak po zapoznaniu się z dokumentacją oraz dostępnym na GitHub kodem, decyzja ta stała się trochę bardziej zrozumiała.

Okazuje się, że instancja kontekstu ma trzy główne zadania:
- Jest fasada umożliwiająca dostęp do operacji na danych. Operacje pogrupowane są w tematyczne katalogi np. Regresja, Klasyfikacja, itd. co pozwala na ich sprawne przeszukiwanie.
- Zapewnia wspólne środowisko dla wykonywanych operacji. Jest to przydatne gdy chcemy aby nasz kod zachowywał się w sposób deterministyczny.
- Umożliwia podgląd logów z operacji utworzonych z jego wykorzystaniem.

Poniżej przedstawię rozwinięcie każdego z wymienionych zadań.

## MLContext jako fasada
Wspomniane wcześniej operacje na danych pogrupowane są w tzw. katalogi (ang. catalog) oraz wystawione jako publiczne właściwości (ang. property) klasy `MlContext`.

Warto w tym miejscu zauważyć, że ponieważ nie działamy na statycznej klasie `MlContext`, a na jej instancji, możliwym staje się definiowanie własnych Extension Method, które pozwolą nam rozbudowywać wachlarz dostępnych opcji.

Domyślnie kontekst udostępnia nam następujące katalogi:

```csharp
public BinaryClassificationCatalog BinaryClassification { get; }

public MulticlassClassificationCatalog MulticlassClassification { get; }

public RegressionCatalog Regression { get; }

public ClusteringCatalog Clustering { get; }

public RankingCatalog Ranking { get; }

public AnomalyDetectionCatalog AnomalyDetection { get; }

public ForecastingCatalog Forecasting { get; }

public TransformsCatalog Transforms { get; }

public ModelOperationsCatalog Model { get; }

public DataOperationsCatalog Data { get; }

/// <summary>
/// This is a catalog of components that will be used for model loading.
/// </summary>
public ComponentCatalog ComponentCatalog => _env.ComponentCatalog;
```

Przykładowo katalog `DataOperationsCatalog` zawiera zbiór operacji niezbędnych do wczytania danych z różnych źródeł. Wczytane dane zostają zwrócone w postaci interfejsu `IDataView` będącym podstawową strukturą danych w ML.NET.

Przykład wczytania danych z pliku tekstowego "data.txt":
```csharp
var data = mlContext.Data.LoadFromTextFile<Data>("data.txt");
```

## MLContext jako współdzielone środowisko
Ponieważ część operacji zdefiniowanych przez ML.NET zakłada losowy charakter zjawisk, podczas tworzenia instancji klasy `MlContext` możemy przekazać opcjonalny parametr `seed` pozwalający na kontrolę wspomnianej losowości. W momencie gdy parametr ten nie jest zdefiniowany, zostaje przypisana mu domyślna wartość `null`, a operacje zakładające losowy charakter zjawisk zachowują się niedeterministyczne. Oznacza to, że za każdym uruchomieniem mogą (i zapewne zwrócą) inny wynik. Zachowanie to sprawdzi się świetnie w środowisku produkcyjnym, jednak podczas nauki, szukania błędów, czy udostępniania materiałów szkoleniowych deterministyczne (przewidywalne i powtarzalne) zachowanie jest zdecydowanie bardziej oczekiwane. Aby wprowadzić determinizm do naszych operacji wystarczy nadać stałą wartość dla parametru `seed`, najczęściej będzie to wartość 0.

Przykłady niedeterministycznych kontekstów:
```csharp
var nondeterministicContext1 = MLContext();
var nondeterministicContext2 = MLContext(null);
```

Przykłady deterministycznych kontekstów:
```csharp
var deterministicContext1 = MlContext(0);
var deterministicContext2 = MlContext(1);
```

Aby lepiej zrozumieć czym jest "środowisko" definiowane przez klasę `MLContext`, poniżej zamieszczam kod jej konstruktora:

```csharp
public MLContext(int? seed = null)
{
  _env = new LocalEnvironment(seed);
  _env.AddListener(ProcessMessage);
  
  BinaryClassification = new BinaryClassificationCatalog(_env);
  MulticlassClassification = new MulticlassClassificationCatalog(_env);
  Regression = new RegressionCatalog(_env);
  Clustering = new ClusteringCatalog(_env);
  Ranking = new RankingCatalog(_env);
  AnomalyDetection = new AnomalyDetectionCatalog(_env);
  Forecasting = new ForecastingCatalog(_env);
  Transforms = new TransformsCatalog(_env);
  Model = new ModelOperationsCatalog(_env);
  Data = new DataOperationsCatalog(_env);
}
```

## MLContext jako źródło logów.
Logi są jednym z podstawowych narzędzi pozwalających na śledzenie co dokładnie wydarzyło się w naszym programie. W przypadku biblioteki ML.NET źródłem logów może stać się dowolna instancja kontekstu. Wystarcz w tym celu zarejestrować funkcję, o odpowiedniej sygnaturze, pod właściwość `Log`. W ten sposób otrzymamy logi zebrane ze wszystkich operacji zdefiniowanych z wykorzystaniem naszej instancji kontekstu.

Przykład logowania do konsoli:
```csharp
var mlContext = new MlContext();
mlContext.Log += (sender, e) => Console.WriteLine(e.Message);
```

Pojedynczy log reprezentowany jest przez klasę `LoggingEventArgs` wystawiającą jedynie jedną publiczną właściwość typu `string` o nazwie `Message`.

W nowszej wersji bilbioteki *1.5.0-preview* klasa `LoggingEventArgs` rozbudowana została o dodatkowe właściwości zawierające więcej informacji na temat źródła logów.

## Podsumowanie
Mam nadzieję, że powyższy artykuł pomógł Ci choć trochę zrozumieć czym jest klasa `MLContext` oraz dlaczego niezbędne jest stworzenie jej instancji.

Poniżej zamieszczam link do źródeł, na których bazowałem pisząc ten artykuł oraz link do mojego repozytorium na GitHub, w którym znajdziesz przykłady wykorzystania biblioteki ML.NET.

---
Linki:
- [https://docs.microsoft.com/en-us/dotnet/api/microsoft.ml.mlcontext](https://docs.microsoft.com/en-us/dotnet/api/microsoft.ml.mlcontext)
- [https://github.com/dotnet/machinelearning/blob/master/src/Microsoft.ML.Data/MLContext.cs](https://github.com/dotnet/machinelearning/blob/master/src/Microsoft.ML.Data/MLContext.cs)
- [https://github.com/piotr-cieslik/MachineLearning](https://github.com/piotr-cieslik/MachineLearning)