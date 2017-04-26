---
layout: post
title: Calling a WCF service from a WP7 Silverlight App
date: '2011-04-07 06:20:22'
---

First right click your WP7 applications project file and select add service reference, add a reference to the WCF service you want to access. In this case lets say you gave the service a namespace of WCFServices and the service is called MyService.

Silverlight accesses services asynchronously so when you call the service you need to give it a method to call once its completed, for example lets say your WCF service has a method called HelloWorld that takes a string as a parameter and returns a string you would call it like this...
<pre class="brush: csharp; toolbar: false;">public void RunWCFHelloWorld()
{
    string InputParam="Hello";
    var wcfService = new WCFServices.MyService();
    try
    {
       wcfService.HelloWorldCompleted += HelloWorldCompleted;
       wcfService.HelloWorldAsync(InputParam);
       wcfService.Close();
    }
    catch
    {
       wcfService.Abort();
    }
}

void HelloWorldCompleted(object sender, HelloWorldCompletedEventArgs e)
{
    string result = e.Result;
    MessageBox.Show(result);
}</pre>
When you call the RunWCFHelloWorld method it will call the HelloWorld method on the WCF service, when this method has returned the result will be passed in to HelloWorldCompleted in the EventArgs.
