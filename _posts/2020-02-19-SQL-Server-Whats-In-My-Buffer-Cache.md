---
layout: post
title: SQL Server, What's In My Buffer Cache?
date: '2019-02-19 05:32:12'
---
When SQL Server reads pages it stores them in an area of memory called the buffer cache, things like memory pressure can then cause items to get removed from the buffer cache. I wrote the below script to check what's in the cache at any given time, it's scoped to the database you are running it in...

{% highlight sql %}
SELECT 
    o.name [Table],
    i.name [Index],
    (COUNT(*) *8)/1024 [SizeMB]
FROM
    sys.allocation_units au 
    INNER JOIN sys.dm_os_buffer_descriptors bd ON au.allocation_unit_id = bd.allocation_unit_id
    INNER JOIN sys.partitions p ON  
        /* allocation_unit types : 
            0=Dropped, 
            1 = In Row, 
            2=LOB, 
            3=Row-Overflow data*/
        (au.[type] IN (1,3) AND au.container_id = p.hobt_id) 
        OR (au.[type] = 2 AND au.container_id = p.partition_id)
    INNER JOIN sys.objects o ON p.object_id = o.object_id
    LEFT JOIN sys.indexes i ON i.object_id = o.object_id AND p.index_id = i.index_id
 WHERE 
    o.type NOT IN ('S','IT') 
GROUP BY
    i.name,
    o.name
{% endhighlight %}

![Backup Status Results]({{site.url}}/content/images/2019-Buffer-Cache/cache-result.PNG)

I find this really useful when looking at things like buffer hit ratios to then get an idea as to what in the buffer cache could be causing memory pressure.