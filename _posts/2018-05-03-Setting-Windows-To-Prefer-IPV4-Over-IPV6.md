---
layout: post
title: SQL Server Dedicated Admin Connection and Why You NEED It!
date: '2018-05-03 08:17:12'
---

This post is a bit different from my usual SQL Serer posts, I recently hit a problem where a legacy piece of software would not work on IPV6 and wanted to document it here so I have the solution to hand next time...

I was recently an old server and one of the systems I had to move was OpenFire 3.7.1 a Jabber Server. This is quite an old version that for reasons could not be updated as part of the move. Once installed on the new server OpenFire wouldnt accept any connections, after a lot of digging I found some hints that maybe older versions of OpenFire do not work with IPV6. When looking for ways to set a Windows machine to prefer IPV4 rather than IPV6 the common opinion was you can't do this unless you remove IPV6 from the adapter.

However after some experimentation I found that using netsh you can actually set the order of preference to make Windows prefer IPV4.

From an admin console run this to get a list of the currently set preferences...

{% highlight bash %}
netsh int ipv6 show prefixpolicies
{% endhighlight %}

Take a screenshot or note down the results as we're going to make a change to this ordering and you may want to at some point move back to how it was before and make IPV6 the default again. In order to prefer IPV4 we need to give ::ffff:0:0/96  the highest priority, we can do this by running this...

{% highlight bash %}
netsh int ipv6 set prefixpolicy ::ffff:0:0/96 51
{% endhighlight %}

If you then ping this machine you should see it now resolves to an IPV4 address rather than the previous IPV6 address. If you ever want to undo this change run the following...

{% highlight bash %}
netsh int ipv6 set prefixpolicy ::ffff:0:0/96 4
{% endhighlight %}