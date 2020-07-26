---
title: Debugging Unexpected SQL Results
date: 2020-07-24 00:00:00
subtitle: ''
description: "Breaking down the complicated"
featured_image: '/images/hero-images/duck.jpg'
---
{% highlight sql %}
SELECT
    TOP 10 *
FROM
    [Users]
{% endhighlight %}

## Missing Data
    Joins
    Where Clauses
    Check The Data

## Too Much Data
    Wrong Groups
    Mising Predicate
    Bad Join

## Other
    Strip Query Back
