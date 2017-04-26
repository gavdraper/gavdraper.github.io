---
layout: post
title: Implementing Infinite Scroll With ASP.Net MVC 4
date: '2012-05-10 06:43:29'
---

<p>There are a lot of ways you could implement this but in this case I will be using the following…</p> <ul> <li>ASP.Net MVC 4  <li>ASP.Net MVC 4 Web API (As OData Source)  <li>jQuery Template plugin</li></ul> <p>The complete project I’m going to walk through can be found on my <a href="https://github.com/gavdraper/MVC-WebApi-Infinite-Scroll">github</a> page.</p> <p>The goal of infinite scroll is to automagically load the next page of data asynchronously when the user scrolls to the bottom of the current page, at this point the next page is appended to the previous page. There are lots of ways you could implement it but ultimately they all boil down to….</p> <ol> <li>Add a JavaScript event to monitor for the user scrolling to the bottom of the page  <li>Use Ajax to fetch the next page from the server  <li>Use JavaScript to render the servers response to the bottom of the page.</li></ol> <p>In this example we are going to create an OData endpoint using the Web API that we can then query from JavaScript to get the relevant page data in a JSON format, we can then use the jQuery Templates plugin to render this data to the page.</p> <h3>Walkthrough</h3> <p>To start with create a new empty ASP.Net MVC 4 project using the Razor view engine.</p> <p>Create a controller called HomeController and just leave the default Index method there as is. Create a new folder in views called Home and add a view to it called Index.</p> <p>Add another controller called FruitController, this will be our OData endpoint using the Web API. To do this we need to change the FruitController class to inherit from ApiController and then add a method that returns an IQueryable list of objects for our page. In this example I’ve bundled the model and dummy data all into the controller just to keep it simple, in the real world you would want to separate this out. Below is what this controller should look like along with some dummy data….</p><pre class="brush: csharp; gutter: false; toolbar: false;">public class FruitApiController : ApiController
{

    public class Fruit
    {
        public int Id { get; set; }
        public string FruitName { get; set; }
        public decimal Price { get; set; }
    }

    public IQueryable&lt;Fruit&gt; Get()
    {
        var Fruits = new List&lt;Fruit&gt;
        {
            new Fruit{Id=1, FruitName="Avocado", Price=.20M},
            new Fruit{Id=2, FruitName="Guava", Price=.20M},
            new Fruit{Id=3, FruitName="Cherry Guavas", Price=.20M},
            new Fruit{Id=4, FruitName="Lychee", Price=.20M},
            new Fruit{Id=5, FruitName="Apples", Price=1.30M},
            new Fruit{Id=6, FruitName="Pears", Price=.70M},
            new Fruit{Id=7, FruitName="Pinapple", Price=.50M,},
            new Fruit{Id=8, FruitName="Cantaloupe", Price=.20M},
            new Fruit{Id=9, FruitName="Casaba", Price=.20M},
            new Fruit{Id=10, FruitName="Crenshaw", Price=.20M},
            new Fruit{Id=11, FruitName="Galia", Price=.20M},
            new Fruit{Id=12, FruitName="Honeydew", Price=.20M},
            new Fruit{Id=13, FruitName="Persian", Price=.20M},
            new Fruit{Id=14, FruitName="Santa Claus", Price=.20M},
            new Fruit{Id=15, FruitName="Sharlyn", Price=.20M},
            new Fruit{Id=16, FruitName="Watermelon", Price=.20M},
            new Fruit{Id=17, FruitName="Blackberry", Price=.20M},
            new Fruit{Id=18, FruitName="Raspberry", Price=.20M},
            new Fruit{Id=19, FruitName="Mulberry", Price=.20M},
            new Fruit{Id=20, FruitName="Strawberry", Price=.20M},
            new Fruit{Id=21, FruitName="Cranberry", Price=.20M},
            new Fruit{Id=22, FruitName="Blueberry", Price=.20M},
            new Fruit{Id=23, FruitName="Jostaberry", Price=.20M},
            new Fruit{Id=24, FruitName="Gooseberry", Price=.20M},
            new Fruit{Id=25, FruitName="Elderberry", Price=.20M},
            new Fruit{Id=26, FruitName="Currant", Price=.20M},
            new Fruit{Id=27, FruitName="Grapes", Price=.20M},
            new Fruit{Id=28, FruitName="Kiwi Fruit", Price=.20M},
            new Fruit{Id=29, FruitName="Papaya", Price=.20M},
            new Fruit{Id=30, FruitName="Mango", Price=.20M},
            new Fruit{Id=31, FruitName="Figs", Price=.20M},
            new Fruit{Id=32, FruitName="Dates", Price=.20M},
            new Fruit{Id=33, FruitName="Olives", Price=.20M},
            new Fruit{Id=34, FruitName="Jujubes", Price=.20M},
            new Fruit{Id=35, FruitName="Pomegranates", Price=.20M},
            new Fruit{Id=36, FruitName="Lemons", Price=.20M},
            new Fruit{Id=37, FruitName="Limes", Price=.20M},
            new Fruit{Id=38, FruitName="Key Limes", Price=.20M},
            new Fruit{Id=39, FruitName="Mandarin", Price=.20M},
            new Fruit{Id=40, FruitName="Orange", Price=.20M},
            new Fruit{Id=41, FruitName="Sweet Lime", Price=.20M},
            new Fruit{Id=42, FruitName="Tangerine", Price=.20M},
            new Fruit{Id=43, FruitName="Clementines", Price=.20M},
            new Fruit{Id=44, FruitName="Grapefruit", Price=.20M},
            new Fruit{Id=45, FruitName="Satsumas", Price=.20M},
            new Fruit{Id=46, FruitName="Tangelos", Price=.20M},
            new Fruit{Id=47, FruitName="Uglis", Price=.20M},
            new Fruit{Id=48, FruitName="Pommelo", Price=.20M},
            new Fruit{Id=49, FruitName="Quinces", Price=.20M},
            new Fruit{Id=50, FruitName="Prickly Pear", Price=.20M},
            new Fruit{Id=51, FruitName="Kumquats", Price=.20M},
            new Fruit{Id=52, FruitName="Minneolas", Price=.20M},
        };
        return Fruits.AsQueryable&lt;Fruit&gt;();
    }

}
</pre>
<p>If you want to know more about OData endpoints in Web API see my blog post on <a href="http://www.gavindraper.co.uk/2012/04/28/exposing-odata-endpoints-using-the-asp-net-mvc-web-api/">Exposing OData Endpoints using the ASP.Net MVC Web API</a>.</p>
<p>So we now have an OData endpoint we can query from JavaScript to get the data we need for each page. We then need a template for how the data should be displayed, in this case I am going to use the jQuery Template plugin to render the template. Create a new partial view in the Shared folder called FruitPageTemplate and put the following code in to it….</p><pre class="brush: csharp; gutter: false; toolbar: false;">&lt;script id="fruitTemplate" type="text/x-jQuery-tmpl"&gt;
    &lt;li style="font-size: 18pt;"&gt;
        ${Id} - ${FruitName} : ${Price}
    &lt;/li&gt;
