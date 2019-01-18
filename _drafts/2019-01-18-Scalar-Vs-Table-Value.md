---
layout: post
title: Supercharge Your Scalar Functions By Switching To Table Value Functions
date: '2019-01-17 06:34:01'
tags: performance-tuning tsql
---
User defined functions in SQL server can cause all kinds of performance problems, there are however some tricks that are well worth knowing when you can't avoid using them...

Examples below are all on the [Stack Overflow Database](https://www.brentozar.com/archive/2015/10/how-to-download-the-stack-overflow-database-via-bittorrent/) which you can restore if you want to follow along.

Imagine for whatever reason there are places where you need to compare a DATETIME as an INT YYYYMMDD, we can write a scalar function to do just that...

{% highlight sql %}
CREATE FUNCTION dbo.DateToNumber (@Date DATETIME) RETURNS INT AS
BEGIN
RETURN CONVERT(VARCHAR(10), @Date, 112)
END
GO
{% endhighlight %}

Now let's imagine we want all badges obtained on a given date and just to make this example relevant let's also imagine for some obscure reason we need to switch all the dates to our numerical format to filter them...

{% highlight sql %}
SELECT * FROM Badges WHERE dbo.DateToNumber([Date]) = '20120804'
{% endhighlight %}

I completely accept it's quicker to just change our parameter to a DATETIME, but let's imagine in a real scenario we can't do that because our predicate is a join on a table that uses this numeric date format. The above query takes about 30 seconds on my machine to return 3500 results. At this point we're kind of out of luck with indexes as no index on date is going to help that index SCAN, if we look at the plan we can immediately see one possible issue...

![Scalar Plan]({{site.url}}/content/images/2019-TVP-UDF/udf-plan.PNG)

This plan did not go parallel anywhere even though it is seemingly pretty high cost, this is because a scalar function used anywhere in your query will force a serial plan. So what happens if we switch our Scalar function to a Table Value Function and use a CROSS APPLY in our query to filter by it?...

{% highlight sql %}
CREATE FUNCTION dbo.DateToNumberTvf(@Date DATETIME)
RETURNS TABLE
AS
RETURN (SELECT CONVERT(VARCHAR(10), @Date, 112) [Date]);
GO
{% endhighlight %}

Now let's adapt our SELECT query...

{% highlight sql %}
SELECT * FROM 
	Badges 
	CROSS APPLY dbo.DateToNumberTvf([Date]) d
WHERE d.[Date] = '20120804'
{% endhighlight %}

This gives that exact same results but now runs in less than a second. What's changed? 

![Scalar Plan]({{site.url}}/content/images/2019-TVP-UDF/tvf-plan.PNG)

Not only do we now have a parallel plan but that filter has been moved into the index scan lowering the amount of rows flowing through our plan, although yes we did still have to read them all as like I said earlier there is no index we can create that will allow a seek.

With the above in mind I find anytime I'm running a function across anything more than a couple of rows I'll lean towards using that Table Value Function first even if it does feel a little less intuitive to write.





