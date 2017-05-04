---
layout: post
title: C#7 Show Me The New Stuff!
date: '2017-05-05 08:05:38'
---

I'm going to walk through an example that we can build up and improve with a number of the new C# 7 features. 

Lets say we have a method that takes an object and tries to convert it to a number. There are two ways it does this

* If it's an int then the object is the number.
* If it's a string then it will try to convert it using int.TryParse.
* If it's anything else then  return null.

If we were writing this in C#6 it might look like this

{% highlight csharp %}
public int? FindNumber(object o)
{
    int? result = null;
    if (o is int)
        result = (int)o;
    if (o is string)
    {
        int number;
        if (int.TryParse(o.ToString(), out number))
            result = number;
        }
    return result;
}
{% endhighlight %}

### Pattern Matching & Output Parameters ###

With C# 7 we can use the is statement to test for shape and assign it to a variable for example

{% highlight csharp %}
if(o is int i)
    result = i;
{% endhighlight %}

In the above example if o in an int it will be casted to the local variable i which is created at that point.

We can then take that further by putting an or statement in for the string check

{% highlight csharp %}
if(o is int i || (o is string s && int.TryParse(s,out i)))
{
    result = i;
}
{% endhighlight %}

You may also notice another C# 7 feature in the code above. That is the out parameter wasn't declared before it was used, In C#7 it doesn't need to be. 

Given the new features our original method can now look like this

{% highlight csharp %}
public int? FindNumber(object o)
{
    if(o is int i || (o is string s && int.TryParse(s,out i)))
        return i;
    return null;
}
{% endhighlight %}

For examples sake let's package this method up into a class that we'll throw some other random stuff into and lets assume the constructor takes the object we're operating on.

{% highlight csharp %}
public class BoxOfRandomStuff
{
    private object obj;

    public BoxOfRandomStuff(object o)
    {
        obj = o;
    }

    public int? FindNumberPatternMatching()
    {
        if (obj is int i || (obj is string s && int.TryParse(s, out i)))
            return i;
        return null;
    }
}
{% endhighlight %}

From here we can look at the next new feature, Constructor expression bodies. We've had expression bodies in C#6 for readonly properties and methods but we've now got them for read/write properties, constructors and deconstructors. With that the constructor in the above class can be rewritten as...

{% highlight csharp %}
public BoxOfRandomStuff(object o) => obj = 0;
{% endhighlight %}

### Tuples ###

Lets say we want a method in our new class that returns the current time in 3 parts hours, minutes and seconds. We could create a class to represent this but in this case we only want to use it in one place and a class may be over kill and that would ruin the example. First lets see how we'd do this in C#6

{% highlight csharp %}
public void GetCurrentTime(out int hours, out int minutes, out int seconds)
{
    var time = DateTime.Now;
    hours = time.Hour;
    minutes = time.Minute;
    seconds = time.Second;
}
{% endhighlight %}

Which we would call like...

{% highlight csharp %}
int hour;
int minute;
int second;
test.GetCurrentTime(out hour, out minute, out second);
{% endhighlight %}

Now lets take the next step and remove the out parameter declarations as they're not needed in C#7

{% highlight csharp %}
test.GetCurrentTime(out hour, out minute, out second);
{% endhighlight %}

Next lets look at the new Tuple syntax. For this you may need to install the System.Tuple nuget package from Microsoft.

{% highlight csharp %}
public (int hours, int minutes, int seconds) GetCurrentTime()
{
    var time = DateTime.Now;
    return (time.Hour, time.Minute, time.Second);
}
{% endhighlight %}

Which can be called like this

{% highlight csharp %}
var timeTuple = test.GetCurrentTime();
Console.WriteLine($"{timeTuple.hours}:{timeTuple.minutes}:{timeTuple.seconds}");
{% endhighlight %}

In this case I named the output in the return type of the method but I could have just done (int,int,int) and then they could be referenced by their position e.g Item1, Item2, etc. 

I find this much cleaner than out parameters and although I rarely use them anyway I can't see myself ever picking them over a named Tuple now.

As a finishing note everything above is just to demo the new features and is in no way best practice. 