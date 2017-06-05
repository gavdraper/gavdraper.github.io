---
layout: post
title: SQL Server Application Roles
date: '2017-06-05 07:47:47'
---
If you're using Windows Authentication then your Database server knows a lot more about each user and makes permissioning a lot easier. For example if an app is using Windows auth to log it's users into the database server, then we most likely have information like Groups from Active Directory and can permission at both User and Group level. If however Windows Authentication is not an option because there is no centralized Active Directory for example if you're running a public facing web application then you most likely have your app always authenticate to SQL Server as the same user. This makes permissioning more difficult as this one user needs full permission to do everything your app could ever need to do, so any one user of the application has as much access as the next user to the database.

Let's look at an example of this. The following scripts create some tables and schemas in a new database...

{% highlight sql %}
CREATE DATABASE AppRoles
GO
USE AppRoles
GO
CREATE LOGIN [WebAppUser] WITH PASSWORD=N'(123ABC)', DEFAULT_DATABASE=[AppRoles]
GO
CREATE USER [WebAppUser] FOR LOGIN [WebAppUser]
GO

CREATE SCHEMA Accounts
CREATE SCHEMA HR
CREATE TABLE Accounts.AccountsTable 
(
    Id INT IDENTITY PRIMARY KEY
)
CREATE TABLE HR.HumanResourcesTable
(
    Id INT IDENTITY PRIMARY KEY
)
{% endhighlight %}

Let's imagine we put all our Accounts tables in the Accounts schema and all our HR tables in the HR schema. We then have our single WebAppUser that our application will use to login to the SQL Server. If we want to stop certain users of our application from accessing the accounts table we'd have to build that in to the application as there is no way we can do this in SQL Server with a single login. If we were able to user Windows Auth we could have added users for an Accounts group and an HR group then permission them to only access their relevant schemas. This is where Application Roles come in. At this point if we try to select from either table as our WebAppUser it will fail as we've not granted the user any permissions or roles...

To follow along with any examples log in to your SQL Server instance as the WebAppUser login we created above.

{% highlight sql %}
SELECT * FROM Accounts.AccountsTable
SELECT * FROM HR.HumanResourcesTable
{% endhighlight %}

![No permission error]({{site.url}}/content/images/2017-application-roles/no-permission.JPG)

We could now grant select permission to this user for both tables which will stop the error but give the user more access than is possibly needed. What we can do here is create something called an Application Role and give that role access to the table, our application will then be responsible for authenticating as part of that role, this means the application has to be explicit to access. As a result our application is then less likely to for example give an HR user access to an Accounting table. Let's look at an example of setting this up...

{% highlight sql %}
CREATE APPLICATION ROLE AccountsAppRole WITH PASSWORD = '(Acc123)', DEFAULT_SCHEMA =Accounts
CREATE APPLICATION ROLE HRAppRole WITH PASSWORD = '(HR123)', DEFAULT_SCHEMA =HR

GRANT SELECT ON SCHEMA::Accounts TO AccountsAppRole
GRANT SELECT ON SCHEMA::HR TO HrAppRole
{% endhighlight %}

We now have an Accounts application role with select access to the Accounts Schema and an HR application role with access to the HR schema.

What happens if we now run our select statements again?

{% highlight sql %}
SELECT * FROM Accounts.AccountsTable
SELECT * FROM HR.HumanResourcesTable
{% endhighlight %}

![No permission error]({{site.url}}/content/images/2017-application-roles/no-permission.JPG)

It stil fails because we've not authenticated as part of an application role. When our application wants to use either of these tables it has to first authenticate as one of our roles...

{% highlight sql %}
EXEC sp_setapprole 'AccountsAppRole', '(Acc123)'
SELECT * FROM Accounts.AccountsTable
{% endhighlight  %}

The select will then work as we've now got permission through the application role. When you authenticate as an Application Role with sp_setapprole you will stay in that role until you either disconnect or you call sp_unsetapprole. 

Whilst not the most elegant solution it does provide another line of security for sensitive tables that you don't want all users to have access to. 