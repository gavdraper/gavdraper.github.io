---
layout: post
title: SQL Server Debugging With RAISERROR Instead Of Print
date: '2018-05-21 20:07:45'
---
When testing/debugging TSQL it's common to use the print statement throughout to see what was happening where in much the same way that other languages use things like console.log.

One of the big issues with print is it buffers and you can't control when it gets written to the output stopping you seeing where a query is at whilst it's running.

For example...

{% highlight sql %}
PRINT 'First'
WAITFOR DELAY '00:00:05'
PRINT 'Second'
{% endhighlight %}

You'll see that neither print statement writes anything to the messages tab until the query has finished. This makes print statement useless for gauging what a query is currently doing.

RAISERRROR to the rescue, if you set a severity of info then it behaves much like print in that it won't change the way the query run and you can output messages. RAISERROR has a NOWAIT option you can use to cause it to return the output straight away and avoid the buffering problems that print has.

{% highlight sql %}
RAISERROR ('First', 0, 1) WITH NOWAIT
WAITFOR DELAY '00:00:05'
RAISERROR ('Second', 0, 1) WITH NOWAIT
{% endhighlight %}

In the above example we're using a severity of zero to make sure it acts as an info message and not an actual error, this will cause it to work just like print even when wrapped in a try block. To prove this...

{% highlight sql %}
BEGIN TRY
   RAISERROR ('Here', 0, 1) WITH NOWAIT
END TRY
BEGIN CATCH
   PRINT 'Won't Ever Get Here'
END CATCH
{% endhighlight %}