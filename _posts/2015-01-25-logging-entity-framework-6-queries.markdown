---
layout: post
title: Profiling Entity Framework 6 Queries
date: '2015-01-25 08:28:45'
---

Entity Framework 6 introduced some API hooks that you can use to monitor log queries that Entity Framework is generating and running. 

At it's most basic level you can log EF calls by setting your DbContext Database.Log to an Action<string> for example....

```language-csharp 
using System;
using System.Data.Entity;
using System.Linq;

namespace ConsoleApp
{
    class Program
    {
        static void Main(string[] args)
        {
            using (var ctx = new MyContext())
            {
                ctx.Database.Log = Console.Write;
                var users = ctx.Users.ToList();
                Console.ReadLine();
            }
        }
    }

    public class MyContext : DbContext
    {
        public DbSet<User> Users { get; set; }
    }

    public class User
    {
        public int Id { get; set; }
        public string Username { get; set; }
    }
}
```

You can swap Console.WriteLine for any Action<string>, this is a quick win way to log some queries. 

Taking this a step further you can implement custom formatters to change the output of the log by creating a class that derives from DatabaseLogFormatter and setting that class to be your log formatter in your contexts Dbconfiguration, for example...

```language-csharp
using System;
using System.Data.Common;
using System.Data.Entity;
using System.Data.Entity.Infrastructure.Interception;
using System.Linq;

namespace ConsoleApp
{
    class Program
    {
        static void Main(string[] args)
        {
            using (var ctx = new MyContext())
            {
                ctx.Database.Log = Console.Write;
                var users = ctx.Users.ToList();
                Console.ReadLine();
            }
        }
    }

    public class User
    {
        public int Id { get; set; }
        public string Username { get; set; }
    }

    public class MyContext : DbContext
    {
        public DbSet<User> Users { get; set; }
    }

    public class MyContextConfig : DbConfiguration
    {
        public MyContextConfig()
        {
            SetDatabaseLogFormatter((context, writeAction) => 
            	new OneLineFormatter(context, writeAction));
        }
    }

    public class OneLineFormatter : DatabaseLogFormatter
    {
        public OneLineFormatter(DbContext context, Action<string> writeAction) 
        	: base(context, writeAction) {}

        public override void LogCommand<TResult>(DbCommand command, 
        	DbCommandInterceptionContext<TResult> interceptionContext)
        {
            Write(string.Format("{0} Executing \n {1}{2}",
                Context.GetType().Name,
                command.CommandText.Replace(Environment.NewLine, ""),
                Environment.NewLine));
        }

        public override void LogResult<TResult>(DbCommand command, 
        	DbCommandInterceptionContext<TResult> interceptionContext){}
    }
}

```  

I've got a work in progress EF6 profiler project on github ([https://github.com/gavdraper/EF6Profiler](https://github.com/gavdraper/EF6Profiler)). By adding a couple of lines to your app to set a profiler you can have it send profile information via SignalR to a client app much like SQL Profiler. This project uses Entity Framework interceptors which I will cover in a future post. 