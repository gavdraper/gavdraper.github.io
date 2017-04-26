---
layout: post
title: ASP.Net MVC Debugging "Internal Server Error 500"
date: '2013-06-22 08:49:22'
---

<p>I was recently seeing “Internal Server Error 500” when trying to make an ajax call from an ASP.Net MVC view back to the controller. No breakpoints or errors were being thrown in the application.</p> <p>I ended up getting the error information by adding the following event to my Global.asax.cs</p><pre class="brush: csharp; toolbar: false;">protected void Application_EndRequest()
{
}
</pre>
<p>You can then put a break point on the first curly brace and put a watch on “this.Context.AllErrors” allowing you to F5 over each request until you come to one that has an error in the AllErrors property. In my case it was a Json deserialization error which gave me all the information I needed to fix the problem.</p>