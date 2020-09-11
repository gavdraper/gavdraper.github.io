---
title: Building Better DotNet WebAPIs
date: 2020-05-27 00:00:00
subtitle: ''
description: "Disocvery and Documentation"
featured_image: '/images/posts/api/switches.jpg'
---

Using a combination of the following self describing, discoverabel APIs can be automatedwith extra functionaility like dashboard for calling the endpoints and even code generation...

* OpenAPI (Formally known as Swagger)
* Swashbuckle/NSwag
* Controller Atrributes to describe API methods

To follow along create a new WebAPI project

    dotnet new webapi -o MyAwesomeApi

OpenAPI formally known as Swagger is at its simplest a speification for a JSON file to describe your API other technologies can then add tolling on the end of this by interpretting that JSON file. Both Swashbuckle and NSwag provide middleware for adding a web based dashboard to your API for browing and calling methods on the API. They work great for both development and providing consumers with a UI driven API for testing how endpoints work, NSwag has the added benefit of also using code generation to create clients for some languages.

Lets first expose the Swagger.Json file...

{% highlight powershell %}
dotnet add package Swashbuckle.AspNetCore
{% endhighlight %}

Make the follwing changes to StartUp.cs

{% highlight csharp %}
using Microsoft.OpenApi.Models;
{% endhighlight %}

The in configure services...

{% highlight csharp %}
services.AddSwaggerGen(c => {
    c.SwaggerDoc("v1", new OpenApiInfo { Title = "AwesomeAPI", Version = "v1" });
}); 
{% endhighlight %}

Finally at the top of your configure method...

{% highlight csharp %}
app.UseSwagger();
{% endhighlight %}

You can now run the project and go to the following URL to see the Swagger.Json file that swashbuckle has created for you, depending on the URL of your API it will be something like this

    https://localhost:5001/swagger/v1/swagger.json

As a next step you can turn on the Swagger UI middleware by adding the following to the configure method in Startup...

{% highlight csharp %}
app.UseSwaggerUI(c =>
{
    c.SwaggerEndpoint("/swagger/v1/swagger.json", "My API V1");
});
{% endhighlight %}



TODO 
Multiple Swagger Documents
NSwag
Code Generation
