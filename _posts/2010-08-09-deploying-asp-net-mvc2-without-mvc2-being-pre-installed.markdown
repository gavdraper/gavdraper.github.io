---
layout: post
title: Deploying ASP.Net MVC2 Without MVC2 Being Pre Installed
date: '2010-08-09 19:54:01'
---

With the speed that all these new frameworks are being released at it is becoming increasingly hard to constantly get them installed on production servers.

I recently needed to deploy an ASP.Net MVC2 site to a server that did not have MVC2 installed. Instead of putting in requests to get this installed which take time I just deployed the required DLLs and it works fine.

In order for this to work the server must already be running .Net 3.5.

If your running .Net 3.5 SP1 then all you need to do is add the following DLLs to your projects BIN directory
<pre>   System.Web.MVC</pre>
If you don't yet have SP1 then you need to also copy these DLLs into your BIN directory
<pre>   System.Web.Routing
   System.Web.Abstractions</pre>
Once this is done your MVC2 site will work as if MVC2 was installed on the server.