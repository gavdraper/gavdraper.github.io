---
layout: post
title: Using The Entity Framework With WCF
date: '2010-12-07 12:26:11'
---

I've had a few problems when I first started trying to use the entity framework with
WCF mainly because it needed to be stateless between requests and all the examples/tutorials
I've seen involve the context being held on to for the life of an entity to manage
changes.

I eventually got to a solution that I'm fairly happy with and am going to run through
a simple step by step version of it. In this solution I'm going to be using POCO objects
with the entity framework for this to work you need to download and install the EF4
CTP from <a href="http://www.microsoft.com/downloads/en/details.aspx?FamilyID=4e094902-aeff-4ee2-a12d-5881d4b0dd3e&amp;displaylang=en">Microsoft</a> .

You will also need SQL Server and .Net 4.

First create a new database with the following 2 tables.

<img src="/content/images/WPImport/2010/12/we1.jpg" border="0" alt="" />

In this example we have a products table and a sales table, it's a one too many relationship
between products and sales.

The next step is to create our Entity Model.

<strong><span style="text-decoration: underline;">In visual studio</span></strong>
<ul>
	<li> Create a new solution</li>
	<li> Add a class library project to the solution called WCFEntityData</li>
	<li> Add a WCF service application to the solution called WCFEntityService</li>
	<li> Add a console app to the solution called WCFEntityConsoleTest</li>
</ul>
<strong><span style="text-decoration: underline;">In the WCFEntityData project...</span></strong>
<ul>
	<li> Add a reference to the EF4CTP normally found in C:\Program Filess\Microsoft ADO.NET
Entity Framework Feature CTP4\Binaries</li>
	<li> Delete Class1.cs</li>
	<li> Add a folder called Repositories</li>
	<li> Add a folder called Model</li>
	<li> Add a folder called Entities</li>
</ul>
<strong>The Model</strong>

Add a new "ADO.Net Entity Data Model" called WcfEntityModel to the Model folder.

<img src="/content/images/WPImport/2010/12/we2.jpg" border="0" alt="" />

Point it at your database and give the connection settings a name of â€œDbEntitiesâ€

<img src="/content/images/WPImport/2010/12/we3.jpg" border="0" alt="" />

Click the tables node to model all the tables and give the model a namespace a name
of WcfEntityModel.

<img src="/content/images/WPImport/2010/12/we4.jpg" border="0" alt="" />

You should then have a model that looks like this

<img src="/content/images/WPImport/2010/12/we5.jpg" border="0" alt="" />

You then need to delete the Product navigation property from the Sale entity as it
creates a circular reference which causes problems with serialization.

In the properties for the model set the 'Code Generation Stratergy' to none, this
stops the entity framework generating your entities classes for you as in this case
we are going to build our own POCO objects that are perfect for serialization in WCF.

<img src="/content/images/WPImport/2010/12/we6.jpg" border="0" alt="" />

For each of the objects in your model click the Version field, then in their properties
window set the 'Concurrency Mode' to fixed. This means when we try to update an object
to the database if the version field in the database is different to the one in the
object then someone has edited the object after we retrieved it, this will cause EF
to throw an exception to avoid concurrency issues. The timestamp data type in MS SQL
will automatically get updated every time a record is added or edited so you donâ€™t
need to worry about maintaining this field.

Add a new class to the Model directory called 'WcfEntityContext'. Make this class
look like
<pre class="brush: csharp; toolbar: false;">using System.Collections.Generic;
using System.Data;
using System.Data.Objects;
using WCFEntityData.Entities;

namespace WCFEntityData.Model
{
    public class WcfEntityContext : ObjectContext
    {
        public ObjectSet&lt;Product&gt; Products { get; set; }
        public ObjectSet&lt;Sale&gt; Sales { get; set; }

        public WcfEntityContext() : this("name=DbEntities") { }

        public WcfEntityContext(string connectionString)
            : base(connectionString, "DbEntities")
        {
            Products = CreateObjectSet&lt;Product&gt;();
            Sales = CreateObjectSet&lt;Sale&gt;();
            //Turned off to avoid problems with serialization
            ContextOptions.ProxyCreationEnabled = false;
            ContextOptions.LazyLoadingEnabled = false;
        }

