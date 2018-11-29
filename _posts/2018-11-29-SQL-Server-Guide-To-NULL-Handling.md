---
layout: post
title: SQL Server Guide To NULL Handling
date: '2018-11-29 09:09:33'
---
Every language handles null equality differently and understanding this is crucial as a misunderstanding here can lead to some quite nasty unexpected results.

In some languages NULL == NULL will be true and in others it will be false, SQL has a couple of caveats around this to be aware of. Before we look at the below examples this post assumes that you are running the default option on your SQL Server instance of ANSI_NULLS = true, if you change this setting then SQL Server will consider nulls as equal. Lets look at some examples...


{% highlight sql %}
/* Returns 'NULL != NULL'*/
IF NULL = NULL
   PRINT 'NULL == NULL'
ELSE
   PRINT 'NULL != NULL'
{% endhighlight %}

If you run this you can see that NULL does not equal NULL. In SQL we use IS NULL to check for a null...

{% highlight sql %}
/* Returns 'NULL Is NULL */
IF NULL IS NULL
   PRINT 'NULL Is NULL'
ELSE
   PRINT 'NULL Is Not NULL'
{% endhighlight %}

What if we want to see if two possibly null fields are equal and return true if they are both NULL or their values match? We can make use of the ISNULL operator here which specifies a value to return if the input parameter is NULL for example...

{% highlight sql %}
/* Returns 'Hello World' */
DECLARE @MyVariable NVARCHAR(100) = NULL
SELECT ISNULL(@MyVariable,'Hello World')
{% endhighlight %}

The above will return hello world because the input was null. You can use this to check if possibly null fields are equal by passing in a default value to use for null types...

{% highlight sql %}
DECLARE @MyVariable NVARCHAR(100) = NULL
DECLARE @MyVariable2 NVARCHAR(100) = NULL
DECLARE @MyVariable3 NVARCHAR(100) = 'Hello'

/* Will print match as both variables get converted to the string 'NULLSTRING' 
and then they both match */
IF ISNULL(@MyVariable,'NULLSTRING') = ISNULL(@MyVariable2,'NULLSTRING')
   PRINT 'Match'

/* Will print 'No Match' as NULLSTRING <> Hello */
IF ISNULL(@MyVariable,'NULLSTRING') <> ISNULL(@MyVariable3,'NULLSTRING')
   PRINT 'No Match'
{% endhighlight %}

What if there is no default value you can use for the ISNULL check? Then we have to get a bit more creative in our predicates...

{% highlight sql %}
DECLARE @MyVariable NVARCHAR(100) = NULL
DECLARE @MyVariable2 NVARCHAR(100) = NULL
/* Will print Match as the IS NULL check is true */
IF
   (@MyVariable IS NULL AND @MyVariable2 IS NULL) OR
   (@MyVariable = @MyVariable2)
      PRINT 'Match'
{% endhighlight %}

Things also get a little weird when you start adding or concatenating non nulls with nulls....

For example this string concatenation will return null

{% highlight sql %}
DECLARE @MyVariable NVARCHAR(100) = 'Hello'
DECLARE @MyVariable2 NVARCHAR(100) = NULL
DECLARE @MyVariable3 NVARCHAR(100) = 'World'
/* Returns NULL */
SELECT @MyVariable + @MyVariable2 + @MyVariable3
{% endhighlight %}

How about this integer calculation that will also return null...

{% highlight sql %}
DECLARE @MyVariable INT = 1
DECLARE @MyVariable2 INT = NULL
DECLARE @MyVariable3 INT = 2
/* Returns NULL */
SELECT @MyVariable + @MyVariable2 + @MyVariable3
{% endhighlight %}

Both of these are fixed by using ISNULL and specifying default values...

{% highlight sql %}
DECLARE @MyVariable INT = 1
DECLARE @MyVariable2 INT = NULL
DECLARE @MyVariable3 INT = 2
/* Returns 3 */
SELECT ISNULL(@MyVariable,0) + ISNULL(@MyVariable2,0) + ISNULL(@MyVariable3,0)
{% endhighlight %}

{% highlight sql %}
DECLARE @MyVariable NVARCHAR(100) = 'Hello'
DECLARE @MyVariable2 NVARCHAR(100) = NULL
DECLARE @MyVariable3 NVARCHAR(100) = 'World'
/* Returns HelloWorld */
SELECT ISNULL(@MyVariable,'') + ISNULL(@MyVariable2,'') + ISNULL(@MyVariable3,'')
{% endhighlight %}

So NULL is always != NULL then right? Well, ummmm, kind of.....

The UNION operator in SQL will return all results from both sides except where they match, in which case it discards the duplicate result from one side, for example the following query will return 1,2 and one of the 1 results will be discarded...

{% highlight sql %}
SELECT 1
UNION SELECT 1
UNION SELECT 2
{% endhighlight %}

So let's try throwing some nulls into the mix...

{% highlight sql %}
SELECT 1
UNION SELECT NULL
UNION SELECT NULL
{% endhighlight %}

Based on what we know (NULL != NULL), you'd probably expect this to return 3 records (1 and 2 NULLs), however a 1 and a single NULL is all that returns, the UNION operator will filter out matching NULLs and treat them as equal.

The except and intersect operators also work in this way...

{% highlight sql %}
/* Returns no results */
SELECT NULL
EXCEPT SELECT NULL
{% endhighlight %}

{% highlight sql %}
/* Returns 1 NULL result */
SELECT NULL	
INTERSECT SELECT NULL
{% endhighlight %}

You're probably starting to see how dangerous any value that can be null is and how easy it is to get unexpected results. One of the largest causes of bugs in software is unexpected null values that have not been properly handled. Where ever you can it will reduce risk later if you just don't allow NULLS to get into your tables or come out of your queries (This will not always be possible). 
