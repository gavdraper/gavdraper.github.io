---
layout: post
title: Debugging Issues With Log4Net Not Logging
date: '2013-07-29 15:02:40'
---

<p>When you find log4net is not writing anything to your logs it can be a pain to debug as obviously Log4Net will (and should) never throw an exception.</p> <p>The first thing you want to do when you find you have this problem is to enable tracing in Log4Net. This is done by adding the following key to your app settings in either your app.config or web.config</p><pre class="brush: xml; toolbar: false;">&lt;add key="log4net.Internal.Debug" value="true" /&gt;</pre>
<p>This tells Log4Net to write a trace. Next you need to turn on tracing for the application as without this Log4Net has nothing to write the trace information to. This is done by adding the following to either your app.config or web.config</p><pre class="brush: xml; toolbar: false;">&lt;trace autoflush="true"&gt;
  &lt;listeners&gt;
    &lt;add
      name="textWriterTraceListener"
      type="System.Diagnostics.TextWriterTraceListener"
      initializeData="appTrace.txt" /&gt;
  &lt;/listeners&gt;
&lt;/trace&gt;</pre>
<p>When you then run the application any traces will be written to the applications running directory in a file called appTrace.txt. You can easily specify any path you want as long as the application has permission to write to this path. With any luck after a quick run of your application through code that should write to the log file you should have enough information in the trace file to work out why nothing is getting logged.</p>