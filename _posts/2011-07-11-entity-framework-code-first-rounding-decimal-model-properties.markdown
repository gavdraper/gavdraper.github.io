---
layout: post
title: Entity Framework Code First Rounding Decimal Model Properties
date: '2011-07-11 06:44:19'
---

If you use Entity Framework code first and user decimal properties with a scale greater than 2 then you will find that by default the table gets generated with a scale of 2 and values get rounded to 2 decimal places.

In order to tell the Entity Framework the scale and precision of these properties you need to override the OnModelCreating event in your data context. For example...

<pre class="brush: csharp; toolbar: false;">
protected override void OnModelCreating(DbModelBuilder modelBuilder)
{
    modelBuilder.Entity&lt;Activity&gt;().Property(a => a.Latitude).HasPrecision(18, 9);
    modelBuilder.Entity&lt;Activity&gt;().Property(a => a.Longitude).HasPrecision(18, 9);
}
</pre>

You will also either need to get the framework to regenerate your database, or manually edit your existing database so the decimal fields have the right precision and scale e.g Decimal(18,9)