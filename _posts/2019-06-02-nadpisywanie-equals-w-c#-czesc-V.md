---
layout: post
title: "Nadpisywanie Equals w C# - część V"
date: 2019-06-02
categories: C#
---

Artykuły wchodzące w skład serii:
1. Nadpisywanie Equals w C#, część I. Metoda Equals
2. Nadpisywanie Equals w C#, część II. Metoda GetHashCode
3. Nadpisywanie Equals w C#, część III. Implementacja metody Equals
4. Nadpisywanie Equals w C#, część IV. Implementacja metody GetHashCode
5. Nadpisywanie Equals w C#, część V. Porównanie wydajności różnych implementacji metody Equals

# Cel
W części pierwszej serii wspomniałem, że domyślna implementacja metody `Equals` dla typów wartościowych jest wolna, ponieważ bazuje na mechanizmie refleksji. Dziś chciałbym pokazać co oznacza stwierdzenie "wolna" i przedstawić dowody na to, że warto ją nadpisywać. 

Przedstawię również jak implementacja interfejsu `IEquatable<T>`, wpływa na czas porównywania obiektów. Dla przypomnienia dodam, że pozwala on na przyśpieszenie czasu porównywania, poprzez uniknięcie operacji *boxing* w przypadku typów wartościowych lub operacji rzutowania w przypadku typów referencyjnych.

Dla przedstawienia pełnego obrazu zaprezentuję również jak szybko porównywane są instancje typu anonimowego, typu `Tuple<>` oraz typu `ValueTuple<>`.

# Pomiary
Pomiary przeprowadzone zostały z wykorzystaniem biblioteki *BenchmarkDotNet* w wersji *v0.11.1* dla platformy *.NET Core 2.1*. Link do repozytorium zawierającego kod źródłowy programu, stworzonego na potrzeby tego tekstu, znajduje się na samym dole artykułu.

Porównywane typy to:
- struktura z domyślną implementacją metody `Equals`,
- struktura z nadpisaną metodą `Equals`,
- struktura z nadpisaną metodą `Equals`, implementująca `IEquatable<T>`,
- klasa z nadpisaną metodą `Equals`,
- klasa z nadpisaną metodą `Equals`, implementująca `IEquatable<T>`,
- typ anonimowy,
- typ `Tuple<>`,
- typ `ValueTuple<>`.

Wszystkie porównywane typy zawierają dwa pola:
- pole referencyjnego typu `string`,
- pole wartościowego typu `int`.

Sprawdzane metody to:
- instancyjna metoda `Equals(object obj)`,
- statyczna metoda `Equals(object objA, object objB)`,
- metoda `Equals(T x, T y)` dostarczona przez domyślny *equality comparer* dla danego typu.

*Domyślny equality comparer dla typu może zostać uzyskany przez wywołanie statycznej metody `Default` udostępnionej przez typ `EqualityComparer<T>`. Przykładowe porównanie dwóch wartości typu `int` z wykorzystaniem domyślnego equality comparer wygląda następująco: `EqualityComparer<int>.Default.Equals(1, 1)`.*

# Średnie czasy obliczeń
Poniższa tabela przedstawia średnie czasy wykonywania metody `Equals` uzyskane dla poszczególnych konfiguracji. Pełne wyniki pomiarów znajduje się w ostatniej sekcji artykułu.

|                              | Instancyjna metoda Equals | Statyczna metoda Equals | Domyślny EqualityComparer | 
|------------------------------|-------------------------:|------------------------:|--------------------------:| 
| Struktura - domyślny Equals  | 436.547 ns               | 436.385 ns              | 438.976 ns                | 
| Struktura - nadpisany Equals | 11.786 ns                | 16.274 ns               | 10.621 ns                 | 
| Struktura - IEquatable<T>    | 3.430 ns                 | 16.365 ns               | 2.386 ns                  | 
| Klasa - nadpisany Equals     | 5.198 ns                 | 6.352 ns                | 7.247 ns                  | 
| Klasa - IEquatable<T>        | 3.267 ns                 | 7.025 ns                | 7.007 ns                  | 
| Typ anonimowy                | 21.009 ns                | 22.801 ns               | -                         | 
| Tuple<T1, T2>                | 63.911 ns                | 64.414 ns               | 66.515 ns                 | 
| ValueTuple<T1, T2>           | 12.461 ns                | 36.526 ns               | 20.833 ns                 | 

