---
layout: post
title: Entity Framework 6 Entity/Property To Table/Column Names
date: '2015-01-26 13:43:24'
---

I was recently working on some code where I needed to be able to go from an EntitySet name to a table name and from an entity property name to a column name. It's not something I've ever needed to do before so had to do some playing around with the Entity Framework meta data API's, I very quickly learned these are a pain to work with. 

After a good few hours and a lot of code I pretty much had it working then I stumbled on to [EntityFramework.MappingAPI](https://efmappingapi.codeplex.com "EntityFramework.MappingAPI"). Within a minute of installing it I'd replaced my 100 odd lines with about 5 lines of very readable code. This project is amazing if you ever want to interrogate the Entity Framework meta data from code. 

As an example lets say you have the following context and model defined

```language-csharp
public class Book
{
    public int Id { get; set; }
    public string Title { get; set; }
}

public class MyContext : DbContext
{
    public DbSet<Book> Books { get; set; }

    protected override void OnModelCreating(DbModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Book>()
        	.ToTable("AwesomeBooks");
        modelBuilder.Entity<Book>()
        	.Property(x => x.Title)
            .HasColumnName("AwesomeTitle");
        base.OnModelCreating(modelBuilder);
    }
}
```

So I have an EntitySet called books that maps to a table called AwesomeBooks and a property on there called Title that maps to a column called AwesomeTitle. Let's say in code I want to find out the Tablename and Column name for these objects, all I'd need to do is install EntityFramework.MappingAPI from nuget and use this code...

```language-csharp
using (var ctx = new MyContext())
{
    var map = ctx.Db(x => x.Books);
	var tableName = map.TableName;
	var propertyName = map.Properties
    	.Where(x => x.PropertyName == "Title")
        .Single()
        .ColumnName;
}
```

Almost feels too easy. There is a wealth of other information exposed by EntityFramework.MappingAPI, take a look at it's documentation over at [https://efmappingapi.codeplex.com/documentation](https://efmappingapi.codeplex.com/documentation "EntityFramework.MappiAPI CodePlex")