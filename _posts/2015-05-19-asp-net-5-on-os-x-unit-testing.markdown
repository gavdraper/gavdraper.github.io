---
layout: post
title: ASP.NET 5 Unit Testing With xUnit.net
date: '2015-05-19 10:08:30'
---

This post is part of a series on ASP.NET 5, the index for this can be found here [ASP.NET 5 Up and Running](https://gavindraper.com/2015/05/20/asp-net-5-up-and-running-series/). We're continuing from [ASP.NET 5 on OS X](https://gavindraper.com/2015/05/13/asp-net-5-vs-code-and-osx-getting-started/). As with the previous post everything here will also work on Windows with or without Visual Studio. I'm using OS X as my focus as for me one of the best new features of ASP.NET 5 is being able to develop/deploy cross platform without the need for Visual Studio, I like having the extra choice.

I'm continuing with the simple MVC project we built in my [previous blog post](https://gavindraper.com/2015/05/13/asp-net-5-vs-code-and-osx-getting-started/) and adding a unit test to it using xUnit.net. This is in no way a guide to unit testing in general or xUnit.net, it's more of a sample to get up and running with xUnit.net in the ASP.NET 5 project system.

First up we need to add xUnit.net to our project dependencies, this should be pretty simple if you followed the previous post, we just need to add the following to our Project.json...

```language-javascript
"xunit.runner.aspnet": "2.0.0-aspnet-beta4-*"
```

Now we need a new command registered in Project.json that will launch the xUnit.net runner...

```language-javascript
"test" : "xunit.runner.aspnet"
```
		
If you've followed along from my previous blog post your Project.json should now look like this

```language-javascript
{
	"commands": {
		"kestrel": "Microsoft.AspNet.Hosting --server Kestrel --server.urls http://localhost:5005",
		"web" : "Microsoft.AspNet.Hosting -- server Microsoft.AspNet.Server.WebListener -- server.urls http://localhost:5005",
		"test" : "xunit.runner.aspnet"
	},
	"dependencies": {
		"Microsoft.AspNet.Mvc": "6.0.0-beta4-*",
		"Kestrel": "1.0.0-beta4-*",
		"Microsoft.AspNet.Server.IIS": "1.0.0-beta4-*",
		"Microsoft.AspNet.Server.WebListener": "1.0.0-beta4-*",
		"xunit.runner.aspnet": "2.0.0-aspnet-beta4-*"
	},
	"frameworks": {
		"dnx451": {}
	}
}
```

As we've added a new dependency we now need to run DNX restore either from Visual Studio Code or shell...

- From shell navigate to project directory then "DNU . restore"
- From Visual Studio Code Cmd+Shift+P then type "restore" and run the "DNU:Restore" command.

Visual Studio Code now needs to be reloaded so intellisense updates to reflect the new packages that have been restored...

- Cmd+Shift+P type "reload" and run the reload window command (As stated in my previous post this step will not be needed in a future version of Visual Studio Code)

At this point you can run the unit tests by either...

- In Visual Studio Code hit Cmd+Shift+P and type "test" and click the "DNX test" command.
- From shell if you navigate to your project directory you can run "DNX . test"

When you run the tests you should see an error saying no tests were found...

![](https://gavindraper.com/content/images/ASPNETXunit/1.png)

 I guess we better add one...

Create a new directory in your project called "Tests" with a new class in there called HomeControllerTests like this...

{% highlight csharp %}
using MinimalMVC.Controllers;
using Microsoft.AspNet.Mvc;
using Xunit;

namespace MinimalMVC.Tests
{
	public class HomeControllerTests
	{
		[Fact]
		public void IndexReturnsView()
		{
			var controller = new HomeController();
			var result = controller.Index() as ViewResult;
			Assert.Equal("Hello",result.ViewData["Hello"]);
		}
	}
}
{% endhighlight %}

This is a simple test that just checks our HomeController index method sets the ViewBag.Hello dynamic property to "Hello". Save this class and run the test command again...

![](https://gavindraper.com/content/images/ASPNETXunit/2.png)

Oops it failed, our Index controller action is setting Hello to "Goodbye" and not "Hello" as the test expects.

**Note** : At the time of writing there is a [bug in Mono](https://bugzilla.xamarin.com/show_bug.cgi?id=28793) that causes the test runner not to quit back to shell after it's finished, just hit Ctrl+C twice to return to shell. Hopefully this will be resolved soon but it is not difficult to work around for now. 

Change the HomeController to set it Hello to "Hello" so our test will pass. Now run the tests again...

![](https://gavindraper.com/content/images/ASPNETXunit/3.png)

You should now get a pass! 

In my next post I'll go over multi project solutions so we can look at putting our tests in separate projects. 

Next : [ASP.NET 5 Multiple Projects With Global.json](https://gavindraper.com/2015/05/20/asp-net-5-multiple-projects-and-global-json/)