---
layout: post
title: C# Hidden Features
date: '2010-08-10 21:48:52'
---

In the last few releases of C# a lot of new features have made there way in allowing
us to write less code. A lot of these just kind of snuck in to the language and didn't
get much press. I'm going to cover a couple of these new and old but less well known
features below.

<strong>Automatic Properties</strong>

Automatic properties allow you to create simple properties without having to create
the backing field.

For example standard properties look like this
<pre class="brush: csharp; toolbar: false;">private string foo;
public string foo
{
    get{return foo;}
    set{foo = value;}
}</pre>
With the addition of automatic properties the code above can be replaced with the
following
<pre class="brush: csharp; toolbar: false;">public string Foo{get;set;}</pre>
Not only is it less code but you then don't have to worry about accessing the field
when you wanted to access the property or accessing the property when you wanted the
field. Obviously this doesn't work if you want logic or manipulations to be run on
the field before it is set or retrieved, in that case you will have to declare the
field like you always did.

<strong>The ?? Operator</strong>

The ?? operator returns the first non null value. For example
<pre class="brush: csharp; toolbar: false;">string test = null;
string test2 = "Hello world"
Console.WriteLine(test??test2);</pre>
This will write â€œHello Worldâ€ to the console as test is null so it ignores it and
checks the next value. This can be daisy chained so you could have more than 2 values.

<strong>Object Initializers</strong>

Object initializers allow you to initialize the public properties of an object when
you create it e.g
<pre class="brush: csharp; toolbar: false;">public class Foo
{
    public string Name{get;set;}
    public int Code{get;set;}
}

public void CreateFoo
{
    var foo = new Foo();
    foo.Name = "MyFoo";
    foo.Code = 123;
}</pre>
Could become
<pre class="brush: csharp; toolbar: false;">public class Foo
{
    public string Name{get;set;}
    public int Code{get;set;}
}

public void CreateFoo
{
    var foo = new Foo(){Name="MyFoo",Code=123};
}</pre>
This simple example would be better achieved with a constructor, in reality this feature
works well on objects with multiple properties that cant be set by passing their values
into the constructor.

Watch this space for more in my next post...