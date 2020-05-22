---
title: "Obiektowy sposób na opis, klasa DescriptionOf..."
date: 2020-05-22
categories: [C#]
---
Praktycznie w każdym projekcie prędzej czy później istnieje potrzeba utworzenia zwięzłego opisu pewnych elementów systemu. Przykładowo może być to:
- połączenie imienia oraz nazwiska osoby;
- przedstawienie pierwszy 30 znaków wiadomości;
- przedstawienie w formie tekstowej okresu czasu od - do;
- zamiana wartości liczbowej lub `enum` na tekst.

Jeśli system jest mały, a opis danego elementu potrzebny jest jedynie w jednym miejscu, to sprawa jest prosta. Możemy skonstruować opis, łącząc ze sobą tekst, w miejscu gdzie jest nam on potrzebny. Jednak wraz z rozbudową systemu konieczne może stać się przedstawienie tego samego opisu w kilku rożnych miejsca. Właśnie tutaj zaczyna się problem.

W tym momencie kod tworzący opis zostaje często przeniesiony do statycznej metody typu `GetPersonDescription` zdefiniowanej wewnątrz statycznej klasy `PersonDescriber`, `Describers` lub (o zgrozo) `Helpers`. Rozwiązanie to niewątpliwie pomaga uniknąć problemu duplikacji kodu, jednak w mojej opinii jest ono dalekie od ideału. Pisząc w językach obiektowych powinniśmy korzystać z typów oraz obiektów, a nie z statycznych funkcji.

Wady tworzenia opisów za pomocą statycznych metod:
1. Statyczne metody nie mogą istnieć bez statycznych klas (przynajmniej nie w C#), w związku z czym oprócz nazwy metody musimy również wymyślić nazwę dla klasy. Ponieważ ciężko jest znaleźć odpowiednią nazwę dla takiej klasy otrzymujemy takie potworki jak `Describer` oraz `Helpers`.
2. Definiując statyczne metody kuszącym rozwiązaniem będzie umieszczenie kilku metod wewnątrz jednej klasy. Prowadzi to do sytuacji, w której nasze statyczne klasy puchną i stają się "workami na metody".

Moją propozycją na tworzenie opisów jest wprowadzenie osobnej klasy dla każdego z nich. Klasa ta powinna:
- posiadać jeden główny konstruktor;
- posiadać dowolną liczbę pomocniczych konstruktorów;
- być niezmienna (ang. *immutable*);
- posiadać nadpisaną metodę .ToString(), wywołanie której spowoduje stworzenie opisu i zwrócenie go jako `string`.

Przykład klasy opisującej osobę:
```csharp
public sealed class Person
{
    public string FirstName { get; set; }
    public string LastName { get; set; }
}

public sealed class DescriptionOfPerson
{
    private readonly string _firstName;
    private readonly string _lastName;
    
    public DescriptionOfPerson(Person person)
        : this(person.FirstName, person.LastName)
    {
    }
    
    // Main constructor
    public DescriptionOfPerson(
        string firstName,
        string lastName)
    {
        _firstName = firstName;
        _lastName = lastName;
    }

    public override string ToString()
    {
        return $"{_firstName} {_lastName}";
    }
}
```

Przykład klasy opisującej `enum`:
```csharp
public enum UserType
{
    Unregistered,
    Registered,
    Administrator,
}

public sealed class DescriptionOfUserType
{
    private readonly UserType _userType;
    
    public DescriptionOfUserType(UserType userType)
    {
        _userType = userType;
    }

    public override string ToString()
    {
        switch(_userType)
        {
            case UserType.Unregistered:
                // Instead of hardcoded strings you can use resources.
                return "Unregistered user";
            case UserType.Registered:
                return "Registered user";
            case UserType.Administrator:
                return "Administrator";
            default:
                // It's also ok to throw exception here.
                return _userType.ToString();
        }
    }
}
```


Co zyskujemy stosując takie rozwiązanie?
1. Utworzona przez nas klasa ma tylko jedno zadanie, co idealnie wpisuje się w zasadę *Single Responsibility Principle*. Trzymając się prostej zasady jeden opis = jedna klasa unikamy problemów klas będących "workami na metody".
1. Ponieważ nasza klasa ma tylko jedną odpowiedzialność, wygodne staje się pisanie testów jednostkowych.
1. Każdy z naszych opisów posiada swój własny typ, dzięki czemu możemy przekazać je dalej nie jako `string` lecz jako `DescriptionOfPerson` lub `DescriptionOfUserType`. W statycznie typowanych językach pozwala to na uniknięcie pomyłek związanych z błędną kolejnością przekazywania parametrów. Jeśli się pomylimy, to kompilator poinformuje nas natychmiast o błędzie.
1. Rozdzielamy w czasie definicję opisu (stworzenie obiektu) od jego obliczenia (wywołanie metody `.ToString()`). Niby drobiazg, jednak pozwala to na wykorzystanie zjawiska *lazy evaluation* oraz wykonywanie obliczeń dopiero gdy jest to konieczne.
1. Nazwa naszej klasy jest eleganckim rzeczownikiem, a nie jego imitacją wynikającą z dopisania końcówki *-er* do angielskiego czasownika. Jeśli nie masz nic przeciwko końcówką *-er* polecam [ten artykuł](https://www.yegor256.com/2015/03/09/objects-end-with-er.html)

Przedstawione tu rozwiązanie jest jedną z możliwości reprezentowania opisów za pomocą obiektów, która w zależności od projektu może sprawdzić się lepiej lub gorzej. Wierzę jednak, że tworzenie opisu za pomocą dedykowanej do tego celu klasy jest znacznie lepsze, niż korzystanie z statycznych metod.