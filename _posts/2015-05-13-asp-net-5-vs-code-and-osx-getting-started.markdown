---
layout: post
title: 'ASP.NET 5 On OS X : Getting Started'
date: '2015-05-13 09:40:45'
tags:
- aspnet5
- dnx
- aspnet-mvc
---

This post is part of a series of posts on ASP.NET 5 the series index can be found here [ASP.NET 5 Up and Running](http://gavindraper.com/2015/05/20/asp-net-5-up-and-running-series/).

Over the last year Microsoft have released a lot of tools/frameworks that have made ASPNET cross platform development almost seamless. It's been possible for a long time now to develop and deploy to *nix platofrms using Mono and tools like Xamarin Studio/Mono Develop, I've tried these tools numerous times but always ended up backing away as I've hit various problems. I gave this another go a few weeks back now that DNX and VSCode have been released and was amazed how far this has come.

- The setup process took about 5 minutes and had no issues.
- Intellisense just worked.
- I was pushing code to/from OS X/Windows and it just worked in both places.

I'll breakdown a few of the different tools that have made this possible then walk through the process to getting this all setup.

## DNX ##
ASP.NET5 is built to run on DNX. DNX is the .Net Execution Environment that will bootstrap your application. It's both and SDK and  runtime. This is what will host the correct CLR and start your application. DNX can bootstrap an application using the Full .Net Framework, CoreCLR or Mono.

## CoreCLR ##
CoreCLR is Microsoft's Cross Platform CLR. This is built to be deployed with your application, to make that possible it's been stripped down to a size of about 10mb. A lot of what what in the full CLR has been moved to nuget packages to allow the CoreCLR to be as lightweight as possible. It's important to note the CoreCLR does not contain everything the Full CLR does and if your application uses objects in the Full CLR that are not in Core you will either have to stick with Full or find an alternatvie solution that is available in the CoreCLR. Switching to CoreCLR *will* ensure your project is cross platform as this CLR is built for Windows, Linux, OS X and FreeBSD, where as the full CLR is only built for Windows. At the time of writting the CoreCLR only runs on Windows as it is still in development so for *nix based apps we're still using Mono.

## VS Code ##
VS Code is text editor much like Vim, Sublime and Atom. It comes with Omnisharp Server setup out of the box to give you full intellisense in .Net projects. Omnisharp has plugins for Sublime/Atom too but from what I've used so far VS Code has by far been the easiest to setup and start using. It has Navigation/Debugging features built in for JS, .Net and CSS making this a bit more of a lightweight IDE than a text editor. Oh and it runs on Windows, Linux and OS X. It's early days for VS Code and is missing a few key features that other editors have (I'd love to see some sort of extension system) but for ease of use it's going in a great direction.

## OS X Setup ##
Ok let's walk through getting this up and running on OS X. First we need to get Mono (This will be optional once CoreCLR is available for OS X) and DNX installed into our path. Both of these are installed via [Homebrew](http://brew.sh/ "Homebrew") if you don't have that installed then you'll need to install it by running this command

```language-csharp
ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

After that completes you'll have homebrew on your system now we need to get DNX and Mono installed. Run these commands in your terminal..

```language-csharp
brew tap aspnet/dnx
brew update
brew install dnvm
```

If DNVM is not in your path after the installation then run this

```language-csharp
source dnvm.sh
```

***Note*** at the time of writting DNX is still very much in development, if the above commands fail it may be because the installation procedure has changed, The official [readme](https://github.com/aspnet/Home "readme") on their github repository should detail the new procedure.

We now need to use the newly installed DNVM (.Net Version Manager) to install DNX by running this

```language-csharp
DNVM upgrade
```
 
At this point your machine is ready to run DNX based .Net projects. Head over to [VS Code](https://code.visualstudio.com/ "VS Code") and grad their installer. Once this is installed create a folder to work in, open VS Code,  Click "Open Folder" and choose the folder you just created to set your working directory.

![](http://gavindraper.com/content/images/openFolder.png)


Create a new file and save it as project.json. 

```language-javascript
{
	"dependencies": {
		"Microsoft.AspNet.Mvc": "6.0.0-beta4-*",
		"Kestrel": "1.0.0-beta4-*"
	},
	"commands": {
		"kestrel": "Microsoft.AspNet.Hosting --server Kestrel --server.urls http://localhost:5005"
	},
	"frameworks": {
		"dnx451": {}
	}
}
```

I'll encourage you to type the code out rather than copy and paste as it's awesome to see how good the intellisense is. Notice how as you're tryping string names and versions you're getting intellisense.

Hit Cmd+Shift+P to open the command window then search for and run the "DNU Restore" command. DNU is a .Net utility client that got installed with DNVM and DNX. In this case we're using it to restore the nuget packages we referenced in project.json (MVC and Kestrel). Once the restore has completed we need to reload the window as currently VS Code doesnt automatically pick up as packages are restored, to do this use Cmd+Shift+P again and run the "Reload Window" command.

We now have the most minimal project.json for running an MVC site on OSX. Next up We need to create a Startup.cs with the following code...

```language-csharp
using Microsoft.AspNet.Builder;
using Microsoft.AspNet.Http;

namespace MinimalMVC
{
    public class Startup
    {
        public void Configure(IApplicationBuilder app)
        {
			app.Run(async(c)=>{
				await c.Response.WriteAsync("Woop Woop!");
			});
        }
    }
}
```

If you run "DNX Kestrel" from VS Code (Cmd+Shift+P) you should get a terminal window reading "Started", if you then navigate to http://localhost:5005 you should see our awesome new webpage "Woop Woop!". We've created a site that returns "Woop Woop!" no matter what URL you request inside it. We'll add in MVC in a minute but before we do I want to break down what we've done above....

### Project.json ###
Notice how we have a dependency on Kestrel. Kestrel is an Http Server based on libuv and is the webserver we're using here when running on *nix systems (On Windows we have IIS and WebListener). The dependency is there to reference the Kestrel nuget package from our project and restore it when we run DNU Restore. 

The commands section is where you can add any commands you want DNX to be able to run. These commands can be run by VS Code as we did above by running "DNX Kestrel" from the command box or you can navigate to the project folder in terminal and run "dnx . kestrel" (Replace kestrel with the name of the command). We've seen the command to start the kestrel webserver but there are many more you can implement your own or use commands provided by other nuget packages, for example EF comes with migration commands.

Lastly in project.json we have Frameworks, here we can list all the frameworks out project supports, your project will then be compilied multiple times when you package it up to support each of these frameworks. 

One thing to note here you'll see I've used * on all my version numbers, that's telling it to get the latest version of beta4, for production systems you'll want to nail this down to a full version number so you know what you're running against.

## What about MVC ##
As you've probably noticed we've specififed MVC as a dependency in our project.json file and have restored the MVC nuget package when we ran DNU Restore, now we just need to write it in to our pipeline...

Modify your Startup.cs to look like this...

```language-csharp
using Microsoft.AspNet.Builder;
using Microsoft.Framework.DependencyInjection;

namespace MinimalMVC
{
    public class Startup
    {
        public void Configure(IApplicationBuilder app)
        {
            app.UseMvc(routes =>
            {
                routes.MapRoute(
                    name: "default",
                    template: "{controller}/{action}/{id?}",
                    defaults: new { controller = "Home", action = "Index" }
                );
            });
        }

        public void ConfigureServices(IServiceCollection services)
        {
            services.AddMvc();
        }
    }
}
```

This will add MVC to our pipeline and setup a default route. Now much like in previous versions of ASP.Net we just need to add a controller and a view...

Create a folder called Controllers then inside it create HomeController.cs with the following code...

```language-csharp
using Microsoft.AspNet.Mvc;

namespace MinimalMVC.Controllers
{
    public class HomeController : Controller
    {
        public ActionResult Index()
        {
            ViewBag.Hello = "Goodbye";
            return View();
        }
    }
}
```

Create Views\Home folders with a new file in Views\Home called index.cshtml containing the following code...

```language-markup
 <html>
	<body>
		<h1>Hello</h1>
		<h2>@ViewBag.hello</h2>
	</body>
</html>
```

I wont go into detail on how the controller/view code works as in this case the code is identical to previous versions of MVC. You should now be able to run DNX Kestrel and navigate to http://localhost:5005 or http://localhost:5005/home/index to hit that view. Also notice that if you throw an error in your controller or hit a route that doesnt exist in your browser you get a blank screen rather than an error, by design in ASP.NET 5 you have to opt in to everything you want rather than being given everything and having to remove what you dont need. Want error pages? No problem just add a dependency in project.json to Microsoft.AspNet.Diagnostics then in your Startup.cs add app.UseErrorPage(); above App.UseMvc() in the configure method.

## Cross Platform
Our project.json currently only tarets Kestrel what if we want to run on Windows? No problem just a minor tweak to our project.json to add IIS/WebListener dependencies and a new command to start the WebListener...

Change project.json to look like this...


```language-javascript
{
	"commands": {
		"kestrel": "Microsoft.AspNet.Hosting --server Kestrel --server.urls http://localhost:5005",
		"web" : "Microsoft.AspNet.Hosting -- server Microsoft.AspNet.Server.WebListener -- server.urls http://localhost:5005"
	},
	"dependencies": {
		"Microsoft.AspNet.Mvc": "6.0.0-beta4-*",
		"Kestrel": "1.0.0-beta4-*",
		"Microsoft.AspNet.Server.IIS": "1.0.0-beta4-*",
		"Microsoft.AspNet.Server.WebListener": "1.0.0-beta4-*"
	},
	"frameworks": {
		"dnx451": {}
	}
}
```

Now on OSX you run the website with DNX Kestrel and on Windows you use DNX Web. Pretty cool huh? 

I'll try to cover a lot of the above in more detail in future posts but this should serve as a good overview for getting started.

Next : [ASP.NET 5 On OS X Unit Testing With xUnit.net](http://gavindraper.com/2015/05/19/asp-net-5-on-os-x-unit-testing/) 