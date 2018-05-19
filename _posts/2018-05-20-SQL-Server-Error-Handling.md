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

## Check State Of Transaction In Catch ##

### @@TRANCOUNT ###
Before we call COMMIT or ROLLBACK in a catch we first need to check there is an active transaction we can call these on. If an error was raised outside of a transaction calling ROLLBACK in the catch will cause another error. One of the ways we can handle this is to check the count of transactions is greater than zero.

For example...

{% highlight sql %}
IF @@TRANCOUNT > 0
    ROLLBACK
{% endhighlight %}

### XACT_ABORT/XACT_STATE  ###
If you set XACT_ABORT to on then things behave slightly differently, normally non fatal errors will fail the statement but with it on the whole batch will fail and the transaction will be changed to a state where it can't be committed. In this case you'll have a transaction count of greater than one but if you call commit on it then it will fail...

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

When using XACT_ABORT ON if you want to commit anything in the catch then you'll need to use XACT_STATE to check if the transaction is in a committable state, XACT_STATE returns 1 of 3 values...

- 1 - There is an active transaction that can be committed.
- 0 - There are no transactions
- -1 There is an active transaction that is in a state that can't be committed

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