# Analiza wyników
Zacznijmy od analizy struktur. Już na pierwszy rzut oka widać, że domyślna implementacja metody `Equals(object obj)`dla struktur jest zdecydowanie wolniejsza, niż jej ręcznie nadpisany odpowiednik. Sprytna, domyślna implementacja tej metody oparta została na mechanizmie refleksji. Mechanizm ten pozwala na odnalezienie wszystkich pól porównywanych typów, w czasie wykonywania programu (ang. *runtime*), a następnie sprawdzenia czy przechowywane w nich wartości są sobie równe.  Mówi się, że przedwczesna optymalizacja może być źródłem wielu problemów, w tym przypadku zaryzykuję stwierdzenie, że zawsze warto nadpisywać `Equals(object obj)` dla typów wartościowych.

Czas porównywania struktur ulega również skróceniu w momencie, gdy implementują one interfejs `IEquatable<T>`. Dzieje się tak, ponieważ unikamy w ten sposób kosztownego zjawiska *boxingu*. W przypadku wykorzystania statycznej metody `Equals(object objA, object objB)` wspomniany interfejs nie przyniesie nam żadnej korzyści, a zjawisko *boxingu* wystąpi każdorazowo.

Pozostając w temacie interfejsu `IEquatable<T>` przyjrzyjmy się jak wpływa on na klasy. Ponieważ każda instancja klasy jest obiektem, w momencie wykorzystania metody `Equals(object obj)`, nie jesteśmy skazani na *boxing*. Nie oznacza to jednak, że nie opłaca się implementować `IEquatable<T>` dla klas. W momencie wykorzystania statycznej metody `Equals(object objA, object objB)`, podobnie jak w przypadku struktur, nie zauważymy korzyści wynikających z implementacji omawianego interfejsu.

Porównując wyniki dla typów `Tuple<T1,T2>` oraz `ValueTuple<T1,T2>` zauważyć można sporą przewagę w prędkości obliczeń na korzyść drugiego z nich. Dzieje się tak między innymi dlatego, że `ValueTuple<T1,T2>` implementuje interfejs `IEquatable<T>`. Jest to niewątpliwie jedna z zalet typów `ValueTuple<>` nad typami `Tuple<>`.

Pewnym rozczarowaniem jest dla mnie prędkość porównywania `ValueTuple<T1,T2>` w stosunku do własnoręcznie stworzonej struktury implementującej `IEquatable<T>`. Dłuższy czas obliczeń, spowodowany jest prawdopodobnie narzutem związanym z tworzeniem domyślnego *equality comparer*, każdorazowo podczas wywołania metody `Equals`. Z kodu źródłowego udostępnionego na platformie GitHub przez Microsoft wynika, że aktualnie implementacja `Equals` dla `ValuTuple<T1,T2>` wygląda następująco:

```
public bool Equals(ValueTuple<T1, T2> other)
{
	return EqualityComparer<T1>.Default.Equals(Item1, other.Item1)
		&& EqualityComparer<T2>.Default.Equals(Item2, other.Item2);
}
```

Zadziwiająca może być również spora różnica pomiędzy czasem porównywania instancji klas z własnoręcznie nadpisaną metodą `Equal(object obj)` oraz instancji typów anonimowych. Podobnie jak w przypadku typu `ValueTuple<T1,T2>`, typ anonimowy tworzy domyślny *equality comparer* podczas każdego wywołania metody `Equal(object obj)`. Typy anonimowe nie implementują również interfejsu `IEquatable<T>`, co negatywnie wpływa na czas ich porównywania.


# Podsumowanie
Wyniki przeprowadzonego porównania były dość łatwe do przewidzenia. Uważam jednak, że zaprezentowanie konkretnych wartości liczbowych to zawsze dobry pomysł na potwierdzenie swoi słów. 

Reasumując:
- tworząc własną strukturę, w przypadku gdy będzie ona porównywana, najlepiej nadpisać metodę `Equals` oraz zaimplementować interfejs `IEquatable<T>`;
- tworząc własną klasę, w przypadku gdy będzie ona porównywana, lepiej implementować `IEquatable<T>`;
- typy `ValueTuple<>` porównywane są szybciej niż typy `Tuple<>`;
- porównanie z wykorzystanie domyślnego *equality comparer* dla typu jest szybsze niż porównanie za pomocą statycznej metody `Equals(object objA, object objB)`.

