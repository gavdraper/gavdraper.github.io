---
layout: post
title: C#7 New Features
date: '2010-08-04 19:05:38'
---

I'm going to walk through an example that we can build up and improve with a number of the new C# 7 features. 

Lets say we have a method that takes an object and tries to convert it to a number. There are two ways it does this

* If it's an int then the object is the number
* If it's a string then it will try to conert it using int.TryParse.
* Else return null.

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

With C# 7 we can use the IS clause the test for shape and assign it to a variable for example

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

You may also noticed another C# 7 feature in the code above. That is the out parameter wasnt declared before it was used, In C#7 it doesnt need to be. 

Given the new featurs our original method can now look like this

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

From here we can look at the next new feature, Constructor expression bodies. We've had expression bodies in C#6 for readonly properties and methods but we've now got them for read/write properties, constructors and deconstructors. With that the constructor in the above class can be rewriten as...

{% highlight csharp %}
public BoxOfRandomStuff(object o) => obj = 0;
{% endhighlight %}