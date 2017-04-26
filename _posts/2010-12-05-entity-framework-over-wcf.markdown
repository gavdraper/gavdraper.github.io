---
layout: post
title: Entity Framework Over WCF
date: '2010-12-05 12:03:32'
---

Iâ€™ve been working with a new WCF project that uses the Entity Framework for its data
access. Iâ€™ve had all sorts of problems due to the fact the because its service based
the lifetime of the database context is just the lifetime of that 1 method call.

I spent a while working on a Repository class for the WCF service to wrap around.
In the end this is what I came up with

As Iâ€™m using my own POCO objects rather than the Entity Framework generated objects
I extended the ObjectContext
<pre class="brush: csharp; toolbar: false;">public class DBContext: ObjectContext
{
   public ObjectSet&lt;Sale&gt; Sales{ get; set; }
   public ObjectSet&lt;SaleContact&gt; SaleContacts{ get; set; } 

   public DBContext() : this("name=DBContextEntities") { }
   public DBContext(string connectionString) : base(connectionString, "DBContextEntities")
   {
      Sales= CreateObjectSet&lt;Sale&gt;();
      SaleContacts= CreateObjectSet&lt;SaleContact&gt;();
      //As I'm using POCO objects Proxy objects are not needed
      ContextOptions.ProxyCreationEnabled = false;
   } 

   //The following helper methods are called from my repositories when re-attaching objects
   public void SetModified(object entity)
   {
      ObjectStateManager.ChangeObjectState(entity, EntityState.Modified);
   }
   public void AttachModify(string entitySetName, object entity)
   {
      AttachTo(entitySetName, entity);
      SetModified(entity);
   }
   public void AttachModify(string entitySetName,IEnumerable&lt;object&gt; entities)
   {
      if(entities!=null)
         foreach (var entity in entities)
            AttachModify(entitySetName,entity);
   }
}</pre>
The two POCO objects in this case are Sale and SaleContact and they look like this
<pre class="brush: csharp; toolbar: false;">[DataContract]
public class Sale
{
   [DataMember] public byte[] Version { get; set; }
   [DataMember] public int Id { get; set; }
   [DataMember] public string ItemName{ get; set; }
   [DataMember] public decimal ItemPrice{ get; set; }
   [DataMember] public virtual List&lt;SaleContact&gt; SaleContacts{ get; set; }
} 

[DataContract]
public class SaleContact
{
   [DataMember] public byte[] Version { get; set; }
   [DataMember] public int Id { get; set; }
   [DataMember] public string Name { get; set; }
   [DataMember] public string Address{ get; set; }
   [DataMember] public int SaleId{ get; set; }
}</pre>
Then the sale repository creates the contexts and manages CRUD operations
<pre class="brush: csharp; toolbar: false;">public class SaleRepository
{
   public static Sale New(Sale sale)
   {
      using (var ctx = new DBContext())
      {
         ctx.Sales.AddObject(sale);
         ctx.SaveChanges();
         return sale;
      }
   } 

   public static Sale Update(Sale sale)
   {
      using (var ctx = new DBContext())
      {
         ctx.AttachModify("Sales", sale);
         ctx.AttachModify("SaleContacts",sale.SaleContacts);
         try
         {
            ctx.SaveChanges();
            return sale;
         }
         catch (OptimisticConcurrencyException)
         {
            //As this is being used over WCF Convert exception to FaultException.
            //Maybe this should be moved into the WCF class
            throw new FaultException(ServiceErrors.ConcurrencyError,new FaultCode(ServiceErrorCodes.ConcurrencyCode));
         }
      }
   } 

   public static List&lt;Sale&gt; GetAll()
   {
      using (var ctx = new DBContext())
      {
         var sales =
            (from s in ctx.Sales.Include("SaleContacts") select s).ToList();
         return sales;
      }
   } 

   public static Sale GetById(int id)
   {
      using (var ctx = new DBContext())
      {
         var sale=
            (from s in ctx.Sales.Include("SaleContacts") where s.Id == id select s).FirstOrDefault();
         return sale;
      }
   }
}</pre>
Touch wood it seems to work quite well, I'll try to put together a step by step in the next week.