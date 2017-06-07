---
layout: post
title: SQL Server 2016 Row Level Security In Action
date: '2017-06-07 08:35:47'
---
Row Level security was introduced with SQL Server 2016 and allows us to specify predicates for what rows can be accessed in the context of for example the logged in user.

For the example in this post imagine we have a multi tenant database and currently the walls that stop Client A seeing Client B's data are all in the application layer. This means any bug in that application code could mean a client gets shown another clients data (Clients don't like this). With Row Level security we can add another line of defense against this by saying all data in Table X with a client id of 1 can only be seen by the database user ClientA. Let's set this up...

{% highlight sql %}
CREATE TABLE dbo.Staff
(
    Id INT IDENTITY PRIMARY KEY,
    ClientId INT,
    FullName NVARCHAR(100)	
)

INSERT INTO dbo.Staff(ClientId, FullName)
VALUES  
    (1,'Claire Temple'),
    (1,'Luke Cage'),
    (1,'Jessica Jones'),
    (2, 'Bruce Wayne'),
    (2, 'Barry Allen')
{% endhighlight %}

Let's imagine our application calls a stored procedure to list all staff in the client of the current user...

{% highlight sql %}
CREATE PROCEDURE dbo.GetAllStaff
(
    @ClientId INT
)
AS
SELECT * FROM dbo.Staff WHERE ClientId = @ClientId
{% endhighlight %}

If we run that with a ClientId of 1 we can see we get just the users in client 1...

{% highlight sql %}
EXEC dbo.GetAllStaff @ClientId = 1
{% endhighlight %}

![Staff By Client]({{site.url}}/content/images/2017-row-security/staff-by-client.JPG)

Let's then imagine that as part of a change the WHERE ClientId = @ClientId part of our procedure gets accidentally commented out and released to production.

{% highlight sql %}
CREATE PROCEDURE dbo.GetAllStaff
(
    @ClientId INT
)
AS
SELECT * FROM dbo.Staff --WHERE ClientId = @ClientId
{% endhighlight %}

![All Staff]({{site.url}}/content/images/2017-row-security/all-staff.JPG)

Oops, Client A can  now see Client B's data. This is a fairly simple example but the more complex the system the easier it is for bugs like this to creep in. Row level security can help combat this by making sure the database user for Client A can never see Client B's data and vise versa.

Let's look at locking this down...

For the sake of the demo I'm going to create 2 users without logins to represent Client A and Client B, in the real world you would create logins too and have your application use the relevant login depending on the client...

{% highlight sql %}
CREATE USER ClientAUser WITHOUT LOGIN
GRANT SELECT TO ClientAUser
GRANT EXECUTE TO ClientAUser
CREATE USER ClientBUser WITHOUT LOGIN
GRANT SELECT TO ClientBUser
GRANT EXECUTE TO ClientBUser
{% endhighlight %}

Now lets create a table to map between these users and their ClientId...

{% highlight sql %}
CREATE TABLE dbo.ClientLogin
(
    ClientUser NVARCHAR(20) PRIMARY KEY,
    ClientId INT
)
INSERT INTO dbo.ClientLogin(ClientUser,ClientId)
VALUES
    ('ClientAUser',1),
    ('ClientBUser',2)
{% endhighlight %}

Next we need a predicate function to check based on the user if they can access that given data...

{% highlight sql %}
CREATE FUNCTION dbo.fn_StaffSecurityPredicate(@ClientId INT)
    RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN
    SELECT 1 AS fn_result
    FROM dbo.Staff
        INNER JOIN dbo.ClientLogin ON ClientLogin.ClientId = Staff.ClientId
    WHERE 
        ClientLogin.ClientUser = CURRENT_USER AND
        Staff.ClientId = @ClientId
{% endhighlight %}

Lastly we need to apply this predicate to our table so it's run when data is requested...

{% highlight sql %}
CREATE SECURITY POLICY StaffByClient
	ADD FILTER PREDICATE dbo.fn_StaffSecurityPredicate(ClientId)
	ON dbo.Staff
	WITH(STATE=ON)
{% endhighlight %}

If we now run our procedure again with the commented out client filter we get no results...

{% highlight sql %}
EXEC dbo.GetAllStaff @ClientId = 1
{% endhighlight %}

![No Results]({{site.url}}/content/images/2017-row-security/no-staff.JPG)

This is because we're not yet using one of our new users that maps to a client. If we change our EXEC statement to run in the context of ClientAUser we should get some data back...

{% highlight sql %}
EXECUTE AS USER = 'ClientAUser';  
EXEC dbo.GetAllStaff @ClientId = 1
REVERT;   
{% endhighlight %}

As you can see even though our application still has a bug allowing Client A's data to be seen by Client B we've been protected by row level security.

![Staff By Client]({{site.url}}/content/images/2017-row-security/staff-by-client.JPG)

The same will also work if we do

{% highlight sql %}
SELECT * FROM dbo.Staff
{% endhighlight %}

Depending on the logged in user we will only ever see staff for the Client we are registered in.

UPDATES and DELETES will also not occur on data that doesn't pass our predicate. One thing to note however is that inserts will still work, so Client A can insert a record into our staff table with Client B's client Id so that will still have to be safeguarded against in other ways.


