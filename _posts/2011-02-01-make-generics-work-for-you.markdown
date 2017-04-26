---
layout: post
title: Make Generics Work For You
date: '2011-02-01 08:16:42'
---

Without a doubt Generics have been one of the most used new .Net features introduced since .Net 2.

I'm not going to go in to detail about exactly what they are as I believe by now mostÂ people will know what they do and will have used them numerous times through thingsÂ like List&lt;t&gt; and Nullable&lt;t&gt;.

What I have noticed is that a lot of people use them with the new Generic types includedwith .Net 2 and not a lot of people use them to write their own Generic types when it may have made sense too. I think this is because some people think its a lot more complicated than it actually is to create your own Generic types.

I'm going to show a simple example here of how this could be done. I'll create a new type that is basically a list but instead of just holding one value per item it will hold 3 values all of which can be different types.

The example will be made up of 2 different types the first being NinjaList and the second being NinjaListItem. When you create an instance of NinjaList you pass it the 3 types you want to use for your NinjaListItem values and it will then take care of the rest for you. NinjaList is the actual list and will hold a List of NinjaListItems that can be added/removed and enumerated over.

Firstly we need to create our NinjaListItem type, this is a very simple generic typeand looks like this
<pre class="brush:csharp;toolbar: false;">public class NinjaListItem&lt;TName, TVal, TDesc&gt;
    {
        public TName Name { get; set; }
        public TVal Value { get; set; }
        public TDesc Description { get; set; }
    }</pre>
The words between the angle brackets are how we specify this class is generic and there are 3 types we want the user to specify when they create an instance of this class. Notice how the same names (TName, TVal, TDesc) are used when declaring the properties this is telling the properties to be of the type the user specifies when creating an instance of this type.

For example to now create an instance of NinjaListItem with the Name and Description being of type string and the Value being of type int you would do the following
<pre class="brush:csharp;toolbar: false;">var NinjaItem = new NinjaListItem&lt;string,int,string&gt;();</pre>
If you then type "NinjaItem." in your code and look at the intellisense options you will see visual studio now knows Name/Description are strings and that value is an int. Everything is type safe and once you have declared the object it works as though its a normal statically typed object.

Ok now for the NinjaList class, this is pretty much a wrapper around the NinjaListItem with the added functionality of IEnumerable and overriden indexer so you can loop over the list using ForEach and return any item at position x. I've also added Add/Remove methods for creating and removing NinjaListItem's from the collection.
<pre class="brush:csharp;toolbar: false;">class NinjaList&lt;TName, TVal, TDesc&gt; : IEnumerable&lt;NinjaListItem&lt;TName,TVal,TDesc&gt;&gt;
    {
        List&lt;NinjaListItem&lt;TName,TVal,TDesc&gt;&gt; items = new
                 List&lt;NinjaListItem&lt;TName,TVal,TDesc&gt;&gt;();
        public int Count { get { return items.Count; } }

        public IEnumerator&lt;NinjaListItem&lt;TName, TVal, TDesc&gt;&gt; GetEnumerator()
        {
            foreach (var item in items)
            {
                yield return item;
            }
        }

        IEnumerator IEnumerable.GetEnumerator()
        {
            return GetEnumerator();
        }

        public void Add(TName name, TVal val, TDesc desc)
        {
            items.Add(new NinjaListItem&lt;TName, TVal, TDesc&gt;()
            {
                Name = name,
                Value = val,
                Description = desc
            }
            );
        }

        public void RemoveAt(int index)
        {
            items.RemoveAt(index);
        }

        public bool Remove(NinjaListItem&lt;TName, TVal, TDesc&gt; item)
        {
            return items.Remove(item);
        }        

        public NinjaListItem&lt;TName,TVal,TDesc&gt; this[int index]
        {
            get
            {
                if (index &gt;= items.Count | index &lt; 0)
                    throw new IndexOutOfRangeException();
                return items[index];
            }
            set
            {
                if (index &gt;= items.Count | index &lt; 0)
                    throw new IndexOutOfRangeException();
                items[index] = value;
            }
        } 
    }</pre>
After examining NinjaListItem there's not much new here in the way of generics its just using more of the same. This simple class demonstrates how to Use Generics in custom types, Overriding the [] operator and Implementing IEnumerable.

If you want to run this example the following code demonstrates how it suports both indexers and foreach loops.
<pre class="brush:csharp;toolbar: false;">   var nList = new NinjaList&lt;string, string, string&gt;();
   nList.Add("Name", "val", "desc");
   nList.Add("Name1", "val", "desc");
   nList.Add("Name3", "val", "desc");
   nList.Add("Name4", "val", "desc");
   //this demonstrates the overriden indexed
   for (int i = 0; i &lt; nList.Count; i++)
   {
      Console.WriteLine(nList[i].Name);
   }
   Console.ReadLine();
   //This demonstrates that fact that iEnumerable&lt;t&gt; was implemented.
   foreach (var i in nList)
   {
      Console.WriteLine(i.Name);
   }
   Console.ReadLine();</pre>
Hopefully this will give enough of a basis to implement your own generic types. Something I haven't covered here are generic methods and will try to put something together shortly to ive some examples of using these. By no means take this example as complete, it is just that an example and in the really world would want a lot more functionality to be usable.