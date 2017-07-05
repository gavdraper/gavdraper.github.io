---
layout: post
title: SQL Server 2017+ How To Replace Multiple Characters In A Single Statement With TRANSLATE
date: '2017-07-05 07:35:36'
---
The REPLACE function in SQL Server has until now been quite limited. SQL Server 2017 has introduced a new TRANSLATE function that addresses many of these shortcomings, namely the need to daisy chain replace calls to replace multiple characters. 

These two functions are however quite different in how they work and what they output.

- REPLACE will replace all occurrences of a set of characters with another set of characters.
- TRANSLATE will one by one replace each character in the match expression with the character in that same position on the replace side. This means both expressions need to be the same length

Let's look at some examples...

{% highlight sql %}
SELECT REPLACE('testinf','testinf','testing')
SELECT TRANSLATE('testinf','testinf','testing')
{% endhighlight %}

In the above example both functions will return 'testing', so what's so good about translate? Let's take something a bit more complex so we can see when translate makes more sense. Imagine we want to swap all periods and commas with a space...

Using REPLACE that would look like this...

{% highlight sql %}
SELECT REPLACE(REPLACE('testing, 123.','.',' '),',',' ')
{% endhighlight %}

Using replace for this gets pretty ugly as we have to chain replace methods. I've seen procedures that chain several or more replaces to clean data. Let's look at how this is done with replace...

{% highlight sql %}
SELECT TRANSLATE('testing, 123.','.,','  ')
{% endhighlight %}

Much neater, each character or the match expression is matched with a space on the replace side and swapped out.