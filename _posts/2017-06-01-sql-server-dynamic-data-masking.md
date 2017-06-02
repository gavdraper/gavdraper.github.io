---
layout: post
title: SQL Server Dynamic Data Masking In Action
date: '2017-06-01 21:05:38'
---
One of the new features SQL Server 2016 introduced is Dynamic Data Masking. This feature allows sensitive data to be masked at the database layer with no changes to the application.

## Demo Time ##
Run the following script in a new database to setup a sample table.

{% highlight sql %}
CREATE TABLE dbo.Employee
(
    Id INT IDENTITY PRIMARY KEY,
    FirstName NVARCHAR(100),
    LastName NVARCHAR(100),
    WorkPhone NVARCHAR(50),
    WorkEmail NVARCHAR(50),
    Salary MONEY,
    PersonalPhone NVARCHAR(50),
    PersonalEmail NVARCHAR(100)
)

INSERT INTO dbo.Employee
    (FirstName, LastName, WorkPhone, WorkEmail, 
    Salary, PersonalPhone, PersonalEmail)
VALUES
    ('Joe', 'Bloggs', '+44 0454 4543', 'joe@work.com',
    44000, '+44 284 4953', 'joe@home.com')
{% endhighlight %}

If we then run a select statement against the new table we get this...

{% highlight sql %}
SELECT * FROM [dbo].Employee
{% endhighlight %}

![No Mask Results]({{site.url}}/content/images/2017-data-masking/no-mask.PNG)

Let's imagine we only want our standard users to be able to see the Work Phone/Email but not the personal ones or the salary. Let's setup a new login for StandardUser with select permission on our table.

{% highlight sql %}
CREATE USER StandardUser WITHOUT LOGIN;  
GRANT SELECT ON dbo.Employee TO StandardUser;   
{% endhighlight %}

Now let's create another user for Accounting that should have access to see all the data unmasked. 

{% highlight sql %}
CREATE USER Accounting WITHOUT LOGIN; 
GRANT SELECT ON dbo.Employee TO Accounting; 
GRANT UNMASK TO Accounting
{% endhighlight %}

Notice how we gave this user the UNMASK permission, allowing them to see unmasked data on tables with masked fields.

Let's start by masking the PersonalEmail field

{% highlight sql %}
ALTER TABLE dbo.Employee ALTER COLUMN PersonalEmail ADD MASKED WITH(FUNCTION='email()')
{% endhighlight %}

If we then execute our previous select statement running as our StandardUser account we'll see that we now get a masked version of an email address...

{% highlight sql %}
EXECUTE AS USER = 'Developer';  
SELECT * FROM dbo.Employee
REVERT;   
{% endhighlight %}

![Masked Results]({{site.url}}/content/images/2017-data-masking/masked-email.PNG)

If we run that same query as the accounting user we can see they do in fact get the unmasked data...

{% highlight sql %}
EXECUTE AS USER = 'Accounting';  
SELECT * FROM dbo.Employee
REVERT;   
{% endhighlight %}

![Unmasked Results]({{site.url}}/content/images/2017-data-masking/no-mask.PNG)

## Masking Functions ##
In the example above we masked the data using the email function, there are 4 different masking functions available depending on the type and shape of mask you want to use...

1. **Default** will apply a mask depending on the type of the data. String types will return X's, numeric types will return 0's and Dates will return 1900-01-01.
1. **Email** when used with a valid email address if will mask it by keeping the first letter, the @  and the domain suffix with X's padded between.
1. **Random** wen used on a numeric type will return a random number inside the range you specify.
1. **Custom** String will expose the first x letters followed by a custom string, followed by the last x letters. 

Let's finish masking our table using the methods above.

### Salary ###
For salary we want a random number between 10,000 and 100,000 which can be done with the random function.

{% highlight sql %}
ALTER TABLE dbo.Employee ALTER COLUMN Salary ADD MASKED WITH(FUNCTION = 'random(10000,100000)')
{% endhighlight %}

### Personal Phone ###
For this we can use the Custom String function to return the 3 letters of the country code then 6 X's then the last 3 characters.

{% highlight sql %}
ALTER TABLE dbo.Employee ALTER COLUMN PersonalPhone ADD MASKED WITH (FUNCTION = 'partial(3,"XXXXXX",3)')
{% endhighlight %}

### End Result ###
As a StandardUser if we query the Employee table with our new masks we now get this...

![Masked Row Results]({{site.url}}/content/images/2017-data-masking/masked-row.PNG)

### Dropping A Dynamic Data Mask ###
If you want to remove a mask then you can use the alter column syntax with DROP MASKED.

{% highlight sql %}
ALTER TABLE dbo.Employee ALTER COLUMN PersonalPhone DROP MASKED
{% endhighlight %}

## Things To Note ##
1. Data masking does not prevent updates so although the StandardUser above can't see the unmasked data it can reset it.
1. Always Encrypted columns can't be masked.
1. Data masking does not protect against users range checking the data to attempt to work out what it is. For example in the email sample above if the StandardUser executed an if statement checking if the email is equal to 'joe@home.com' it would return true. Also the salary field would still allow things like where statements that return results in a given range to work out what the underlying value is.
1. Computed columns can't be masked.