&lt;/script&gt;
</pre>
<p>This will later allow us to bind our JSON Fruit list returned from the server to the template.</p>
<p>Now we need to code up the JavaScript that will do the fetching and binding of the data. Create a new JavaScript file in the scripts directory called infinite-scroller.js and add the following code to it…</p><pre class="brush: csharp; gutter: false; toolbar: false;">//Constructor params are all needed except loadingContainer and sortOrder
function InfiniteScroll(itemsPerPage, oDataUrl, templateId, containerId, loadingContainer, sortOrder) {
    var currentPage = 1;
    var allPagesLoaded = false;
    var loadingPage = false;

    var readyForNextPage = function () {
        var first = $(window).scrollTop();
        var second = $(document).height() - $(window).height();
        var atBottom = first &gt; second ? first - second : second - first;
        return (atBottom &lt; 3 &amp;&amp; !allPagesLoaded &amp;&amp; !loadingPage);

    }

    var renderPageToTemplate = function (data) {
        if (data == "") {
            $("#" + loadingContainer).hide();
            allPagesLoaded = true;
            loadingPage = false;
        }
        else {
            $("#" + templateId).tmpl(eval(data)).appendTo("#" + containerId);
            currentPage++;
            loadingPage = false;
            loadNextPage();
        }
    }

    var loadNextPage = function () {
        if (readyForNextPage()) {
            //Prevents the page being requested multiple times whilst it is still loading
            loadingPage = true;
            if (loadingContainer)
                $("#" + loadingContainer).show();
            //Build the OData URL
            var url = oDataUrl +
                '?$skip=' + (itemsPerPage * (currentPage - 1)).toString() +
                '&amp;$top=' + itemsPerPage.toString();
            if (sortOrder)
                url += '&amp;$orderby=' + sortOrder;
            //Get Page From Server
            $.get(url, null, function (data) { renderPageToTemplate(data); });
        }
        else {
            if (loadingContainer)
                $("#" + loadingContainer).hide();
        }
    }

    $(window).scroll(loadNextPage);
    loadNextPage();
}
</pre>
<p>That's pretty much all the code we need, all that's left to do is reference our scripts in the Index view and call InfiniateScroll’s constructor with the relevant parameters.</p>
<p>Open up your Index.cshtml in the Views/Home folder and make it look like this</p><pre class="brush: csharp; gutter: false; toolbar: false;">@{
    ViewBag.Title = "Index";
}
@Html.Partial("/Views/Shared/FruitPageTemplate.cshtml")
&lt;h2&gt;Fruits&lt;/h2&gt;
&lt;div id="containerDiv"&gt;&lt;/div&gt;
&lt;div id="loadingContainer" style="display: none"&gt;LOADING.....&lt;/div&gt;
&lt;script src="http://ajax.aspnetcdn.com/ajax/jquery.templates/beta1/jquery.tmpl.min.js"&gt;&lt;/script&gt;
&lt;script type="text/javascript" src="~/Scripts/infinite-scroller.js"&gt;&lt;/script&gt;
&lt;script type="text/javascript" charset="utf-8"&gt;
    $(function () {
        var infiniteScroll = new InfiniteScroll(10, "api/FruitApi", "fruitTemplate", "containerDiv", "loadingContainer","FruitName");
    });
&lt;/script&gt;
</pre>
<p>If you now run the project you will see infinite scroll in action. Running locally it might be a bit quick to really see what is happening so for the examples sake you might want to put a Thread.Sleep in the FruitController Get method so you can see the pages coming in a bit slower.</p>
<p>Disclaimer : this is a naive example with just enough code to get infinite scroll working, it should however give you a good starting point.</p>