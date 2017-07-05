---
layout: post
title: IIF and CHOOSE Instead of CASE
date: '2017-07-06 08:54:01'
---
SQL Server 2012 introduced IIF and CHOOSE functions and I completely missed they even existed until recently. They make some quite messy CASE statements go away. Lets have a look...

Let's imagine we have a need to display something different in our select depending on the value of a field in a table. In the case of this example I'll use a variable rather than a table. If my variable is equal to 'Test' then I want to return 'Woop' if it's not I'll return 'Oh No'. I've always used CASE statements for this previously and that looks a bit like this...

{% highlight sql %}
DECLARE @Testing NVARCHAR(4) = 'Test'
SELECT CASE WHEN @Testing = 'Test' THEN 'Woop' ELSE 'Oh No' END
{% endhighlight %}

This is fine but lets see if we can reduce some of the clutter with IIF. IIF takes an expression and true/false values, If the expression evaluates to true then the true value is returned else it's the false value...

{% highlight sql %}
DECLARE @Testing NVARCHAR(4) = 'Test'
SELECT IIF(@Testing = 'Test','Woop','Oh No')
{% endhighlight %}

Let's now turn our heads to the true function. Time for another example.... Let's imagine we have a forum and we store the amount of posts each user has made, we then convert that into a level 1-5. We then want to return a status for each user depending on their level. Using a CASE statement we could do this...

{% highlight sql %}
DECLARE @UserLevel INT = 4
SELECT 
	CASE @UserLevel 
	WHEN  1 THEN 'Newbie'
	WHEN 2 THEN 'Regular'
	WHEN 3 THEN 'Chatterbox'
	WHEN 4 THEN 'Solcial Butterfly'
	WHEN 5 THEN 'Admin'
END
{% endhighlight %}

The CHOOSE statement can massively reduce the code needed to do this, It takes an index and a list of items, it then returns the item at the specified index or NULL if there is no item at that index...

{% highlight sql %}
SELECT CHOOSE(4,'Newbie','Regular','Chatterbox','Solcial Butterfly','Admin')
{% endhighlight %}