# Pełne wyniki badania
``` ini

BenchmarkDotNet=v0.11.1, OS=Windows 10.0.17134.285 (1803/April2018Update/Redstone4)
Intel Core i5-8250U CPU 1.60GHz (Max: 1.80GHz) (Kaby Lake R), 1 CPU, 8 logical and 4 physical cores
Frequency=1757815 Hz, Resolution=568.8881 ns, Timer=TSC
.NET Core SDK=2.1.402
  [Host] : .NET Core 2.1.4 (CoreCLR 4.6.26814.03, CoreFX 4.6.26814.02), 64bit RyuJIT
  Core   : .NET Core 2.1.4 (CoreCLR 4.6.26814.03, CoreFX 4.6.26814.02), 64bit RyuJIT
Job=Core  Runtime=Core  
|                                                           Method |       Mean |     Error |    StdDev |        Max |        Min | Iterations |
|----------------------------------------------------------------- |-----------:|----------:|----------:|-----------:|-----------:|-----------:|
|                CompareStructureWithDefaultEqualsByInstanceMethod | 339.468 ns | 5.9223 ns | 5.5397 ns | 351.314 ns | 332.248 ns |      15.00 |
|               CompareStructureWithOverrideEqualsByInstanceMethod |  10.182 ns | 0.1382 ns | 0.1225 ns |  10.451 ns |  10.087 ns |      14.00 |
|          CompareStructureWithIEquatableInterfaceByInstanceMethod |   1.823 ns | 0.0089 ns | 0.0083 ns |   1.840 ns |   1.810 ns |      15.00 |
|                  CompareStructureWithDefaultEqualsByStaticMethod | 334.403 ns | 3.0337 ns | 2.5333 ns | 339.605 ns | 331.756 ns |      13.00 |
|                 CompareStructureWithOverrideEqualsByStaticMethod |  17.783 ns | 0.1256 ns | 0.1175 ns |  18.013 ns |  17.614 ns |      15.00 |
|            CompareStructureWithIEquatableInterfaceByStaticMethod |  18.265 ns | 0.3613 ns | 0.3203 ns |  18.722 ns |  17.810 ns |      14.00 |
|       CompareStructureWithDefaultEqualsByDefaultEqualityComparer | 351.716 ns | 2.3698 ns | 2.2167 ns | 356.125 ns | 347.777 ns |      15.00 |
|      CompareStructureWithOverrideEqualsByDefaultEqualityComparer |  21.439 ns | 0.4568 ns | 0.3815 ns |  22.184 ns |  20.995 ns |      13.00 |
| CompareStructureWithIEquatableInterfaceByDefaultEqualityComparer |  10.931 ns | 0.0066 ns | 0.0052 ns |  10.940 ns |  10.925 ns |      12.00 |
|                   CompareClassWithOverrideEqualsByInstanceMethod |   4.503 ns | 0.0231 ns | 0.0205 ns |   4.542 ns |   4.467 ns |      14.00 |
|              CompareClassWithIEquatableInterfaceByInstanceMethod |   3.265 ns | 0.0080 ns | 0.0071 ns |   3.278 ns |   3.255 ns |      14.00 |
|                     CompareClassWithOverrideEqualsByStaticMethod |   6.427 ns | 0.0565 ns | 0.0529 ns |   6.467 ns |   6.258 ns |      15.00 |
|                CompareClassWithIEquatableInterfaceByStaticMethod |   6.749 ns | 0.0352 ns | 0.0329 ns |   6.791 ns |   6.693 ns |      15.00 |
|          CompareClassWithOverrideEqualsByDefaultEqualityComparer |   5.275 ns | 0.0154 ns | 0.0136 ns |   5.301 ns |   5.256 ns |      14.00 |
|     CompareClassWithIEquatableInterfaceByDefaultEqualityComparer |   5.927 ns | 0.1020 ns | 0.0904 ns |   6.112 ns |   5.844 ns |      14.00 |
|                            CompareAnonymousTypesByInstanceMethod |  10.811 ns | 0.0239 ns | 0.0199 ns |  10.850 ns |  10.792 ns |      13.00 |
|                               CompareValueTuplesByInstanceMethod |  17.811 ns | 0.0927 ns | 0.0724 ns |  17.950 ns |  17.683 ns |      12.00 |
|                                    CompareTuplesByInstanceMethod |  54.533 ns | 0.9591 ns | 0.8971 ns |  56.209 ns |  53.605 ns |      15.00 |
|                              CompareAnonymousTypesByStaticMethod |  11.197 ns | 0.5329 ns | 0.6137 ns |  13.033 ns |  10.826 ns |      20.00 |
|                                 CompareValueTuplesByStaticMethod |  25.751 ns | 0.4937 ns | 0.4618 ns |  26.752 ns |  25.285 ns |      15.00 |
|                                      CompareTuplesByStaticMethod |  56.999 ns | 0.3764 ns | 0.3521 ns |  57.529 ns |  56.492 ns |      15.00 |
|                      CompareValueTuplesByDefaultEqualityComparer |  20.369 ns | 0.2340 ns | 0.2189 ns |  20.869 ns |  20.108 ns |      15.00 |
|                           CompareTuplesByDefaultEqualityComparer |  55.591 ns | 0.5643 ns | 0.5278 ns |  56.688 ns |  54.995 ns |      15.00 |
