---
layout: post
title: .Net CLI Cross Platform Awesomeness
date: '2016-02-07 20:12:58'
---

.Net CLI is the new tooling that sits in front of .Net Core, allowing you to create and run projects from the command line on Windows, Linux and OS X.

It's features include

* Launch the compiler
* Create a new project
* Restore Dependancies
* Run custom commands like EF Migrations
* Start a REPL (Read, Evaluate, Print Loop)

Installation
---
The quickest way to get started is to download and run the installer for your platform, the latest built installers can be found here https://github.com/dotnet/cli	

Usage
---
The quickest way to get up and running with this stuff is to create a quick console app. Now that .Net CLI is installed you should be able to run it from your command line so let's walk through creating that new project....

1. Create a new directory for your project and set that as your working directory in your terminal of choice.
	* mkdir myDotNetApp	
	* cd myDotNetApp
2. We can then create a new project from the built in new project template by running....
	* DotNet New
3.	This will have given us the minimum we need for a new console app. Before we can run it we need to restore the dependancies from NuGet, to do that run this command...
	* DotNet Restore
4. Now we have our dependancies we can run the project, let's give it a go...
	* DotNet Run

If all that went well you should see the output of our Console app. 

![Animated Example]({{ site.url }}/content/images/dotnet.gif)

Other Languages
--
As you saw above the .Net CLI tools default to C# for creating new projects, so what about other languages? We can pass DotNet New a language argument to specify which language project we want to create, let's give that a go...

1. DotNet New -l F#
2. DotNet Restore
3. DotNew Run

In 3 commands we create an F# console app and ran it. Notice that instead of Program.cs we now have Program.fs, you'll also see that project.json brings in the F# dependancies this time, where as it didn't need them for the C# project. What's more those same 3 commands will create and run that same app on Windows, Linux and OS X. 

The Future
--
This makes it a very exciting time for .Net given how quickly a new machine can be up and running on .Net, this opens .Net up to a lot of environments where it would have been ruled out due to Cross Platform, Licensing, or time to writting code on a fresh machine. Hopefully we will start to see .Net being used more in Education and at Hackathons. 