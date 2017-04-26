---
layout: post
title: C# Explicit Interface Implementation
date: '2012-06-07 07:20:26'
---

<p>In C# it is possible to write classes that implement an interface but only show the methods of the interface when the object is casted to the type of the interface.</p> <p>For example lets say we have the following code</p><pre class="brush: csharp; gutter: false; toolbar: false;">void Main()
{
	var f = new SuperFoo();
	f.doFoo();
	f.doSuperFoo();	
}

public interface IFoo
{
	void doFoo();
}

public class SuperFoo : IFoo
{
	public void doFoo()
	{
		Console.WriteLine("Foo");
	}
	
	public void doSuperFoo()
	{
		Console.WriteLine("Do Super Foo");
	}
}
</pre>
<p>Lets say we only want to see the doSuperFoo method when we create an instance of SuperFoo. However if a user casts SuperFoo to IFoo obviously doSuperFoo is then lost so in this case we want to fall back to the doFoo method. As you probably know methods defined in interfaces cannot be made private however through use of explicit interface implementation we can still achieve this and here is how... </p><pre class="brush: csharp; gutter: false; toolbar: false;">void Main()
{
	var f = new SuperFoo();
	f.doSuperFoo();	
	var f2 = (IFoo)f;
	f2.doFoo();
}

public interface IFoo
{
	void doFoo();
}

public class SuperFoo : IFoo
{
	void IFoo.doFoo()
	{
		Console.WriteLine("Foo");
	}
	
	public void doSuperFoo()
	{
		Console.WriteLine("Do Super Foo");
	}
}
</pre>
<p>By not specifying an access modifier on SuperFoo's implementation of DoFoo and prefixing the method with the interface name we have defined an explicit interface implementation for that method. In the above code f can only see doSuperFoo and f2 can only see doFoo. This can be especially useful when a class implements 2 interfaces with overlapping methods/properties, this allows you to define separate implementations of the members in the implementing class and have the type choose the correct member based on what type the class has be casted to. </p>