---
layout: post
title: SQL Server Dynamic Pivot
date: '2017-05-07 09:01:32'
---
Often when trying to pivot data you wont know which all the possible values are that you need to pivot on, in this case you can use dynamic SQL.

Lets say we have a table with employee sale counts

| Employee | Sales |
| --- | --- |
| Gavin | 10 |
| Troy | 4 | 
| Joe | 3 |

If we want to pivot that to have a column per employee but we don't know all the employee upfront we'll need to use dynamic SQL. This can be achived like this...

{% highlight sql %}
{% endhighlight %}