---
layout: post
title: SQL Server Error Handling
date: '2018-05-20 20:34:01'
---
## TRY/CATCH RAISERROR Pre SQL Server 2012 ##
Since SQL Server 2005 we've had TRY CATCH syntax in SQL Server handle errors. If an error occurs inside a try block execution will leave the try block and enter the catch statement It's inside the catch statement that you can do any error handling ROLLBACKs, Logging, Data fixes etc and then either rethrow the error or swallow it so nothing higher up knows anything went wrong. Until SQL Server 2012 throwing an error involved the use of the RAISERROR command. See below for a simple example of this...

{% highlight sql %}
BEGIN TRAN
BEGIN TRY
   /*Data modifications
   ...*/
   SELECT 1/0 /*Simulate Divide by zero exception*/
   COMMIT
END TRY
BEGIN CATCH
   IF @@TRANCOUNT > 0
      ROLLBACK;
   /*Perform any error handing steps
   ...*/
   /*Rethrow Exception*/
   DECLARE @Msg NVARCHAR(200) = ERROR_MESSAGE()
   DECLARE @Severity INT = ERROR_SEVERITY()
   DECLARE @State INT = ERROR_STATE()
   RAISERROR(@Msg, @Severity, @State)
END CATCH
{% endhighlight %}

A couple of things to notice here...

- It's a bit of a pain to rethrow an exception as we must first call the error functions to construct a new error based on the original.
- The line number that comes back in the exception is the line number of the RAISERROR call not the line number the actual error occurred on.

Both of the above are improved in SQL Server 2012 and I'll cover that shortly. First lets quickly look at throwing custom exceptions with RAISERRROR

{% highlight sql %}
BEGIN TRAN
BEGIN TRY
   /*Data modifications
   ...*/
   IF NOT EXISTS(SELECT TOP 1 * FROM User)
      RAISERROR('No Users',16,1)
   COMMIT
END TRY
BEGIN CATCH
   IF @@TRANCOUNT > 0
      ROLLBACK;
   /*Perform any error handing steps
   ...*/
   /*Rethrow Exception*/
   DECLARE @Msg NVARCHAR(200) = ERROR_MESSAGE()
   DECLARE @Severity INT = ERROR_SEVERITY()
   DECLARE @State INT = ERROR_STATE()
   RAISERROR(@Msg, @Severity, @State)
END CATCH
{% endhighlight %}

The first parameter we're passing in is the message text we want our error to contain. You can instead pass in the ID of a system error message and have it use that.

