---
layout: post
title: Debugging A .Net Windows Service
date: '2011-03-01 09:30:18'
---

<p>
Windows services can be a pain to debug, you can by adding a couple of simple changes to program.cs make the service act different when running in debug mode allowing you to debug like you would any other application. 
</p>
<p>
When you create a windows service in Visual Studio you should have in your program.cs a Main method that looks like this 
</p>
<pre class="brush: csharp; toolbar: false;">
        static void Main()
        {
            ServiceBase[] ServicesToRun;
            ServicesToRun = new ServiceBase[] 
			{ 
				new Service1() 
			};
            ServiceBase.Run(ServicesToRun);
        }
</pre>
<p>
To allow debugging and the use of things like breakpoints change this method to look like this 
</p>
<pre class="brush: csharp; toolbar: false;">
        static void Main()
        {
#if !(DEBUG)
			ServiceBase[] ServicesToRun;
			ServicesToRun = new ServiceBase[] 
			{ 
				new Service1() 
			};
			ServiceBase.Run(ServicesToRun);
#else
            Service1 service = new Service1();
            service.DebugRun(new string[] { });
            System.Threading.Thread.Sleep(System.Threading.Timeout.Infinite);
#endif
        }
</pre>
<p>
The DebugRun method is a method you need to create in your service class that will just call OnStart as OnStart can not be called here directly due to its protection level. This method looks like this 
</p>
<pre class="brush: csharp; toolbar: false;">
        public void DebugRun(string[] args)
        {
            OnStart(args);
        }
</pre>
<p>
You can now put a break point in your OnStart method and debug as you would with a Console or WinForms app. Because of the Thread.Sleep(System.Threading.Timeout.Infinite) line we added to the Main method the service will not end until you close the Debugger when you are running it from Visual Studio. It will perform as normal when installed as a service. 
</p>