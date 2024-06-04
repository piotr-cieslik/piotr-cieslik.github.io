---
title: "Different types of DTOs"
date: "2024-06-04"
tags: ["c#"]
---

Working on various REST API and WCF projects in large organizations, I've repeatedly encountered the same destructive patterns. One of them is the use of a single DTO model for both read and write operations. Whether this is due to bad luck or a once-popular "best practice" is unclear. What matters is explaining why I find this approach harmful and presenting a better alternative.

# Problem
Consider a simple controller with endpoints for invoice manipulation, as shown below. You've probably seen many endpoints like this, where the same type of DTO is used for both read and write operations. At first glance, it looks nice and clear, but when you start working with one of these endpoints, you'll likely find it complicated. Let's dive deeper into it.

``` csharp
public sealed class InvoicesController : ControllerBase
{
  public IEnumerable<InvoiceDto> GetMany()
  {
    /* code here */
  }

  public InvoiceDto GetOne(int id)
  {
    /* code here */
  }

  public InvoiceDto Create(InvoiceDto dto)
  {
    /* code here */
  }

  public InvoiceDto Update(InvoiceDto dto)
  {
    /* code here */
  }

  public void Delete(int id)
  {
    /* code here */
  }
}
```

It's our DTO.

``` csharp
public sealed class InvoiceDto
{
  public int Id { get; set; }

  public DateTime CreatedAt { get; set; }

  public string CreatedBy { get; set; }

  public string Number { get; set; }

  public int CompanyId { get; set; }

  public CompanyDto Company { get; set; }

  /* ... */
}
```
The DTO probably looks exactly the same as the related database model. This might sound like a good design because it allows you to use AutoMapper to map between the DTO and the database model. Unfortunately, in most cases, this is a poor and problematic design. I'll explain why in a moment.

*Quick note: Depending on the implementation, we might find an invoice model with only a `CompanyId` property, only a `Company` property, or both. In this example, I've chosen the last option, but it doesn't affect the problem I'm describing.*

Why is it poor design? Let's walk through an example: creating a new invoice using this API. Imagine that we've got the company ID by calling a different endpoint, and now we're ready to create the invoice.

``` csharp
var newInvoice = new InvoiceDto
{
  Id = default, // What put here?
  CreatedAt = default, // What put here?
  CreatedBy = default, // What put here?
  Number = "2024/06/01", // OK, it's fine.
  CompanyId = 7, // OK, it's fine.
  Company = default, // What put here, do I need to fetch whole DTO?
};
```

You can probably guess that properties like Id, CreatedAt, and CreatedBy can be safely skipped, but what about Company? I would assume it can be skipped as well, but recently I faced a similar API that didn't allow me to skip it. Now, you have to sit and think about which properties to fill. Actually, not only you, but every person who ever uses this API has to do the same. And it's all for the convenience of API development.

Using `InvoiceDTO` as a read model is also problematic. Imagine you want to add one more property to it, just to present it on a details page. You'll have to change a model that is commonly used and probably have to test a lot of places now, just to be sure it will not break anything.

# Solution

The solution to this problem is to stop sharing the same model for read and write operations and instead expose properties specific to the given context. I understand that this approach entails writing more code, but it's a one-time effort that ultimately simplifies the lives of all API consumers. Additionally, this code is easier to maintain because it's less coupled. While I mentioned splitting into read and write models, I often go further and divide my models into three or even four different types of DTOs. Below, I'll present each of them along with a brief explanation of their purpose and when they should be used.


Please keep in mind that the solution presented below is not a silver bullet that solves all your problems, but it can be treated as a starting point when designing a new API. There are still some scenarios where a DTO is so simple that using the same model is perfectly acceptable. I use this solution for all my current projects (SPA written in React.js + .NET Core Web API), and I find it useful in almost every situation.

It's also worth mentioning that a very similar approach is used by GitHub. You can find more information about it in their REST API documentation: https://docs.github.com/en/rest/using-the-rest-api/getting-started-with-the-rest-api?apiVersion=2022-11-28#detailed-versus-summary-representations

## ReadDTO

This is my basic read model. I primarily use it to return detailed information about a record with all necessary relationships. It's mainly utilized for the details page. If the model is simple and doesn't have many dependencies, I use it as the sole read model.

``` csharp
public sealed class InvoiceDto
{
  public int Id { get; set; }

  public DateTime CreatedAt { get; set; }

  public string CreatedBy { get; set; }

  public string Number { get; set; }

  // You can also use CompanyReadDto here.
  public CompanyReferenceDto Company { get; set; } 

  /* ... */
}
```

## WriteDTO

The second basic model is the WriteDTO. Most of the models exposed by the API have both a ReadDTO and a WriteDTO model.Here, we expose only editable properties to avoid confusing API consumers about what is editable and what is not.

In our example, I would expose only two properties: `Number` and `CompanyId`. Please note that I also do not include the Id property in these models. In my opinion, it's better to pass the ID in the URL rather than in the model.

``` csharp
public sealed class InvoiceDto
{
  public string Number { get; set; }

  public int CompanyId { get; set;}
}
```

Now it's much easier to see which properties have to be filled to create an invoice.

``` csharp
var newInvoice = new InvoiceDto
{
  Number = "2024/06/01",
  CompanyId = 7,
};

```
## ReferenceDTO

This type of DTO is used to represent the relationship between items. Often, in the returned DTO, we want to return more than just the ID of the related model, but less than the entire model.

I've found the ReferenceDTO to be very useful when combined with React.js. I create a link component that takes a ReferenceDTO as a parameter and presents a nicely formatted link to the related entity. These links maintain consistency across the entire project.

``` csharp
public sealed class CompanyReferenceDto
{
  public int Id { get; }

  public string Name { get; }
}
```

## SummaryDTO

Last on our list is SummaryDTO. This type of DTO model can be used instead of ReadDTO when returning list models is resource-intensive and we need to return many of them.

I don't use them very often, but invoices are great examples of where I would use it. The ReadDTO model may contain all positions of an invoice, as we probably want to present them on the details page. On the other hand, SummaryDTO can present only the total net, VAT, and gross value to display them in a table.

``` csharp
public sealed class InvoiceSummaryDTO
{
  public int Id { get; }

  public string Number { get; }

  public CompanyReferenceDto Company { get; } 
}
```

# Conclusion

It might be an uncommon approach, but the design of the API should prioritize ease of consumption over ease of development. If you can achieve both, that's great, but if not, you may have to sacrifice the development aspect.
