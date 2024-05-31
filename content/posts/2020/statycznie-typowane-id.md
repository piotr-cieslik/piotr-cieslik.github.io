---
title: "Statycznie typowane ID"
date: 2020-04-13
tags: ["c#"]
draft: true
---

Czy zdarzyło Ci się kiedyś przekazać do funkcji argumenty w złej kolejności? Jeśli pomyłka ta sposowuje podmianę imienia z nazwiskiem, to najprawdopodobniej nic złego się nie stanie. Istnieją jednak sytuacje, gdzie taka pomyłka może mieć katastrofalne skutki. Potrafię wybrazić sobie jak miło jest tłumaczyć się przed klientem, dlaczego widzi on dane nie swojej firmy, a co gorsza, czemu inna firma widzi jego dane.

Informacje zawarte w tym artykule bazują głównie na doświadczeniach wyniesionych z wykorzystania bazy danych MS SQL oraz biblioteki Entity Framework. Jestem jednak przekonany, że przedstawione tu podejście jest na tyle uniwersalne, że bez problemu sprawdzi się również w połączeniu z innymi technologiami.

## Problem
Rozważmy następujacą funkcję:

``` csharp
public TaskDetails GetTaskDetails(
    int companyId,
    int personId,
    int taskId)
{
    // Implementation here.
    throw new NotImplementedException();
}
```

Nazwa oraz zwracany typ nie mają tutaj znaczenia, ważne jest to, iż przyjmuje ona 3 argumenty typu `int`, na podstawie których zwraca jakieś dane. Jeśli w naszym projekcie istnieje chociaż jedna klasa typu serwis, fasada czy repozytorium, znajdziemy tam funkcję o podobnej budowie.

Popatrzmy teraz na jej przykładowe wykorzystanie:
``` csharp
var taskDetails =
    GetTaskDetails(
        company.Id,
        person.Id,
        task.Id);
```

Do tej pory jest wszystko ok. Co się jednak stanie, gdy z jakiegoś powodu pomylimy kolejność przekazywanych argumentów? Przykładowo zamieńmy miejscami `personId` oraz `companyId`. 

``` csharp
var taskDetails =
    GetTaskDetails(
        person.Id,
        company.Id,
        task.Id);
```

Z punktu widzenia logiki biznesowej pomieszaliśmy przysłowiowe jabłka z gruszkami. Nie ma sensu identyfikowania firmy po numerze osoby, podobnie jak bez sensu jest identyfikowanie osoby po numerze firmy (pomińmy proszę przypadek jednoosobowych działalności gospodarczych :)).  Jednak z punktu widzenia naszego kompilatora nie zmieniło się nic. Nasz program nadal się kompiluje oraz możemy go uruchomić. Jeśli mamy szczęście okaże się, że w naszej bazie danych nie istnieją wpisy odpowiadające wprowadzonej kombinacji `personId` oraz `companyId` i w efekcie zwrócony zostanie brak danych lub rzucony zostanie wyjątek. Gorzej jeśli w naszej bazie istnieją pasujące wpisy. Skutkiem tego nasi klienci mogą zobaczyć informację na temat osób oraz zadań przypisanych do innej firmy. Taki drobny błąd może nas sporo kosztować, łącznie z pozwem ze strony naszych klientów.

Istnieje oczywiście szansa, że błąd ten zostanie wychwycony na etapie testowania aplikacji i wszystko skończy się dobrze. Chciałbym jednak przedstawić rozwiązanie, które pozwoli na wykrycie tego typu problemu wcześniej, już na etapie kompilacji programu.

## Rozwiązanie
Zamiast reprezentować wartości pól *Id* poszczególnych elementów za pomocą typu `int` zdefiniujmy w tym celu własne typy danych. Nazwijmy je:
- `CompanyId`,
- `PersonId`,
- `TaskId`.

Jednak zanim to zrobimy zastanówmy się przez chwilę jakie właściwości powinny mieć powyższe typy aby mogły być w wygodny sposób wykorzystywane jako *Id* elementów. W mojej ocenie powinny być one:
- niezmienne,
- pozwalać na porównywanie za pomocą metody `Equals` oraz operatorów `==` i `!=`.

Mając na uwadze powyższe właściwości, przykładowa implementacja typu `CompanyId` może wyglądać tak:
``` csharp
public sealed class CompanyId : IEquatable<CompanyId>
{
    public CompanyId(int value) => Value = value;

    public int Value { get; }

    public override bool Equals(object obj) => Equals(obj as CompanyId);

    public bool Equals(CompanyId other) => other != null && Value == other.Value;

    public override int GetHashCode() => HashCode.Combine(Value);

    public static bool operator ==(CompanyId left, CompanyId right) => EqualityComparer<CompanyId>.Default.Equals(left, right);

    public static bool operator !=(CompanyId left, CompanyId right) => !(left == right);
}
```

Typy `PersonId` oraz `ContractorId` mogą być zdefiniowane w analogiczny sposób. Pełna implementacja znajduje się na w podlinkowanym repozytorium na GitHub. Warto w tym miejscu zauważyć, że nowe wersję Visual Studio pozwalają na automatyczną implementację interfejsu `IEquatable<T>`.

