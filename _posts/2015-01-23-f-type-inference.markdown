---
layout: post
title: F# Type Inference
date: '2015-01-23 08:36:22'
---

I'm just starting to look at functional languages from an Object Oriented background. One of the first things I thought was quite cool is F#'s type inference model and the fact that parameters types on functions can be inferred. 

Let's take this C# code....

```language-csharp
var myInt = 10
```

The C# compiler will infer the type of myInt to be int as that is what we have assigned to it. Nothing can be done away from this line to change the inferred type of myInt. F# takes this a step further by allowing function parameter types to be infered...

```language-csharp
let myFunction x y = x + y
```

If you have that line on it's own F# will infer x,y and the result of myFunction to be int because of how they are used in the function body. Probably as you would expect, however what I was surprised at is if you add the following line the infered types will change...

```language-csharp
let myFunction x y  = x + y
printfn "%s" (myFunction "1" "3")
```

The input and output types of myFunction will now be all string. This is quite powerful but also something to watch out for. Keep in mind the code calling myFunction could be no where near it in the codebase making it hard scan the code and see the types of functions, obviously you can check this by mousing over the function name in VS to get see a hint with the type. I really like the expessiveness of functional languages from what I've used so far, it will be interesting to see how long it takes for me to get to a point where I'm comfortable enough in the functional world to actually put it to good use.