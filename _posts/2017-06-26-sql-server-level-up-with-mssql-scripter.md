---
layout: post
title: Level Up Your SQL Server Cross Platform Skills With mssql-scripter
date: '2017-06-27 08:05:38'
---
The ability to export a script of database objects has for a long time been a feature of SQL Server Management Studio. With the new OSS cross platform tooling from Microsoft we can now have ths feature everywhere. mssql-scripter allows you to export schema and or data from SQL Server from pretty much any terminal

I do a lot of my SQL Server sandbox stuff on macOS using a Docker image of SQL Server 2017 with VS Code and the mssql plugin as my editor. With the new mssql-scripter tool I can now export SQL scripts of my database objects and data directly from the VS Code terminal.

For example here is my VS Code window listing all the databases in my Docker hosted SQL Server 2017 server...

![List Database In VS Code]({{site.url}}/content/images/2017-mssql-scripter/list-dbs.png)

From here I can open the terminal in VS Code and run the mssql-scripter tool to generate a new SQL Script called InitialDb.sql in my working directory...

![MSSQL-Scripter From VS Code]({{site.url}}/content/images/2017-mssql-scripter/script-db.png)

Once this has run I get my new script file and inside is the SQL Script to create that database and all it's objects...

![DB Creation Script]({{site.url}}/content/images/2017-mssql-scripter/db-create-script.png)

This tool supports a great deal of arguments for specifying what you want to script from the database. For example we can Script Schema only, data only, Schema and Data then we can go down to exactly what types of objects to script for example Statistics, logins etc....

To get a full list of what mssql-scripter supports at the console run...

>   mssql-scripter --help

Here's a small set of the arguments that --help lists...

![mssql-scripter arguments]({{site.url}}/content/images/2017-mssql-scripter/commands.png)

Instructions for installing mssql-scripter on each platform can be found on their [GitHub installation page](https://github.com/Microsoft/sql-xplat-cli/blob/dev/doc/installation_guide.md)