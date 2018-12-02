---
layout: post
title: SQL Server Minimum Memory Setting
date: '2018-04-29 19:06:57'
tags: maintenance
---
There is almost never a reason to set this setting unless you're running other apps on the same server as your SQL Instance and are trying to stop them taking the memory you want the SQL Server to have. It's always preferable to have your SQL Server be the only thing running on the server but in the real world there are always reasons why this can't be the case.

If you are setting this to reserve memory for SQL you should be aware that this setting probably doesn't behave how you would expect. You'd probably think that if you set this setting to 50GB assuming the server has at least this then the instance will always use at least this amount. This is not true, SQL Server will not reserve memory based on what you set here. If you restart an instance with the minimum set it wont straight away allocate that amount of memory, instead the memory will grow as it normally does. What it does do is stop SQL Server releasing memory that will take it below that amount, so if the memory in use has grown to 51GB over a few hours and SQL Server releases some it wont release any more than 1GB assuming you've set the minimum limit to 50GB.

Knowing it works this way probably gives even less reason to ever set this setting but I'm sure there are some use cases, If you have one please let me know in the comments as I'd be interested to hear more.



