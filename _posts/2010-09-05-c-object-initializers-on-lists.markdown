---
layout: post
title: C# Object Initializers On Lists
date: '2010-09-05 08:35:36'
---

<p>
I've had a few comments on the Object Initializers I posted about in my previous post
namely how to use them for lists. 
</p>
<p>
Well its pretty much the same syntax as for single objects and here it is 
</p>
<pre class="brush: csharp; toolbar: false;">
  Rooms = new List&lt;Room&gt;()
     {
      new Room(){Name="Room 1", Id=1},
      new Room(){Name="Room 2", Id=2},
      new Room(){Name="Room 3", Id=3},
     };</pre>