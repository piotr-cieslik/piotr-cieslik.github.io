---
layout: post
title: "C# 7.0 - out variables"
date: 2018-10-05
categories: C#
---

W poprzednich wersjach C# przekazanie do funkcji argumentu z wykorzystaniem słowa kluczowego `out` wymagało wcześniejszej deklaracji zmiennej.

``` csharp
int result;
if (int.TryParse("1", out result))
{
    result++;
}
```

Out variables wprowadzone w C# 7.0 pozwalają nam na uproszczenie powyższego kodu. Zmienna lokalna `result` może zostać zdefiniowana w momencie wywołania funkcji.

``` csharp
if (int.TryParse("1", out int result))
{
    result++;
}
```

Dodatkowo definiując zmienną `result` możemy skorzystać z słowa kluczowego `var`.

``` csharp
if (int.TryParse("1", out var result))
{
    result++;
}
```

Out variables to proste udogodnienie, które zwiększa przejrzystość kodu.