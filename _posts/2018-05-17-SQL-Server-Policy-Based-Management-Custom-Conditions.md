---
layout: post
title: Creating Custom Conditions In SQL Server Policy Based Management
date: '2018-05-17 13:34:01'
---
In my last [post](https://gavindraper.com/2018/05/16/Managing-SQL-Servers-With-Policy-Based-Management/) I covered some basic policy based management examples, in this post I want to cover writing custom conditions in TSQL.

Let's image we want a policy that makes sure every table in our database has a RowVersion type field in it. As in the previous post you would first create a condition to pick the databases for the policy to run on. Then we need a condition that will fail if any table doesn't have a RowVersion field, as there are no facets built in for this we'll have to write a custom one.

The TSQL to do this for a given table looks like this

{% highlight sql %}
IF EXISTS(
      SELECT *
      FROM sys.columns
      WHERE
         Object_id = OBJECT_ID('MyTableWithRowVersion')
         AND system_type_id = 189
)
   SELECT 1
ELSE
   SELECT 0
{% endhighlight %}

We can replace the hard coded table name in the above example with @@ObjectName and the policy will replace that when it runs with the name of each object that it tests against.

{% highlight sql %}
IF EXISTS(
      SELECT *
      FROM sys.columns
      WHERE
         Object_id = OBJECT_ID(@@ObjectName)
         AND system_type_id = 189
)
   SELECT 1
ELSE
   SELECT 0
{% endhighlight %}

To use this we'll create a new condition with a facet of table and instead of picking a field click the ellipsis button to the right of it so we can enter out custom SQL. You need to use the ExecuteSql function and tell it we're returning a number...

{% highlight sql %}
ExecuteSql('Numeric', '
IF EXISTS(
      SELECT *
      FROM sys.columns
      WHERE
         Object_id = OBJECT_ID(@@ObjectName)
         AND system_type_id = 189
)
   SELECT 1
ELSE
   SELECT 0'
)
{% endhighlight %}

We can then finish the condition by putting 0 into the value so it will error for anything that returns 1 (No RowVersion found).

![Has ROWVERSION Condition]({{site.url}}/content/images/2018-custom-condition/has-row-version.PNG)

You can then create a policy that targets your database and uses the condition above to check for the existence of ROWVERSION.

![Has ROWVERSION Policy]({{site.url}}/content/images/2018-custom-condition/has-policy.PNG)
