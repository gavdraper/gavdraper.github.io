---
layout: post
title: Optimizing ASP.Net
date: '2010-10-06 07:07:15'
---

I'm going to run through some short short easy steps that will give you pointers on optimizing your ASP.Net sites for performance. In future I plan to dedicate a post to each point below but for now I'm just going to provide links to external articles where relevant.

<strong>Paging</strong> : If your displaying rows of data on your page returned from a datasource of some sort and the quantity is unknown make sure you are using paging.
This way only x amount of records are returned and the user can then page through the result set to request the next set of records. For an example of custom paging using SQL 2005 and ASP.Net see this <a href="http://weblogs.asp.net/scottgu/archive/2006/01/07/434787.aspx">article</a>.

<strong>Caching : </strong>ASP.Net has a great caching API built in and where it makes sense to you can gain massive benefits by using it. If you are returning generic data
that is not specific to a user and doesn't change regularly then you should be thinking about using the ASP.Net caching API's. See this <a href="http://www.c-sharpcorner.com/uploadfile/vishnuprasad2005/implementingcachinginasp.net11302005072210am/implementingcachinginasp.net.aspx">article</a> for
more information on using the caching API.

<strong>Asynchronous Actions :</strong> Where the user does not need to wait for an action to be performed don't make them. For example if the user clicks a button to send an email but doesn't need any sort of confirmation don't make the user wait for the email to send before you load the page. I like to remove actions like sending email from ASP.Net and put them in to some sort of service you can then use your preferred communication method e.g WCF, Windows Messaging to ask the service to perform the action without having to wait for the result or consuming any time in the ASP.Net engine processing this action.

<strong>Output Page Caching </strong>: If you have a page where the content rarely changes then it may be worth marking the page for output caching. Doing this allows
the page to be cached and will then bypass going through the page lifecycle when it is requested. More information on using output caching can be found <a href="http://www.asp.net/mvc/tutorials/improving-performance-with-output-caching-vb">here</a>.

<strong>Session State </strong>: Did you know session state can be disable or marked as read only at a page or application level. If you don't use session state turn it off, its one less thing for IIS to manage and it can give performance gains by removing this from the loop.

<strong>View State : </strong>As with session state if your not using it turn it off. Also if your encrypting your view state make sure its only being encrypted for pages that need it to be, there is no point spending CPU cycles encrypting data that does not need to be.

<strong>Compression : </strong>This deals with a different kind of performance bottleneck, if your application is struggling due to bandwidth you may want to try using GZip
compression built in to IIS6+. This can drop your bandwidth by over 50% the drawback to this is that it is at the cost of extra CPU cycles its kind of a balancing act. For more information on enabling compression within IIS6 see <a href="http://support.microsoft.com/kb/322603">here</a>, If your using IIS7 then this <a href="http://biasecurities.com/blog/2009/iis7-how-to-quickly-and-easily-optimize-your-website-using-gzip-compression/">page</a> should get you going.