Mając zdefiniowane własne typy, możemy wykorzystać je jako parametry metody `GetTaskDetails`:

``` csharp
public TaskDetails GetTaskDetails(
    CompanyId companyId,
    PersonId personId,
    TaskId taskId)
{
    // Implementation here.
    throw new NotImplementedException();
}
```

W tym momencie zamiana miejscami `companyId` oraz `personId` spowoduje błąd kompilacji.

``` csharp
Error    CS1503    Argument 1: cannot convert from 'PersonId' to 'CompanyId'
Error    CS1503    Argument 2: cannot convert from 'CompanyId' to 'PersonId'    
```

## Efekt skali
Początkowo zdefiniowanie własnych typów reprezentujących ID elementów może wydawać się sztuczne i powodujące sporo problemów. Tym bardziej, że będą istniały miejsca, w których konieczna będzie ręczna konwersja "opakowanego" typu (`int`/`string`/`Guid`/...) na zdefiniowany przez nas typ
``` csharp
var stronglyTypedPersonId = new PersonId(personId);
```

oraz konwersja w drugą stronę
``` csharp
var personId = stronglyTypedPersonId.Value();
```

Jeśli jednak będziemy konsekwentnie korzystać z naszych nowych typów wyprą one, praktycznie całkowicie, wykorzystanie innych typów.

## Klucze typu String, Guid, ...
Nic nie stoi na przeszkodzie aby przedstawione rozwiązanie zastosować do reprezentacji *Id* innego typu niż `int`. Przykładowo gdy kluczem identyfikującym osobę w naszym systemie jest typ `string`, jak ma to miejsce w przypadku Identity Framework, nasz typ `PersonId` może wyglądać tak:

``` csharp
public sealed class PersonId : IEquatable<PersonId>
{
    public PersonId(string value) => Value = value;

    public string Value { get; }

    public override bool Equals(object obj) => Equals(obj as PersonId);

    public bool Equals(PersonId other) => other != null && Value == other.Value;

    public override int GetHashCode() => HashCode.Combine(Value);

    public static bool operator ==(PersonId left, PersonId right) => EqualityComparer<PersonId>.Default.Equals(left, right);

    public static bool operator !=(PersonId left, PersonId right) => !(left == right);
}
```

## Złożone klucze
Zdefiniowanie własnego typu reprezentującego *Id* sprawdza się również, w przypadku gdy nasz klucz składa się z więcej niż jednej wartość.

Przykładowo definiując, jako identyfikator pracownika, parę wartości (osoba, firma), możemy zdefiniować typ `EmployeeId` w następujący sposób:

``` csharp
public sealed class EmployeeId : IEquatable<EmployeeId>
{
    public EmployeeId(
        CompanyId companyId,
        PersonId personId)
    {
        CompanyId = companyId;
        PersonId = personId;
    }

    public CompanyId CompanyId { get; }

    public PersonId PersonId { get; }

    public override bool Equals(object obj) => Equals(obj as EmployeeId);

    public bool Equals(EmployeeId other) =>
        other != null &&
            Equals(CompanyId, other.CompanyId) &&
            Equals(PersonId, other.PersonId);

    public override int GetHashCode() => HashCode.Combine(CompanyId, PersonId);

    public static bool operator ==(EmployeeId left, EmployeeId right) => EqualityComparer<EmployeeId>.Default.Equals(left, right);

    public static bool operator !=(EmployeeId left, EmployeeId right) => !(left == right);
}
```

## Klasa czy struktura?
Niestety nie znam odpowiedzi na to pytanie. Osobiście definiuję  typy reprezentujące *Id* jako klasy. Wynika to jedynie z faktu, że domyślną wartością dla klasy (typ referencyjny) jest `null`. Jeśli w jakimś miejscu otrzymam domyślną wartość dla mojego typu, próba wyciągnięcia przechowywanej przez niego wartości skończy się rzuceniem wyjątku.

``` csharp
var personId = default(PersonId);
var value = personId.Value(); // Null reference exception
```

Gdyby w powyższym przykładzie typ `PersonId` był strukturą otrzymałbym wartość `0`, co potencjalnie może prowadzić do trudnych w odnalezieniu błędów. 

Wybór pomiędzy klasą, a strukturą zależy w dużej mierze od domeny aplikacji oraz preferencji programistów. Jedyne co mogę polecić w to miejscu to być konsekwentnym i trzymać się wybranej opcji w całym programie.

## Podsumowanie
Zaletą przedstawionego rozwiązania jest jego prostota oraz niewiele dodatkowego kodu, który musi zostać napisany. Wystarczy, że zdefiniujemy nasz typ raz (przykładowo podczas dodawania nowej encji do systemu) a następnie możemy korzystać z niego tak samo jak w przypadku typów `int`, `string`, `Guid`, ...

Przedstawione tu rozwiązanie nie jest rozwiązaniem jedynie teoretycznym. Zostało ono z sukcesem wprowadzone do dwóch średniej wielkości systemów.

Zachęcam Cię serdecznie do jego przetestowania.

---
- [https://github.com/piotr-cieslik/Blog.StaticallyTypedIds](https://github.com/piotr-cieslik/Blog.StaticallyTypedIds)
