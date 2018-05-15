---
layout: post
title: Managing SQL Servers With Policy Based Management
date: '2018-05-15 06:12:24'
---
Policy Based Management has been in SQL Server since 2008 and allows you to define policies that can report issues when certain conditions are violated. It can do this in a couple of ways...

- On Demand : You execute the policy manually
- On Schedule : A SQL Agent job executes the policy on a schedule
- On Change : Depending on the policy you may be able to use on demand which uses DDL triggers to trigger the policy to run when a change is made that meets the policy condition. For On Change policies you have the option to prevent the change if it violates the policy or let the change go through but log the issue. 

Policy based management comes with hundreds of facets which are basically objects and proprties on those objects that you can evaluate and fail the poilcy if it doesnt match an expect result. You can also define custom conditions using TSQL if none of the existing facets meet your requirements.

I'm going to run through a couple of examples of policies that you could create both using built in facets and custom TSQL...

## Deny Create Index If Index Name Contains Missing Index ##
Firstly lets create a rule that will fail if anyone creates an index with missing_index in the name and schedule this to run every morning.

First up we need to define the conditions for this policy. Let's create a new condition that will ignore system databases from our check...

![New Condition]({{site.url}}/content/images/2018-policy-based-management/new-condition.png)

![Ignore Sys Databases Condition]({{site.url}}/content/images/2018-policy-based-management/ignore-system-databases.PNG)

Now create another condition to check the index name does not contain the string missing_index...

![Check For Missing_Index In Name Condition]({{site.url}}/content/images/2018-policy-based-management/new-condition-wizard.PNG)

With these 2 conditions created we can now define a policy that runs on all databases that are not system databases and checks every index in them for missing_index in the name...

![New Policy]({{site.url}}/content/images/2018-policy-based-management/new-policy.png)

![New Policy Wizard]({{site.url}}/content/images/2018-policy-based-management/new-policy-wizard.PNG)

![New Schedule]({{site.url}}/content/images/2018-policy-based-management/new-schedule.PNG)

Once this is created if you look under SQL Agent\Jobs in SSMS you'll see a new job has been created to run our policy at the scheduled time that is named something like

    syspolicy_check_schedule_FC8D54AD-2B54-4AF7-A349-5F87CC7B1EE2

If you now execute the job you'll then be able to view failures by looking at the history under Policy Management...

![View History]({{site.url}}/content/images/2018-policy-based-management/view-history.png)

![View Failures]({{site.url}}/content/images/2018-policy-based-management/failures.PNG)

Using this process you can have a good deal of your morning checks run before you come in to work then review the results under the policy management history.

## Check Full Recovery Databases Have Done Log Backups Within X Minutes ##

## On Demand/On Demand Prevent ##
Only some facets are available to be used with the On Change evaluation mode as this works using DDL Triggers, facets that can't be triggered in this way can't be used with the On Change option.

The facets available are all stored in msdb..syspolicy_management_facets which has an execution_mode field that defines what modes you can use (Demand, Schedule, On Change Log, On Change Prevent). This field is a bitwise flag and can be checked with this script

{% highlight sql %}
select 
	facets.management_facet_id Id,
	facets.name Name,
	CASE WHEN execution_mode & 1 = 1 THEN 1 ELSE 0 END AS OnChangePrevent,
	CASE WHEN execution_mode & 2 = 2 THEN 1 ELSE 0 END AS OnChangeLogOnly,
	CASE WHEN execution_mode & 4 = 4 THEN 1 ELSE 0 END AS OnSchedule
FROM
	msdb..syspolicy_management_facets facets
{% endhighlight %}

You can also query what events a facet can use by looking in msdb.dbo.syspolicy_facet_events. 

Armed with this information let's create a new policy that uses a condiation that supports On Change Prevent. Let's look at the StoredProcedure facet as in the above script it shows as supporting On Change Prevent, we can then look at what events this facet supports by querying the events table on it's ID...

{% highlight sql %}
SELECT * FROM msdb.dbo.syspolicy_facet_events WHERE management_facet_id = 61
{% endhighlight%}

![Facet Event List]({{site.url}}/content/images/2018-policy-based-management/facet-events.PNG)

We can see here using this facet our policy will be called for all those events. Based on this lets create a new policy that will prevent anyone prefixing stored procedure name with "sproc". As before we need to create a new condition...

![SP Condition]({{site.url}}/content/images/2018-policy-based-management/sp-condition.PNG)

Now create a new policy that uses our new check condition against all databases condition with the evaulation mode set to On Change Prevent...

![SP Policy]({{site.url}}/content/images/2018-policy-based-management/sp-policy.PNG)

If you then try to create a stored procedure that violated this rule it will fail...

{% highlight sql %}

CREATE PROCEDURE sproc_MyProcedure
AS
SELECT 1 
{% endhighlight %}

![SP Policy Prevent Error]({{site.url}}/content/images/2018-policy-based-management/error.PNG)