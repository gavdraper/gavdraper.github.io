---
layout: post
title: Entity Framework Navigation Property Not Saving For Re-Attached Entity
date: '2011-01-03 11:29:37'
---

I thought I had the whole disconnected Entity Framework over WCF working well but
this week I came accross a problem to do with Navigation Properties not saving when
you reattach Entities and set them as modified.

Take the following example of a console app that uses EF4 with POCO objects
<pre class="brush: csharp; toolbar: false;">static void Main(string[] args)
{
    Service svc = GetService();
    Console.WriteLine("Service : " + svc.Name + " , Status : " + svc.Status.Name);
    //Change and save Status
    svc.Status = GetStatus("Stopped");
    using (var ctx = new TestEFContext())
    {
        //Status is changed
        Console.WriteLine("Service : " + svc.Name + " , Status : " + svc.Status.Name);
        ctx.AttachModify("Services", svc);
        ctx.AttachTo("Statuses", svc.Status);
        ctx.SaveChanges();
    }
    //Re-fetch service from db and check status
    svc = GetService();
    //Status is set back to its old value!!!!!!!!
    Console.WriteLine("Service : " + svc.Name + " , Status : " + svc.Status.Name);
    Console.ReadLine();
}

private static Service GetService()
{
    using (var ctx = new TestEFContext())
    {
        return ctx.Services.Include("Status").FirstOrDefault();
    }
}

private static Status GetStatus(string name)
{
    using (var ctx = new TestEFContext())
    {
        return ctx.Statuses.Where(n=&gt;n.Name == name).FirstOrDefault();
    }
}

public class Service
{
    [DataMember] public int ServiceID { get; set; }
    [DataMember] public string Name { get; set; }
    [DataMember] public Status Status { get; set; }
}

public class Status
{
    [DataMember] public int StatusID { get; set; }
    [DataMember] public string Name { get; set; }
}</pre>
The Table Schema looks like this

<strong>Service</strong>
ServiceID <em>INT</em>
Name <em>NVARCHAR(40)</em>
StatusID <em>INT</em>

<em></em><strong>Status</strong>
StatusID <em>INT</em>
Name <em>NVARCHAR(20)</em>

The POCO objects look like this

<strong>Service</strong>
ServiceID <em>int</em>
Status <em>Status</em>
Name <em>string</em>

<strong>Status</strong>
StatusID <em>int</em>
Name <em>string</em>

As in previous posts the AttachModify is a method I created in my context to attach
an entity and mark it as modified, it looks like this
<pre class="brush: csharp; toolbar: false;">public void SetModified(object entity)
{
    ObjectStateManager.ChangeObjectState(entity, EntityState.Modified);
}

public void AttachModify(string entitySetName, object entity)
{
    AttachTo(entitySetName, entity);
    SetModified(entity);
}</pre>
When I save the service it updates the name but leaves the StatusID as its previous
value. This seems to be because I'm working disconnected EF doesnt know the navigation
property has been changed and therefore doesn't update the StatusID even though I
explicitly set the Service object to modified. I tried also seting the Status object
to modified but the only effect this has was that it then also updates the Status
table but still no update for the StatusID on the service table.

The solution for this seems to be either keep the foreign keys in your model rather
than just the Navigation Properties or you can do the following
<pre class="brush: csharp; toolbar: false;">        static void Main(string[] args)
        {
            Service svc = GetService();
            svc.Status = GetStatus("Stopped");
            using (var ctx = new TestEFContext())
            {
                var svc2 = ctx.Services.Where(s=&gt;s.ServiceID == svc.ServiceID).FirstOrDefault();
                svc2.Status = ctx.Statuses.Where(n =&gt; n.StatusID == svc.Status.StatusID).FirstOrDefault();
                ctx.ApplyCurrentValues("Services", svc);                                
                ctx.SaveChanges();
            }
        }</pre>
On save this then refetches the entities and any child entities from the database
and uses ApplyCurrentValues on them to apply the updated POCO object values. I realize
that this is no where near as neat as just re-attaching the entities and setting them
to modified but it was the only way I could get this to work. In future if I'm working
in a disconnected way like this I will probably keep the Foreign Keys in my model
then I can avoid having to re-fetch entities from the database every time I want to
save a disconnected POCO object that uses Navigation Properties.