        public void SetModified(object entity)
        {
            ObjectStateManager.ChangeObjectState(entity, EntityState.Modified);
        }

        public void AttachModify(string entitySetName, object entity)
        {
            AttachTo(entitySetName, entity);
            SetModified(entity);
        }

        public void AttachModify(string entitySetName, IEnumerable&lt;Object&gt; entities)
        {
            if (entities != null)
                foreach (var entity in entities)
                    AttachModify(entitySetName, entity);
        }

    }
}</pre>
This class is telling the Entity Framework how to map your Entity Model to your POCO
objects. If Proxy Creation is true the entity framework wraps your POCO objects in
proxy classes for change tracking, as we are working disconnected there will be no
change tracking so I disabled this in the constructor where I also disabled Lazy Loading
as again due to the disconnected state Lazy Loading will not work over WCF. The 3
methods at the end are just helper methods that we will make use of later.

<strong>The Entities</strong>

Now its time to create our entities. In the Entities folder create two classes called
'Product' and 'Sale'

The product class should look like this
<pre class="brush: csharp; toolbar: false;">using System.Collections.Generic;
using System.Runtime.Serialization;

namespace WCFEntityData.Entities
{
    [DataContract]
    public class Product
    {
        [DataMember] public int Id { get; set; }
        [DataMember] public string Name { get; set; }
        [DataMember] public decimal Price { get; set; }
        [DataMember] public byte[] Version { get; set; }
        [DataMember] public virtual IList&lt;Sale&gt; Sales { get; set; }
    }
}</pre>
The sale class should look like this
<pre class="brush: csharp; toolbar: false;">using System;
using System.Runtime.Serialization;

namespace WCFEntityData.Entities
{
    [DataContract]
    public class Sale
    {
        [DataMember] public int Id { get; set; }
        [DataMember] public int ProductId { get; set; }
        [DataMember] public decimal Price { get; set; }
        [DataMember] public DateTime SaleDate { get; set; }
        [DataMember] public byte[] Version { get; set; }
    }
}</pre>
The attributes of DataContract and DataMember are needed for the Entity Framework
to use these objects. Notice on the product object I have a virtual list of sale objects,
making this virtual allows for entity framework to use lazy loading. In this case
I'm going to be turning lazy loading off as it doesn't really work for the example
over WCF as the objects will be disconnected once they hit the client so no lazy loading
can occur. The reason I still made the property virtual is so this could possible
use Lazy Loading in the future.

<strong>The Repository</strong>

Lets start out simple create a new class in the Repositories folder called ProductRepository.
<pre class="brush: csharp; toolbar: false;">using System.Collections.Generic;
using System.Linq;
using WCFEntityData.Entities;
using WCFEntityData.Model;

namespace WCFEntityData.Repositories
{
    public class ProductRepository
    {
        public static Product New(Product product)
        {
            using (var ctx = new WcfEntityContext())
            {
                ctx.Products.AddObject(product);
                ctx.SaveChanges();
                return product;
            }
        }

        public static Product Update(Product product)
        {
            using (var ctx = new WcfEntityContext())
            {
                ctx.AttachModify("Products", product);
                ctx.AttachModify("Sales", product.Sales);
                ctx.SaveChanges();
                return product;
            }
        }

        public static IList&lt;Product&gt; GetAll()
        {
            using (var ctx = new WcfEntityContext())
            {
                var sales = (from s in ctx.Products.Include("Sales") select s).ToList();
                return sales;
            }
        }
    }
}</pre>
This gives us a method to create a new product with any sales you want to add to it
and a method to return all the products and their associated sales. Notice how the
WcfEntityContext is wrapped in a using this is because WCF is stateless so we are
having to create and destroy the context as and when we need it.

<strong><span style="text-decoration: underline;">In the WcfEntityService Project</span></strong>

Now we need to define the methods our service is going to expose.

Add a reference to your WcfEntityData project.

Open app.config in your WcfEntityData project and copy everything between and including &lt;connectionStrings&gt; and &lt;/connectionStrings&gt;. Open web.config in the WcfEntityService project and paste the connection string info right below the opening tag.

<img src="/content/images/WPImport/2010/12/we7.jpg" border="0" alt="" />

