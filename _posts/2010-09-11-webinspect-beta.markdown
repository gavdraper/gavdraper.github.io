---
layout: post
title: WebInspect Beta
date: '2010-09-11 14:44:40'
---

I've just uploaded a very early version of a new web app I've been working on. 

<strong>So what is WebInspect?</strong>

Well let me start by saying how it came about...

I currently have a number of small low traffic ASP.Net sites that might get 10 hits a day. Due to the low traffic nature of the sites and the fact the IIS app pools seem to recycle after a period of inactivity (regardless of how long a duration you enter in to IIS) users normally got a slow response the first time they hit the site whilst IIS does its thing to start the site up.

I looked around and couldn't find a good solution to stop IIS from doing this automatic recycle. In my opinion a site taking 6-10 seconds to spin up every time a user used after a period of inactivity doesn't seem very acceptable, So I spent an hour or so and hacked together a windows service that read in a list of URLs from a .config XML file and then did a WebRequest on each of them every 5 minutes. I let this run for a week and it seemed to solve all my problems.

It wasn't long before I decided I wanted an interface to add and remove sites from this service and configure custom intervals. Ok another 4 or so hours and I'd turned the XML files in to a small database and got a very basic ASP.Net MVC front end on it for adding and removing sites. I also used the ASP.Net Membership classes and tables to get a quick security model in place where users can sign up and add their own sites to be included in the windows service.

A day or two later all seemed to be holding up ok and I thought all these WebRequests are good but it seems a shame not to report back the results e.g response times and fail counts so you can see when a site goes down and how responsive it is at different times of the day. Another 4 or so hours and I had the windows service logging each request to the database indicating if it succeeded and how long the request took. I used the Google Visualisations API to put some funky looking charts in so you can see your responsiveness over a period of time.

<strong>In A Nutshell</strong>

WebInspect can be used to monitor, prevent app pool recycles and send automatic notifications for each of your websites.

For anyone who has used Pingdom its like a cut down free version of that.

<strong>Results</strong>

The graph below shows how using WebInspect has cut my response times massively.

<img src="/content/images/WPImport/2010/09/WebInspectReport_thumb.png" border="0" alt="WebInspectReport" width="353" height="195" />

The first 6 requests you see had a 20 min interval which is what I initially set my sites up to have. I then ran some reports on the data and saw an average response time of 6000ms, clearly something's not right and it looks like the app pool was still recycling between requests.

I then changed the interval time to 15 minutes and you can see I've stopped the app pool from automatically recycling.

<strong>What's Next</strong>

<strong> </strong>
<ul>
	<li> Ability to subscribe to automatic email notifications to tell you when a sites goes down and when it comes back up.</li>
	<li> Work on the Windows Service to make better use of threads. Currently the windows service will spin up a maximum of 5 threads clearly when your monitoring hundreds of sites this wont be enough as there will be a lot of unused CPU time whilst you wait for sites to respond or time out.</li>
	<li> Lots of UI CSS/HTML work as minimal time has been spent on this so far.</li>
</ul>
I'll post some of the code up in coming weeks so you can see how it works.