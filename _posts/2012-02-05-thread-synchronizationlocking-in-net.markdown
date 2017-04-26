---
layout: post
title: Thread Synchronization/Locking In .Net
date: '2012-02-05 15:57:17'
---

<p>When you are creating multiple threads that need to access a shared resource then you need to implement some form of locking on that resource to prevent race conditions. A race condition can occur when two threads try to access the shared resource at the same time, for example if two threads run this counter = counter+1 at the same time then counter may well only get incremented by 1 as both threads just add 1 to the initial value of counter.</p>  <p>Here is an example of a C# Console App that has these race conditions. When the app finishes the number in firstNumber and secondNumber should be 100 however it will most likely be some random number significantly smaller than that due to the race condition.</p>  <pre class="brush: csharp; gutter: false; toolbar: false;">class Program
{
    static int firstNumber = 0;
    static int secondNumber = 0;

    static void Main(string[] args)
    {
        var threads = new List&lt;Thread&gt;();
        //Create 4 threads
        for(int i = 0;i&lt;4;i++)
            threads.Add(new Thread(AddNumbers));
        foreach(var t in threads) t.Start();
        //wait for all the threads to finish then output the shared resource values
        foreach (var t in threads) t.Join();
        Console.WriteLine(&quot;First Number : {0}\nSecond Number : {1}&quot;,firstNumber,secondNumber);
     }

    static void AddNumbers()
    {
        for (int i = 0; i &lt; 25; i++)
        {
            firstNumber++;
            secondNumber++;
            Thread.Sleep(5);
        }
    }
}</pre>

<p>There are various ways to solve this problem, however in this post I am going to look the most used lock types that .Net provides. It is worth noting at this point that any code samples will be C# 4 and may be slightly different from earlier versions of C#.</p>

<h3><font style="font-weight: bold">Monitor</font></h3>

<p>This is the most common lock used and is actually the lock type that gets used when you use the &quot;Lock&quot; keyword in C#. Monitor locks can be placed directly on the shared resource or on a new object that you can then use to control the access to a collection of shared resources. </p>

<p>In order to make the above sample work using a Monitor lock it is as simple as adding a new lock object to the class and making a couple of changes to our AddNumbers method.</p>

<pre class="brush: csharp; gutter: false; toolbar: false;">static object resourceLock = new object();

static void AddNumbers()
{
    for (int i = 0; i &lt; 25; i++)
    {
        Monitor.Enter(resourceLock);
        try
        {
            firstNumber++;
            secondNumber++;
            Thread.Sleep(5);
        }
        finally
        {
            Monitor.Exit(resourceLock);
        }          
    }
}</pre>

<p>What this does is wait at the Monitor.Enter line until there are no locks open on the resourceLock object, it then obtains a lock on the object, makes its changes and releases the lock on the object so the next thread can begin its lock process.</p>

<p>Its worth noting that the above code can be shortened by using the “Lock” keyword…</p>

<pre class="brush: csharp; gutter: false; toolbar: false;">static void AddNumbers()
{
    for (int i = 0; i &lt; 25; i++)
    {
        lock(resourceLock)
        {
            firstNumber++;
            secondNumber++;
            Thread.Sleep(5);
        }       
    }
}</pre>

<h3><font style="font-weight: bold">Mutex</font></h3>

<p>This type of lock is built in to the Window Kernel and .Net includes a wrapper class for using it. The advantages of using a Mutex lock over a Monitor are </p>

<ul>
  <li>You can specify a timeout. This can be especially useful in preventing deadlocks. </li>

  <li>You can give them a name in their constructor and because they sit in the Kernel that same named lock could be used in multiple processes. This means you could have two applications using the same lock. </li>
</ul>

<p>Because these locks sit in the Windows Kernel there is a bit more overhead in using them as .Net has to call out to unmanaged code to create and remove them. In most cases a monitor lock is sufficient but when its not the Mutex may be the right choice.</p>

<p>Below is an example of avoiding deadlocks in Mutex’s by specifying a maximum wait time when trying to acquire a lock.</p>

<pre class="brush: csharp; gutter: false; toolbar: false;">private static Mutex lockOne = new Mutex();
private static Mutex lockTwo = new Mutex();

static void Main()
{
    lockTwo.WaitOne();
    var threads = new List&lt;Thread&gt;();
    for (var i = 0; i &lt; 10; i++)
    {
        var threadNo = i;
        threads.Add(new Thread(() =&gt;
               {
                   lockOne.WaitOne();
                   Console.WriteLine(&quot;Thread {0} acquired 1st lock&quot;, threadNo);
                   Thread.Sleep(500);
                   if (lockTwo.WaitOne(1000))
                   {
                       //Will never get here as lock is already open
                       lockTwo.ReleaseMutex();
                   }
                   else Console.WriteLine(&quot;Acquiring Lock 2 Timed Out, Not Waiting Any Longer As Possible Deadlock&quot;);
                   lockOne.ReleaseMutex();
               }));
        threads[threads.Count - 1].Start();
    }

    foreach (var t in threads)
        t.Join();
}</pre>

<h3><font style="font-weight: bold">Semaphore</font></h3>

<p>A semaphore allows you to limit the number of threads that can access a shared resource at any one time. Both the Mutex and Monitor locks are opened exclusively to one thread so the Semaphore is the lock to use if you need to allow more than one thread at a time to access a shared resource. A Semaphore can be thought of as limited pool of locks. </p>

<p>See below for an example of creating a Semaphore that allows 3 concurrent threads to access a shared resource.</p>

<pre class="brush: csharp; gutter: false; toolbar: false;">public static void Main()
{
    //Create a semaphore that allows 3 simultaneous threads
    pool = new Semaphore(0, 3);
    //The main thread owns the entire semaphore count so we first need to free it so 
    //our other threads can take a lock from the pool
    pool.Release(3);
    var threads = new List&lt;Thread&gt;();
    for (var i = 1; i &lt;= 10; i++)
    {
        var threadNo = i;
        threads.Add(new Thread(() =&gt;
               {
                   pool.WaitOne();
                   Console.WriteLine(&quot;Thread {0} acquired a lock&quot;, threadNo);
                   Thread.Sleep(2000);
                   pool.Release();
               }));
        threads[threads.Count-1].Start();
    }
    foreach (var t in threads)
        t.Join();        
}</pre>

<p>Like Mutex’s Semaphores can also be named and time out durations can be specified in the WaitOne method.</p>

<p>There are a number of other prime features of the locks mentioned above but this post was just a very general overview. I may do another post on things like Pulsing/Waiting and Data partitioning to show more detailed threading techniques in the future.</p>