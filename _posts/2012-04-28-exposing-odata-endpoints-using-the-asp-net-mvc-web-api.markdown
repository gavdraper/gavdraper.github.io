---
layout: post
title: Exposing OData Endpoints using the ASP.Net MVC Web API
date: '2012-04-28 10:06:18'
---

<p>The new Web API that comes with ASP.Net MVC 4 has been getting a lot of posts on creating RESTful services, however one thing that I’ve not seen mentioned a great deal is it’s ability to expose OData endpoints.</p> <p>It almost makes it too easy, rather than talk about it too much lets run through an example. </p> <p>If you don’t already have it you will need to download ASP.Net MVC 4 from <a href="http://asp.net/mvc">http://asp.net/mvc</a>. At the time of writing this is currently in beta, once its released you will be able to get it from the Web Platform Installer. Alternatively if you are running the Visual Studio 11 beta then you already have MVC 4 installed.</p> <p>Create a new ASP.Net MVC 4 Web Application in Visual Studio</p> <p><a href="/content/images/WPImport/2012/04/OdataNewProject10.png"><img style="background-image: none; border-right-width: 0px; padding-left: 0px; padding-right: 0px; display: inline; border-top-width: 0px; border-bottom-width: 0px; border-left-width: 0px; padding-top: 0px" title="OdataNewProject[10]" border="0" alt="OdataNewProject[10]" src="/content/images/WPImport/2012/04/OdataNewProject10_thumb.png" width="592" height="343"></a></p> <p>When asked to pick a project template choose empty, don’t worry too much about what view Engine you choose as we’re not going to be using it.</p> <p><a href="/content/images/WPImport/2012/04/ODataExampleProjectTemplate5.png"><img style="background-image: none; border-right-width: 0px; padding-left: 0px; padding-right: 0px; display: inline; border-top-width: 0px; border-bottom-width: 0px; border-left-width: 0px; padding-top: 0px" title="ODataExampleProjectTemplate[5]" border="0" alt="ODataExampleProjectTemplate[5]" src="/content/images/WPImport/2012/04/ODataExampleProjectTemplate5_thumb.png" width="361" height="334"></a> </p> <p>To start with we need some data that we are going to expose through our OData endpoint. In this case I’m just going to create a quick model and a repository that populates that model with some dummy data. Don’t worry too much about how this is coded as it’s only purpose is to create dummy data for the example.</p> <p>Create a new class in the models folder called Product and make it look like this….</p><pre class="brush: csharp; gutter: false; toolbar: false;">public class Product
{
    public int Id { get; set; }
    public string ProductName { get; set; }
    public decimal Price { get; set; }
    public int Quantity { get; set; }
    public string Description { get; set; }
}</pre>
<p>We then need a repository to generate some dummy data. Create a new folder in the Models folder called Repositories then create a new class in this folder called ProductRepository. Make your new class look like this…</p><pre class="brush: csharp; gutter: false; toolbar: false;">public class ProductRepository
{
    public IEnumerable&lt;product&gt; Products { get; set; }

    public ProductRepository()
    {
        var products = new List&lt;product&gt;
        {
            new Product{Id=1, ProductName="Carrots", Description="Make you see in the dark", Price=1.99M, Quantity=100},
            new Product{Id=2, ProductName="Cabbage", Description="Yummy", Price=.50M, Quantity=23},
            new Product{Id=3, ProductName="Sprouts", Description="Yuck!", Price=.75M, Quantity=71},
            new Product{Id=4, ProductName="Peas", Description="Small and round", Price=.20M, Quantity=3},
            new Product{Id=5, ProductName="Apples", Description="Tasty", Price=1.30M, Quantity=1},
            new Product{Id=6, ProductName="Pears", Description="Fruity", Price=.70M, Quantity=101},
            new Product{Id=7, ProductName="Pinapple", Description="Nom Nom", Price=.50M, Quantity=73},
        };
    }
}</pre>
<p>We can now start implementing our API for exposing OData. I’m not going to go too much into the conventions for using the Web API as its been covered a lot elsewhere and following the example should be fairly self explanatory.</p>
<p>Create an empty controller called Products. Change the controller to inherit from ApiController rather than Controller, you will need a reference to System.Web.Http for the ApiController class to be found. We then just need a single action method that returns an IQueryable list of our Products. Which should look like this…</p><pre class="brush: csharp; gutter: false; toolbar: false;">using System;
using System.Collections.Generic;
using System.Linq;
using System.Web;
using System.Web.Http;
using System.Web.Mvc;
using ODataExample.Models;
using ODataExample.Models.Repositories;

namespace ODataExample.Controllers
{
    public class ProductsController : ApiController
    {
        public IQueryable&lt;product&gt; Get()
        {
            var productRepository = new ProductRepository();
            return productRepository.Products.AsQueryable();
        }
    }
}</pre>
<p>Assuming you've used the same names as above you should then be able to run the project and navigate to <strong>api/Products</strong> to see a list of the products. </p>
<p>OK so that's pretty neat now check out what happens if you navigate to <strong>api/Products?$top=3&amp;$orderby=Price</strong> pretty cool huh? We now have OData support for the following parameters top, skip, orderby and filter. The other OData parameters are not currently supported but I would guess they will come in a future version. The reason you get this OData functionality is because our controllers get method is returning IQueryable, if we had returned IEnumerable or IList then we would not have got the OData support. </p>
<p>This is a great way to implement things like paging in your APIs without having to code it in your repository. For example the following URL would get page 2 with 3 items per page without any extra code in your repository or controller <strong>api/Products?$skip=3&amp;$top=3&amp;$orderby=id</strong></p>