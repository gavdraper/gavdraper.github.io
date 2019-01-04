---
layout: post
title: Profiler Is Dead, Long Live Extended Events
date: '2019-01-03 09:34:01'
tags: tsql maintenance
---
SQL Server profiler has been deprecated since about 2012 and has had no new features since then, Extended Events on the other hand is getting new events and fields added all the time. And Yet... And yet for the most part people are still using profiler and ignoring Extended Events. In this post I'm going to go over a couple of common profiler use cases and show how it's actually easier to use in Extended Events once you've got your head round it. 

I'm not going to go into detail on extended events, if you want more then Jonathan Kehayias has some great posts on it here https://www.sqlskills.com/blogs/jonathan/category/extended-events/.

Lets take 3 common profiles that I used to use before switching to extended events

1) Show RPC Completed queries from user X
2) Show RPC Completed queries over x seconds
3) Show queries with memory grants > 100MB
4) Show deadlock graphs

Run the following script to create Extended Event sessions for the above 3 profile types.

{% highlight sql %}
Extended event SQL goes here
{% endhighlight %}

All of the above can be done through the extended event wizard in SSMS but I like to save these scripts and edit them when I need to create new Extended Event sessions to save time.

Once you've run the script you'll see the 3 Extended Events in SSMS...

INSERT PICTURE HERE

At this point none of these are running, to start one right click and select Start. You can then right click and "Watch Live Data" to see the events in real time.

INSERT PICTURE HERE

These are all currently scripts to use a 1MB ring buffer and wont persist the data, you can use other storage mechanisms to keep the data if needed. TODO CHECK THIS