The second parameter is the severity level, more info on these can be found [here](https://docs.microsoft.com/en-us/sql/relational-databases/errors-events/database-engine-error-severities?view=sql-server-2017). In our case we're using 16 which is a severity that is classified as an error rather than info but not so serious it won't allow us to handle it.

The last parameter is state, if we're raising the same error in multiple places we would give each one a different state to show where it came from.

One thing to beware of with RAISERROR is if it's not in a try block the execution will continue after your RAISERROR line...

{% highlight sql %}
PRINT 'Before Error'
RAISERROR('No Users',16,1)
PRINT 'After Error'
{% endhighlight %}

If you run this you'll see both prints are run. If this was inside a try block only the first print would run and then control would switch to the catch block.

## TRY/CATCH THROW Post SQL Server 2012 ##
Life got a whole load easier with the THROW statement. All the code we wrote with raise error to bubble the correct errors up is no longer needed. Instead we can just call THROW from inside our catch statement and the error will be re-thrown.

We could rewrite the first example in this post using the 2012 syntax like this...

{% highlight sql %}
BEGIN TRAN
BEGIN TRY
   /*Data modifications
   ...*/
   SELECT 1/0 /*Simulate Divide by zero exception*/
   COMMIT
END TRY
BEGIN CATCH
   IF @@TRANCOUNT > 0
      ROLLBACK;
   /*Perform any error handing steps
   ...*/
   /*Rethrow Exception*/
   THROW;
END CATCH
{% endhighlight %}

You'll notice this approach as well as being cleaner also displays the correct line number in the error message.

Like RAISERROR you can also use throw to throw custom errors. For example...

{% highlight sql %}
THROW 50000, 'Error Message', 1
{% endhighlight %}

- The first argument is the error number and must be between 50000  and 2147483647
- The second argument is the error message
- The third is the state

Where as RAISERROR will continue executing the batch when outside of a try block THROW will not...

{% highlight sql %}
PRINT 'Before Error';
THROW 50000, 'Error Message', 1
PRINT 'After Error'
{% endhighlight %}

The second print wont run as throw will end the batch unless you catch it.

## Check State Of Transaction In Catch ##

### @@TRANCOUNT ###
Before we call COMMIT or ROLLBACK in a catch we first need to check there is an active transaction we can call these on. If an error was raised outside of a transaction calling ROLLBACK in the catch will cause another error. One of the ways we can handle this is to check the count of transactions is greater than zero.

For example...

{% highlight sql %}
IF @@TRANCOUNT > 0
    ROLLBACK
{% endhighlight %}

### XACT_ABORT/XACT_STATE  ###
If you set XACT_ABORT to on then things behave slightly differently, normally non fatal errors will fail the statement but if you turn this on the whole batch will fail and the transaction will be changed to a state where it can't be committed. In this case you'll have a transaction count of greater than one but if you call commit on it then it will fail...

{% highlight sql %}
SET XACT_ABORT OFF
BEGIN TRAN
BEGIN TRY
   SELECT 1/0 /*Simulate Divide by zero exception*/
END TRY
BEGIN CATCH
   IF @@TRANCOUNT > 0
      COMMIT
END CATCH
{% endhighlight %}

I know we're not actually changing any data here but it will allow us to finish the transaction by calling commit in our catch. If you now run that again but with SET XACT_ABORT ON you'll see that it errors and won't allow the commit.

![XACT_ABORT Error]({{site.url}}/content/images/2018-error-handling/xact.PNG)

You can see by turning this on we can enforce no data be saved in transactions where any single statement has an error.

If you ever want to perform a commit in a catch (An odd scenario) then it's not enough to just check @@TRANCOUNT, you need to instead check XACT_STATE to make sure the transaction is in a state that is allowed to be committed. Some severity errors will put the transaction in a state where it's marked as readonly and no commits can be performed. XACT_STATE returns 1 of 3 values...

- 1 - There is an active transaction that can be committed.
- 0 - There are no transactions
- -1 There is an active transaction that is in a state that can't be committed. This will automatically be rolled back when the batch finishes.

Let's change the above query to use XACT_STATE...

{% highlight sql %}
SET XACT_ABORT ON
BEGIN TRAN
BEGIN TRY
   SELECT 1/0 /*Simulate Divide by zero exception*/
END TRY
BEGIN CATCH
   IF XACT_STATE() = 1
      COMMIT
   ELSE IF XACT_STATE() = -1
      ROLLBACK
END CATCH
{% endhighlight %}

Here we're using XACT_ABORT ON for force any error to put the transaction in a readonly state. If you run this code it will fall into the rollback in the catch block because of this.

Another thing to note here is as in the example above where we showed RAISERRORR continues execution of the batch if there is no TRY/CATCH this same behavior occurs with XACT_ABORT ON. RAISERROR does not use XACT_ABORT where as THROW does...

{% highlight sql %}
 SET XACT_ABORT ON
 PRINT 'Before Error';
 RAISERROR('Error',16,1);
 PRINT 'After Error'
 {% endhighlight %}

 Both prints here will run even with SET XACT_ABORT ON. If you want a batch to fail on error then use THROW. In fact you should pretty much always use THROW unless you have a good reason to want the RAISERROR behavior.

 I mentioned above that with XACT_ABORT OFF (The default) some levels of error will not fail the batch but instead just fail the statement. To see this run the following..

 {% highlight sql %}
SET XACT_ABORT OFF

 DROP TABLE Test
 CREATE TABLE Test(Blah NVARCHAR(100) NOT NULL)

 INSERT INTO Test VALUES(NULL)
 INSERT INTO Test VALUES('Test')
 SELECT * FROM Test
 {% endhighlight %}

The first insert will fail as it violates the NOT NULL constraint however the second oen will still run and succeed as can be seen if you run the select runs we have a single row. If you run it again with XACT_ABORT ON you'll see that it fails at the batch level and everything gets rolled back. However outside of an explicit transaction everything before the line that errors will still commit fine...

{% highlight sql %}
 SET XACT_ABORT ON
 INSERT INTO Test VALUES('One')
 INSERT INTO Test VALUES(NULL)
 INSERT INTO Test VALUES('Two')
 {% endhighlight %}

 This will insert a single record of value 'One'. If you want the batch to ROLLBACK before the error as well then you need to wrap it all in a transaction...

 {% highlight sql %}
  SET XACT_ABORT ON
 BEGIN TRAN
 INSERT INTO Test VALUES('One')
 INSERT INTO Test VALUES(NULL)
 INSERT INTO Test VALUES('Two')
 COMMIT
 {% endhighlight %}

 This will insert zero records because one of the statements in the batch had an error.