Open the IService1.cs file and change it to look like this
<pre class="brush: csharp; toolbar: false;">using System.Collections.Generic;
using System.ServiceModel;
using WCFEntityData.Entities;

namespace WcfEntityService
{
    [ServiceContract]
    public interface IService1
    {
        [OperationContract] IList&lt;Product&gt; ProductGetAll();
        [OperationContract] Product ProductUpdate(Product product);
        [OperationContract] Product ProductNew(Product product);
    }
}</pre>
Open the Service1.svc file and change it to look like this
<pre class="brush: csharp; toolbar: false;">using System.Collections.Generic;
using WCFEntityData.Entities;
using WCFEntityData.Repositories;

namespace WcfEntityService
{
    public class Service1 : IService1
    {
        public Product ProductNew(Product product)
        {
            return ProductRepository.New(product);
        }

        public Product ProductUpdate(Product product)
        {
            return ProductRepository.Update(product);
        }

        public IList&lt;Product&gt; ProductGetAll()
        {
            return ProductRepository.GetAll();
        }
    }
}</pre>
We have now defined three methods on our service that wrap the three methods in our
repository for fetching, inserting and updating products. Right click the service
project and click build so we can then reference it in our other projects.

<strong><span style="text-decoration: underline;">In the WcfEntityConsoleTest Project</span></strong>

Right click the project and click 'Add Service Reference'. As the service is in the
solution just click the discover button and it will find our service, give it a namespace
of "MyService" and click OK.

<img src="/content/images/WPImport/2010/12/we8.jpg" border="0" alt="" />

Now your console app knows about the service it has access to all of its methods.

Open Program.cs and make it look like this
<pre class="brush: csharp; toolbar: false;">using System;
using WcfEntityConsoleTest.MyService;

namespace WcfEntityConsoleTest
{
    class Program
    {
        static void Main(string[] args)
        {
            CreateProduct();
            GetAllProducts();
            UpdateProduct();
            Console.ReadLine();
        }

        private static void CreateProduct()
        {
            var newProduct = new Product()
             {
                 Name = "Skates",
                 Price = (decimal)250.99,
                 Sales = new Sale[]
                     {
                         new Sale(){SaleDate = DateTime.Now, Price = (decimal)240},
                         new Sale(){SaleDate = DateTime.Now, Price = (decimal)230},
                        new Sale(){SaleDate = DateTime.Now, Price = (decimal)235},
                    }
             };
            var client = new Service1Client();
            var savedTransition = client.ProductNew(newProduct);
            Console.WriteLine("Inserted Product ID Is : " + savedTransition.Id.ToString());
            Console.WriteLine("--------------");
        }

        private static void UpdateProduct()
        {
            var client = new Service1Client();
            var products = client.ProductGetAll();
            products[0].Name = "Updated Product";
            client.ProductUpdate(products[0]);
        }

        private static void GetAllProducts()
        {
            var client = new Service1Client();
            var products = client.ProductGetAll();
            foreach (var p in products)
            {
                Console.WriteLine(p.Name + " : " + p.Price);
                if (p.Sales != null)
                    foreach (var s in p.Sales)
                    {
                        Console.WriteLine("   " + s.Id.ToString());
                        Console.WriteLine("   " + s.Price);
                        Console.WriteLine("   " + s.SaleDate.ToShortDateString());
                    }
            }
        }
    }
}</pre>
If you now run the console app, it should insert a new Product with 2 associated sales
and tell you the ID of the newly inserted product. It should then list all the products
in the database and their sales and then update the product with a new name. Try running
it a couple of times to see the product collection grow.

<strong><span style="text-decoration: underline;">Where Next</span></strong>

I know that's a very basic example but it should be a good foundation to working with
the Entity Framework in a disconnected way over WCF.

Obviously you would also want to create a Sales repository for manipulating sales.
You will probably need a GetById method and a GetByName method. The example is also
missing delete methods. All of this should be very simple to implement by just adding
the required CRUD functionality to the Repository and then wrapping it in the service.

This example is also missing any error handling. You will want to implement some sort
of Exception handling on the service so the client can get a helpful exception when
something goes wrong rather than the generic WCF exception. This can be achieved by
using FaultException, information on FaulExceptions can be found <a href="http://msdn.microsoft.com/en-us/library/ee942778.aspx">here</a>.