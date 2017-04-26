---
layout: post
title: Entity Framework Fluent API and Indexing
date: '2014-06-26 07:10:41'
---

With the release of Entity Framework 6.1 the Fluent API can now be used to create indexes. It's still pretty basic and will hopefully evolve to become more complete in future releases. 

For all the examples below I'm using the following model

```language-csharp
public class People
{
    public int Id { get; set; }
    public string Firstname { get; set; }
    public string Lastname { get; set; }
    public string PhoneNumber { get; set; }
    public string NationalInsuranceNo { get; set; }
}
```

## Samples ##

You can find the Visual Studio project for these samples at [https://github.com/gavdraper/EntityFrameworkFluentApiIndexing](https://github.com/gavdraper/EntityFrameworkFluentApiIndexing "https://github.com/gavdraper/EntityFrameworkFluentApiIndexing")

### Single column Indexes ###
```language-csharp
modelBuilder.Entity<People>()
    .Property(x => x.Firstname)
    .HasColumnAnnotation("Index", new IndexAnnotation(new IndexAttribute("ix_people_firstname")));
```

One thing to note here is that if your single column index matches the start of a multi column index the Entity Framework is smart enough not to create it as it's already covered by the multi column one.

### Multi Column Indexes ###
```language-csharp
modelBuilder.Entity<People>()
    .Property(x => x.Firstname)
    .HasColumnAnnotation("Index", new IndexAnnotation(new IndexAttribute("ix_people_fullname", 1)));
modelBuilder.Entity<People>()
    .Property(x => x.Lastname)
    .HasColumnAnnotation("Index", new IndexAnnotation(new IndexAttribute("ix_people_fullname", 2)));
```
 


### Unique Indexes ###
```language-csharp
modelBuilder.Entity<People>()
    .Property(x => x.NationalInsuranceNo)
    .HasColumnAnnotation("Index",
        new IndexAnnotation(new IndexAttribute("ix_people_nationalinsurance") {IsUnique = true}));
```
 


### Clustered Indexes
```language-csharp
modelBuilder.Entity<People>()
    .Property(x => x.Id)
    .HasColumnAnnotation("Index",
        new IndexAnnotation(new IndexAttribute("ix_people_nationalinsurance") { IsClustered = true}));
```
 


## What's Not Possible Using Fluent API
As of Entity Framework 6.1.1

-  Include Fields
-  Sort Order (ASC,DESC)
-  Filtered Indexes

Whilst this is a big step forwards it still feels like a work in progress. I would hope to end up with a friendlier and more complete syntax for doing this something like...

```language-csharp
modelBuilder.Entity<People>().Index("ix_people_firstname").SortByAsc();
```