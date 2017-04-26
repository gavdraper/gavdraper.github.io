---
layout: post
title: App Domains and Their Uses
date: '2012-01-31 20:35:35'
---

<h3><span style="font-weight: bold;">What is an app domain</span></h3>
An App Domain can be thought of as a kind of lightweight process. So What is a process?

A process is an isolated container where the resources needed to run a program are stored. Each process gets its own virtual address space and no process can access another processes memory. On a 32bit Windows system  each process has access a maximum of 3GB of memory depending on how the application was built, the version of Windows and how much memory Windows is currently using. On a 64bit system this RAM limit is raised significantly and will be limited more by the software than the hardware for example Windows 7 will limit this to 192GB where as Windows Server 2008 will go up to 2TB.

Application domains sit inside the process your .Net application is running in, they can be created and destroyed at runtime and exist to put isolation boundaries around certain areas of your code. Code running in one application domain has no direct access to code running in another although they can still communicate via marshalling. If one application domain throws an unhandled exception then only that domain will go down and any other application domains in the process will carry on as they were.

Even if you haven't heard of application domains before you’ve actually already been using them. When you execute a .Net application a single Process is created by Windows then once the CLR has started a single Application Domain is created within that Process where the required Assemblies are loaded and begin execution. Its at this point (Runtime) where you can create additional Application Domains.

See the below diagram for examples of processes with single/multiple app domains and app domains with single/multiple threads.

<a href="/content/images/WPImport/2012/01/ProcessAppDomainsDiagram.png"><img style="background-image: none; padding-left: 0px; padding-right: 0px; display: inline; padding-top: 0px; border-width: 0px;" title="ProcessAppDomainsDiagram" src="/content/images/WPImport/2012/01/ProcessAppDomainsDiagram_thumb.png" alt="ProcessAppDomainsDiagram" width="587" height="480" border="0" /></a>
<h3><span style="font-weight: bold;">When to use Application Domains</span></h3>
Ok so now we kind of know what an Application Domain is but when would you want to use one? Well any of the scenarios below would be prime candidates
<ul>
	<li>Security – For example if your application runs with a high set of security permissions and you want to use a third party assembly that doesn't need all those permissions you could create an App Domain at run time with a custom permission set to run the third party assembly in.</li>
	<li>Loading Assemblies that you later want to unload - Currently if you load a large assembly in to your main App Domain there is no way to unload it when you are finished with it. However this is possible if you load it in its own App Domain then unload the App Domain when you are done with it.</li>
	<li>Isolation- You can use them to isolate separate blocks of your application. This is what ASP.Net does, it hosts each website in its own App Domain preventing any individual site from affecting any other site even if a site throws an exception the others sites remain unaffected. This gives the ability to unload one site whilst keeping the others running.</li>
</ul>
<h3><span style="font-weight: bold;">Here's One I Made Earlier</span></h3>
In this example I load a third party DLL that contains a class implementing one of my interfaces (in this case IPlugin) in to an App Domain that has ReadOnly file system access, I then later unload the App Domain and the contained assembly. This code sample shows what happens when an App Domain tries to execute code above its privilege level and what happens when you unload an assembly.
<pre class="brush: csharp; gutter: false; toolbar: false;">static void Main(string[] args)
{
    const string thirdPartyClassPath = "ThirdPartyAssembly.UntrustedClass";
    const string thirdPartyAssemblyPath = "C:\\Temp\\ThirdPartyAssembly.dll";
    var applicationBase = Path.GetDirectoryName(Assembly.GetExecutingAssembly().Location);

    Console.WriteLine("Creating App Pool And Permissions");
    //Create a permission Set containing only Execute, Read and PathDiscovery permissions
    var ps = new PermissionSet(PermissionState.None);
    ps.AddPermission(new SecurityPermission(SecurityPermissionFlag.Execution));
    ps.AddPermission(new FileIOPermission(PermissionState.None)
    {
        AllLocalFiles = FileIOPermissionAccess.Read
    });
    ps.AddPermission(new FileIOPermission(PermissionState.None)
    {
        AllLocalFiles = FileIOPermissionAccess.PathDiscovery
    });
    //Create an app domain with the custom permission object
    var domain = AppDomain.CreateDomain(
                                "LowSecurityDomain",
                                null,
                                new AppDomainSetup { ApplicationBase = applicationBase },
                                ps);

    Console.WriteLine("Loading Third Party DLL");
    //Load the third party DLL into the new AppDomain
    var plugin = (IPlugin)domain.CreateInstanceFromAndUnwrap(thirdPartyAssemblyPath, thirdPartyClassPath);

    Console.WriteLine("Attempting To Read A file");
    var canRead = plugin.ReadFile("C:\\Temp\\ReadFile.txt");
    Console.WriteLine(" {0}",canRead);

    Console.WriteLine("Attempting To Write A File");
    var canWrite = plugin.CreateFile("C:\\Temp\\ShouldNeverGetCreated.txt");
    Console.WriteLine(" {0}",canWrite);

    Console.WriteLine("Unloading Domain");
    AppDomain.Unload(domain);
    Console.WriteLine("Domain Unloaded");

    Console.WriteLine("Attempting To Run Method On ThirdParty DLL");
    try
    {
        plugin.ReadFile("filePath");
        Console.WriteLine(" Ran after unload (WILL NEVER HAPPEN");
    }
    catch
    {
        Console.WriteLine(" Failed to run as have been unloaded");
    }
}</pre>
The complete example is available <a href="http://www.gavindraper.co.uk/AppDomains.zip" target="_blank">here</a>.