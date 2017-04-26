---
layout: post
title: Entity Framework 6 Interceptors
date: '2015-01-27 07:34:04'
---


Interceptors in Entity Framework 6 allow you to hook in to before and after query events. The before events even allow you to modify the query that is about to be run. To write an interceptor you need to derive a class from DbCommandInterceptor and override the events you want to hook in to...

```language-csharp
public class CommandInterceptor : DbCommandInterceptor
{
	public override void NonQueryExecuting(DbCommand command, DbCommandInterceptionContext<int> interceptionContext)
	{
		base.NonQueryExecuted(command,interceptionContext);
	}

	public override void NonQueryExecuted(DbCommand command, DbCommandInterceptionContext<int> interceptionContext)
	{
		base.NonQueryExecuted(command, interceptionContext);
	}

	public override void ScalarExecuting(DbCommand command, DbCommandInterceptionContext<object> interceptionContext)
	{
		base.ScalarExecuting(command, interceptionContext);
	}

	public override void ScalarExecuted(DbCommand command, DbCommandInterceptionContext<object> interceptionContext)
	{
		base.ScalarExecuted(command, interceptionContext);
	}

	public override void ReaderExecuting(DbCommand command, DbCommandInterceptionContext<DbDataReader> interceptionContext)
	{
		base.ReaderExecuting(command,interceptionContext);
	}

	public override void ReaderExecuted(DbCommand command, DbCommandInterceptionContext<DbDataReader> interceptionContext)
	{
		base.ReaderExecuted(command, interceptionContext);
	}
}
```

To tell EF to use this new interceptor you just need to put this someone in the startup of your app

```language-csharp
DbInterception.Add(new CommandInterceptor());
```

This gives you a lot of hooks to enable logging/profiling when troubleshooting. As for modifying queries you have full access to the DbCommand, this always makes me remember the issues I used to have debugging triggers with that in mind use this with care as it's not always obvious to the developer why their query gets changed. 