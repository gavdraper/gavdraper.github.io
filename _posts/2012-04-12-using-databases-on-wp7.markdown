---
layout: post
title: Using Databases on WP7
date: '2012-04-12 11:27:00'
---

<p>From version 7.5 (Mango) WP7 runs a version of SQL Server CE which is all setup waiting for your applications to make use of it. The good news is that this couldn't be easier. One note though the phone does not include ADO.Net so the only way to query the database is via LinqToSql not that this is a problem.</p> <p>Lets run through a quick example of how this works...</p> <p>Create a new Windows Phone Application project in Visual Studio making sure to set the Windows Phone OS version to 7.1</p> <p><a href="/content/images/WPImport/2012/04/projecttarget.png"><img style="background-image: none; border-right-width: 0px; padding-left: 0px; padding-right: 0px; display: inline; border-top-width: 0px; border-bottom-width: 0px; border-left-width: 0px; padding-top: 0px" title="projecttarget" border="0" alt="projecttarget" src="/content/images/WPImport/2012/04/projecttarget_thumb.png" width="373" height="178"></a></p> <p>Add a reference to System.Data.Linq.</p> <p>Next we need to define our model, create a Models folder in your project and add a class called Film to it. Edit your new Film class to look like this….</p><pre class="brush: csharp; gutter: false; toolbar: false;">using System;
using System.Data.Linq.Mapping;

namespace DatabasesExample.Models
{
    [Table]
    public class Film
    {
        [Column(IsPrimaryKey = true, IsDbGenerated = true)]
        public int Id { get; set; }
        [Column(CanBeNull=false)]
        public string Name { get; set; }
        [Column]
        public DateTime ReleaseDate { get; set; }
        [Column]
        public string Description { get; set; }
    }
}
</pre>
<p>You can see we are making use of the Linq Mapping attributes in order to describe how the table schema should look. We then need to create a DataContext class, to do this create a new class in Models called MyContext and make it look like this…..</p><pre class="brush: csharp; gutter: false; toolbar: false;">using System.Data.Linq;

namespace DatabasesExample.Models
{
    public class MyContext : DataContext
    {
        public Table&lt;film&gt; Films;

        public MyContext() : base("DataSource=isostore:/MyDatabase.sdf;") { }
    }
}
</pre>
<p>You can change MyDatabase to be whatever name you want your database to be stored under in Isolated Storage. The last thing that needs to be done before we can start accessing our database is to tell the application to physically create the database. This can be done by adding the following code to your main App.cs constructor...</p><pre class="brush: csharp; gutter: false; toolbar: false;">using (var myContext = new MyContext())
{
    if (!myContext.DatabaseExists())
        myContext.CreateDatabase();
}
</pre>
<p>On start your application will then check if the database exists, if it doesn't it will create it.</p>
<p>We can now access it using LinqToSql, lets say we wanted to add a new film to the films table...</p><pre class="brush: csharp; gutter: false; toolbar: false;">var myContext = new MyContext();
var ConAir = new Film { Name="Con Air", Description="Planes And Guns", ReleaseDate = new DateTime(1997,6,2) };
myContext.Films.InsertOnSubmit(ConAir);
myContext.SubmitChanges();
</pre>
<p>Or if we wanted to get a list of all the films...</p><pre class="brush: csharp; gutter: false; toolbar: false;">var myContext = new MyContext();
var films = (from film in myContext.Films select film).ToList() ;
</pre>
<h3>Schema Modifications</h3>
<p>Unfortunately updating a tables schema or adding new tables isn't as simple as just changing the model or the context. The best way to achieve this is to version your database and use the DatabaseSchemaUpdater object. For this example lets assume the database in its current state is version 0 (The default version) and we are working on changes to make it version 1. Lets assume we have created a new MusicAlbum class much like our Film class and updated our MyContext to have a second table called MusicAlbums. In your main app.cs class add the following </p><pre class="brush: csharp; gutter: false; toolbar: false;">private int dbVersion = 1;
</pre>
<p>Now we need to modify our DB creation code to look like this....</p><pre class="brush: csharp; gutter: false; toolbar: false;">using (var myContext = new MyContext())
{
    if (myContext.DatabaseExists() == false)
    {
        myContext.CreateDatabase();
        DatabaseSchemaUpdater dbUpdater = myContext.CreateDatabaseSchemaUpdater();
        dbUpdater.DatabaseSchemaVersion = dbVersion;
        dbUpdater.Execute();
    }
    else
    {
        DatabaseSchemaUpdater dbUpdater = myContext.CreateDatabaseSchemaUpdater();
        if (dbUpdater.DatabaseSchemaVersion &lt; dbVersion)
        {
            if (dbUpdater.DatabaseSchemaVersion &lt; 1)
                dbUpdater.AddTable&lt;musicalbum&gt;();
            dbUpdater.DatabaseSchemaVersion = dbVersion;
            dbUpdater.Execute();
        }
    }
}
</pre>
<p>The default version for a database schema is zero and as we never specified a version when we first created the database that is what version our database currently is. So if you run the application with the above changes (Assuming you have created a MusicAlbum class and updated the MyContext class to have the new table) then the new MusicAlbum table will get created as soon as it launches. The DatabaseSchemaUpdater class also has methods for AddIndex and AddColumn. When adding columns you need to either make the column nullable or set a default value so existing records can be populated. This can be done by using attributes on the property like this... </p><pre class="brush: csharp; gutter: false; toolbar: false;">[Column(DbType = "int DEFAULT 0 NOT NULL ")]
public int Rating{get;set;}
</pre>
<p>One thing to note is that in the current version there is no way for the DatabaseSchemaUpdater to remove columns so be a bit careful with what you create.</p>