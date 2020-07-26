---
title: 'TableTopSocial'
subtitle: 'Multiplayer Gaming In Blazor'
date: 2020-05-14 00:00:00
description: Playing with client side blazor and SignalR to implement browser based multiplayer tabletop games
featured_image: '/images/hero-images/dice.jpg'
---

I have a thing for board games and more specifically collecting them. The global pandemic has somewhat put a stop to our regular boardgame nights and has had me frantically downloading any digital tabletop games I can find. One of the popular games in our group is One Night Ultimate Werewolf, a hidden identity game where most of the game takes place via conversation and voting. This seemed like an ideal fit for playing over Zoom but there were no digital versions of the game available, and that's where this project came from, and ugly UI over a functional codebase :)

I thought about using Angular initially, but then after reading a post on client-side Blazor with Web Assembly, I went all-in on that.

This project comprises of...

| Tech                 | Purpose        | 
|----------------------|---------------|
| ASPNet Core WebAPI    | HTTP API used by the client to get and update game |
| Client Side Blazor  (Web Assembly) | The user's interface to the game |
| GitHub Actions For CI/CD | Actions to run units tests and deploy to Azure |
| Azure WebApp | To host the server and client side apps |
| Azure App Insights | For Logging and diagnostics | 
| xUnit | Unit Tests |
| VSCode, Coverage and DotNet Watch | Continous Testing and Code Coverage |

Cool, takeaways from messing around with all this...

* VS Code can be incredibly powerful, it's never going to be a full replacement for VS + Resharper + NCrunch for me but for smaller projects like this with a little setup you can get a lot of the enterprisey features like live code coverage and continuous testing for free.
* Being able to code in C# in the Browser is incredibly productive for someone who knows C# better than Javascript.
* Not having to code client-side models in a different language to their server-side counterpart saves a lot of time and massively increased shared code.
* Other than slow load times on low-end devices client-side "Just Worked", when you consider what it's doing it is quite amazing. I still can't get used to using Linq in the Browser, but I love it!



<div class="gallery" data-columns="1">
	<img src="/images/posts/tabletop/coverage.PNG">
	<img src="/images/posts/tabletop/room.PNG">
	<img src="/images/posts/tabletop/cards.PNG">
	<img src="/images/posts/tabletop/dealt-cards.PNG">
	<img src="/images/posts/tabletop/witch.PNG">
	<img src="/images/posts/tabletop/witch2.PNG">
	<img src="/images/posts/tabletop/vote.PNG">
	<img src="/images/posts/tabletop/results.PNG">

</div>

