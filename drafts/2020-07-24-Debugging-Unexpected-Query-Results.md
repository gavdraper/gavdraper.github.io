---
title: 10 Debugging Tips For SQL Server
date: 2020-07-24 00:00:00
subtitle: ''
description: "Breaking down the complicated"
featured_image: '/images/hero-images/duck.jpg'
---


## Missing Data
1.  Change inner joins to outer joins to see if the data pulls through, if it does then either the join condition is wrong or the data doesn't exist in both sides of the join.
2.  One at a time comment out where clause predicates, if the data pulls through then you now know the clause thats removing it.
3.  Does the data actually exist? Try to query the underlying tables one at a time for the smallest subset of the missing data that you can identify.

## Too Much Data
4.  Wrong Groups
5.  Mising Predicate
6.  Bad Join

## Other
7.  Strip Query Back
8.  If its grouping try running the query without the group and add a predicate for that returns just one group.





{% highlight sql %}
SELECT
    TOP 10 *
FROM
    [Users]
{% endhighlight %}
