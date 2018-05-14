---
layout: post
title: SQL Server Building On Calculations With APPLY
date: '2018-05-14 22:12:24'
---
I find myself using Apply more and more in my queries, aside from the usual reasons I've been finding I can write much clearer queries that reuse calculations and build on them rather than constantly repeating code.

Let's look at an example imagine we have the following table...

{% highlight sql %}
CREATE TABLE sales
(
    Id INT IDENTITY PRIMARY KEY,
    Price DECIMAL(8,2),
    Quantity INT,
    DiscountPercent INT,
    DiscountFixedAmount DECIMAL(8,2)
)
{% endhighlight %}

We then want to return the following...

- Price
- Quantity
- ValueBeforeDiscount : Price+Quantity
- DiscountFixedAmount
- DiscountPercentAmount : ((ValueBeforeDiscount)/100)*DiscountPercent
- ValueWithDiscount : ValueBeforeDiscount - DiscountPercentAmount - DiscountFixedAmount

You might attempt to do something like this...

{% highlight sql %}
SELECT
    Price,
    Quantity,
    Price + Quantity AS ValueBeforeDiscount,
    DiscountFixedAmount,
    ((ValueBeforeDiscount)/100)*DiscountPercent AS DiscountPercentAmount,
    ValueBeforeDiscount - DiscountPercentAmount - DiscountFixedAmount AS ValueWithDiscount
FROM
    Sales
{% endhighlight %}

This however will not run. The reason for this is down to the order SQL Server parses and runs the query, you cannot reference select fields from another select. This leads us to rewrite the above solution to look more like this...

{% highlight sql %}
SELECT
    Price,
    Quantity,
    Price + Quantity AS ValueBeforeDiscount,
    DiscountFixedAmount,
    ((Price + Quantity)/100)*DiscountPercent AS DiscountPercentAmount,
    (Price + Quantity)
      - (((Price + Quantity)/100)*DiscountPercent)
      - DiscountFixedAmount
   AS ValueWithDiscount
FROM
    Sales
{% endhighlight %}

As you can see this leads to a lot of repeated code as as our calculations get longer they become more and more unreadable. Enter APPLY, because of the order of operations SQL performs APPLY operations can reference other APPLY operations so we can rewrite our query to look like this...

{% highlight sql %}
SELECT
    Price,
    Quantity,
    BeforeDiscount.Value AS ValueBeforeDiscount,
    DiscountFixedAmount,
    DiscountPercent.Value DiscountPercentValue,
    ValueWithDiscount.Value AS ValueWithDiscount
FROM
    Sales
    CROSS APPLY (SELECT Price+Quantity) AS BeforeDiscount(Value)
    CROSS APPLY (SELECT (BeforeDiscount.Value/100)*Sales.DiscountPercent) AS DiscountPercent(Value)
    CROSS APPLY (SELECT (BeforeDiscount.Value - Sales.DiscountFixedAmount - DiscountPercent.Value)) 
      AS ValueWithDiscount(Value)
{% endhighlight %}

If you're worried about performance insert a few sample rows and run look at the execution plan for the query with and the query without the APPLY, you'll see both queries are using the exact same plan.

So in reality in this case it's probably more code this way with lengthier syntax but I think this is far more readable,  each calculation is defined only once and builds on the previous steps making code changes easier to make. You possibly wouldnt use this approach for examples as simple as the one above but it can be very useful for more complex queries.