---
layout: post
title: SQL Server Unique Constraints With Where Conditions
date: '2017-08-02 19:05:38'
---

Imagine we have the following table

Users
| Field | Type |
| --- | --- |
| Id | INT |
| Username | NVARCHAR |
| Deleted | BIT | 

Then imagine we want Username to be unique for non deleted users. Normally we would make a field unique by doing something like this...

{% highlight sql %}
ALTER TABLE dbo.Users ADD CONSTRAINT uq_username UNIQUE(Username)
{% endhighlight %}

This will fail as multiple deleted users can have the same username in our system. To make a constraint that only constrains a subset of data we need to user a filtered index and make that unique, in the example above that looks like this

{% highlight sql %}
CREATE UNIQUE INDEX ndx_non_deleted_username ON dbo.Users(username) WHERE Deleted = 0
{% endhighlight %}

We can then do this without any errors

{% highlight sql %}
INSERT INTO dbo.users(username,deleted)
VALUES('gavin',1),
        ('gavin',1),
        ('gavin',0)
        {% endhighlight %}

Ir will also successfully error if we try to create a second non deleted user with the same name

{% highlight sql %}
INSERT INTO dbo.users(username,deleted)
VALUES('gavin',0)
{% endhighlight %}