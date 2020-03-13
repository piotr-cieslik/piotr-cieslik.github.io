---
layout: post
title: "MLContext"
date: 2020-03-09
categories: ml.net
---

*Wpis został stworzony na bazie ML.NET w wersji 1.4*

Cechą, która na wstępie odróżnia bibliotekę ML.NET od innych bibliotek pozwalających na wykorzystanie algorytmów uczenia maszynowego, z którymi miałem styczność, jest konieczność stworzenia instancji kontekstu.

```
var mlContext = new MlContext();
```

Stworzony kontekst staje się dla nas punktem wyjścia przy definiowaniu dalszych operacji na danych. Początkowo miałem problem ze zrozumieniem dlaczego Microsoft zdecydował się na takie rozwiązanie, jednak po przejrzeniu dokumentacji oraz kodu z GitHuba decyzja ta stała się trochę bardziej zrozumiała. Okazuje się, że instancja kontekstu ma trzy główne zadania:
- Jest fasada umożliwiająca definiowanie operacji na danych (operacje pogrupowane są w tematyczne katalogi np. Regresja, Klasyfikacja, ...).
- Zapewnia, że operacji zdefiniowane za jego pomocą wykonywane są w wspólnym środowisku (wytłumaczenie poniżej).
- Umożliwia podgląd logów z operacji utworzonych z jego wykorzystaniem.

# MLContext jako fasada.
Wspomniane wcześniej operacje na danych pogrupowane są w tzw. katalogi (ang. catalog) oraz wystawione jako publiczne właściwości (ang. property) klasy `MlContext`. Warto w tym miejscu zauważyć, że ponieważ nie działamy na statycznej klasie `MlContext` a na jej instancji możliwym staje się definiowanie własnych *Extension Method*, które pozwolą nam rozbudowywać wachlarz dostępnych metod. Domyślnie kontekst udostępnia nam następujące katalogi:

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

# MLContext jako współdzielone środowisko.
Ponieważ część operacji zdefiniowanych przez ML.NET zakłada losowy charakter zjawisk, podczas tworzenia instancji klasy `MlContext` możemy przekazać opcjonalny parametr `seed`. Pozawala on na kontrolę wspomnianej losowości (pseudolosowości). W momencie gdy parametr ten nie jest zdefiniowany, zostaje przypisana mu domyślna wartość `null`, a operacje zakładające losowy charakter zjawisk zachowują się niedeterministyczne. Oznacza to, że za każdym uruchomieniem mogą (i zapewne zwrócą) inny wynik. Jest to jak najbardziej oczekiwane zachowanie, które nie zawsze dobrze się sprawdza, Przykładowo podczas nauki, szukania błędów, czy udostępniania materiałów szkoleniowych deterministyczne (przewidywalne i powtarzalne) zachowanie jest zdecydowanie bardziej oczekiwane. Aby wprowadzić determinizm do naszych operacji wystarczy nadać stałą wartość dla parametru `seed`, najczęściej będzie to wartość 0 lub 1.

```csharp
var nondeterministicContext1 = MLContext();
var nondeterministicContext2 = MLContext(null);

var deterministicContext1 = MlContext(0);
var deterministicContext2 = MlContext(0);
```

Aby lepiej zrozumieć w jaki sposób klasa `MLContext` definiuje środowisko dla operacji, zamieszczam poniżej pełny kod jej konstruktora.
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

# MLContext jako źródło logów.
Aby dokładniej zrozumieć jakie operację, oraz w jakiej kolejności, wykonywane są w obrębie naszego kontekstu klasa `MlContext` pozwala na zarejestrowanie się pod źródło logów. Pojedynczy log reprezentowany jest przez klasę `LoggingEventArgs`, wystawiającą jedynie jedną publiczną właściwość typu `string` o nazwie `Message`.

```
var mlContext = new MlContext();
mlContext.Log += (sender, e) => Console.WriteLine(e.Message);
```

W nowszej wersji bilbioteki *1.5.0-preview* klasa `LoggingEventArgs` rozbudowana została o dodatkowe właściwości określające źródło loga(logu?) oraz jego rodzaj (Trace, Info, Warning, Error).

---
Linki:
- https://docs.microsoft.com/en-us/dotnet/api/microsoft.ml.mlcontext
- https://github.com/dotnet/machinelearning/blob/master/src/Microsoft.ML.Data/MLContext.cs
- https://github.com/dotnet/machinelearning/blob/master/src/Microsoft.ML.Data/Utilities/LocalEnvironment.cs