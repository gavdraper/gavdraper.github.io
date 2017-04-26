---
layout: post
title: Retrieving Accurate CPU Usage In C#
date: '2011-03-01 14:29:53'
---

Anyone who has used the PerformanceCounter object in C# to retrieve % Processor time has probably noticed that this counter is a bit random to say the least. It quite often reports 0% when usage is considerable higher than that.

Most code I've seen to get the current usage looks like this
<pre class="brush:csharp;toolbar: false;">public int GetCpuUsage()
{
var cpuCounter = new PerformanceCounter("Processor", "% Processor Time", "_Total", "MyComputer");
return (int)cpuCounter.NextValue();
}</pre>
If you try this is will often just report 0, the reason being is that the PerformanceCounter object needs 2 values to give an accurate reading. This might lead you to think that inserting cpuCounter.NextValue() before the return line would fix the problem however this is not the case. These counters tend to only be updated about once or twice a second so calling it twice in succession would likely just return the same value.

The method below returns an int representing the accurate % of CPU usage at that time.
<pre class="brush:csharp;toolbar: false;">public int GetCpuUsage()
{
var cpuCounter = new PerformanceCounter("Processor", "% Processor Time", "_Total", "MyComputer");
cpuCounter.NextValue();
System.Threading.Thread.Sleep(1000);
return (int)cpuCounter.NextValue();
}</pre>
As you can see this implementation gets the initial value then waits a second to get the now updated performace counter for CPU. In all situations that I have used this code 1 second has been long enough to get an accurate reading.