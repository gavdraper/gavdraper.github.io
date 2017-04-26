---
layout: post
title: ASP.NET 5 Multiple Projects and Global.json
date: '2015-05-20 09:28:31'
tags:
- aspnet5
- dnx
---

This post is part of a series of posts on ASP.NET 5 the series index can be found here [ASP.NET 5 Up and Running](http://gavindraper.com/2015/05/20/asp-net-5-up-and-running-series/).

If you've followed my previous 2 blog posts on ASP.NET 5 you'll now have a single project with some MVC bit's in it and an xUnit.net unit test. This is probably a good point to think about creating a separate project for your unit tests. Up until now we've just had a single Project.json and no notion of a solution. In ASP.NET 5 the .sln files of the past are gone and have been replaced with a JSON file called Global.json. 

In order to split these out let's first create a directory structure suitable for our projects. If you've followed along the previous posts create the following directory structure inside your project route.

```
Src
	Web
Test
	Web.Tests
```

Then time to shuffle some files around...

1. Move the contents of your current Tests folder into Test/Web.Tests and delete the existing Tests folder.
2. Move everything else into Src/Web.

Your directory listing should then look like this... 	

```
Src
	Web
		Controllers
			HomeController.cs
		Views
			Home
				index.cshtml
		project.json
		Startup.cs
Test
	Web.Tests
		HomeControllerTests.cs
```

We then need to create a project.json for our test project. First remove the xUnit dependencies and command from the Web project.json as it's no longer needed there. Then create a new project.json in Tests/Web.Tests that looks like this...

{% highlight javascript %}
{
	"commands": {
		"test" : "xunit.runner.aspnet"
	},
	"dependencies": {
		"xunit.runner.aspnet": "2.0.0-aspnet-beta4-*",
		"Microsoft.AspNet.Mvc": "6.0.0-beta4-*"
	},
	"frameworks": {
		"dnx451": {}
	}
}
{% endhighlight %}
In your shell navigate to the root directory of this solution and run "DNU restore", if you look at the output you'll see it only restored references for Src/Web/Project.json, this is because it looks in Src and the current directory for projects by default but won't look anywhere else. To fix this we need a global.json file that specified where to look for our projects. This global.json goes in the root of the solution and looks like this...

{% highlight javascript %}
{
	"projects" : ["Src","Test"]
}
{% endhighlight %}

If you then run "DNU Restore" at the solution root again you'll see it finds both project.json files and restores the references. If you're using VS Code then reload the window to get it to see the new global.json changes by pressing Cmd+Shift+P then running "Reload Window".

If you then run the Kestrel command for the Web project as we have done before (If doing this from shell you need to be in the web directory) you'll see the website runs fine as it always has. If you however try to run the Test command in the Web.Tests project you'll see it fail with an error saying it can't find the namespace MinimalMvc.Controllers, this is because our Test project does not reference our Web project. To fix this open the test projects project.json and add a new dependency to MinimalMvc like this...

{% highlight javascript %}
{
	"commands": {
		"test" : "xunit.runner.aspnet"
	},
	"dependencies": {
		"Web" : "",
		"xunit.runner.aspnet": "2.0.0-aspnet-beta4-*",
		"Microsoft.AspNet.Mvc": "6.0.0-beta4-*"
	},
	"frameworks": {
		"dnx451": {}
	}
}
{% endhighlight %}

It will resolve the web dependency by looking at the folder system and finding a folder called Web under source. If you now run the Test command again you will see it resolved our 1 test and it passed.

Based on what we've done above you could add many more projects to the solution should you need to. For example it would be trivial to create a new project for any models then reference that from the web project.