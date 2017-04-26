---
layout: post
title: Getting Under The Hood
date: '2015-03-19 20:47:00'
---

Whenever a piece of software does something that makes me think this is awesome I find myself blocking out the actually software and picturing how it's working behind the scenes

- How did they implement this
- How can I use this
- How can I modify this

I often think as developers we have this urge to tinker and understand how things work. As someone who has this urge I can't think of a better industry to scratch that itch than software.

- For the most part this is no risk of blowing something up.
- If something goes wrong it's pretty easy to revert back to the original state without having to buy new parts. 
- The freely available information on the web for learning is growing at an incredible rate.
- Javascript! The language people love to hate, If it's on the web you can view/tinker all you like.
- Open Source just pick up any projects the interest you and hack away.

I recently switched from Spotify to Deezer mainly due to a year’s free subscription coming with a Sonos 1. I imported my playlists across from Spotify and began using the Deezer web client. It quickly became apparent that there were a couple of issues with long playlists

They get paged and initially only the first 40 tracks are loaded. Subsequent pages are loaded if you manually click a track further down the playlist. However the following actions do not load the subsequent pages and leave you only the first 40 tracks

1. Let a playlist play though without touching the controls and it will loop back to track 1 after track 40.
2. Click the next and back buttons.
3. Click the shuffle button.

This can be got round by loading a playlist and manually starting the last track in the list. At this point all the tracks are loaded in and you can perform any of the actions above with no issue. I decided to do some digging and see if I could provide any valuable information in a bug report to Deezer. It really is amazing how much you can learn with just the Chrome Dev tools and Fiddler. Below is the process I went through...

I opened Deezer in chrome, picked one of my long playlists and hit F12 to display the Developer Tools. Once Open I switched to the sources tab to see what JavaScript files they were loading. In this case they were loading a lot of content from different places, I decided to start by only looking at the JavaScript they were loading off their own CDN.

![](/content/images/2015/Mar/1.jpg)

By then clicking on one of the JS files you can see it's had all its formatting and whitespace removed.

![](/content/images/2015/Mar/2.jpg)

Luckily in this case it's not been minified and variable names are intact making debugging much more manageable. First things first though we need to sort out the formatting. I took a copy of the 5 JS files in the Deezer CDN, loaded them in to my editor and ran JS formatter on them to reformat them. I then saved these formatted JS files to my disk.

![](/content/images/2015/Mar/3.jpg)

We now need to get the Deezer site to use our local JS files rather than the ones on their CDN. To do this I used fiddler and created an auto responder rule so when one of those files is requested it serves up the one on my disk. 

To do this...

1. Open Fiddler and make sure "Capture Traffic" is checked in the file menu
2. Refresh the Deezer website
3. Find the files you want to swap out in the capture list and click one
4. On the right click the Auto Responder Tab
5. Check "Enable Automatic Responses"
6. Click Add Rule and it should create a new rule for the file you selected
7. Click the dropdown in the bottom right and click find file
8. Pick the file you saved to your local disk


Now when chrome requests that file from the Deezer CDN Fiddler will serve the local version.

Once this has been done for each of the JS files we're interested in we can start debugging with nicely formatted code. At this point I ran a few searches across the code for things like

- nextSong
- getSong
- nextTrack

You can do a global search across the sites source by pressing Ctrl+Shift+F in dev tools.

After a bit of reading the source I could see most of the stuff in core referenced an object called _dzPlayer that contained my first 40 playlist tracks. After some more digging I found that the reason only the first 40 are initially loaded is...

```language-javascript
var params = {
   playlist_id: playlist.id,
   lang: SETTING_LANG,
   start: 0,
   nb: 40,
   tags: false
};
```

These params are then used to fire an AJAX call to request tracks 0 to 40. I wanted to then see where more tracks were paged in, after finding a function called nextSong I decided to put a console.log in there to alert the current track number and the amount of tracks currently loaded with this...

```language-javascript
console.log(_dzPlayer.trackListIndex + ":" + _dzPlayer.trackList.length);
```

I refreshed the page and didn’t see any log messages in my console. After a quick search for console.log I found that Deezer set Console.Log to {} when in release mode, I like this approach as it stops debug messages appearing on users machines. I removed the line that reset console.log, refreshed the Deezer client and sure enough there were my log messages appearing in the console each time a track ended or I clicked skip.

My logs looked like this

![](/content/images/2015/Mar/4-1.jpg)

So it's clearly never paging more results in. The jump from 40 to 80 then to 814 is where I manually clicked a track in the grid triggering the playlist to be fully loaded.

So it looks like the loading of the rest of the playlist only occurs if you click a track. After some more searching I found a playAllTracks function in naboo.js. This has a loadAllTracks call which sounds like exactly what I was looking for. I put a breakpoint in here and it became apparent this is only being called when tracks are manually selected and not when you use the Next/Back buttons or a song finishes.  

It looks like the left panel of the client (The bar with the Next/Back/Shuffle commands) is using the JS code in core.js which doesn’t do any paging. Where as the playlist grid on the right is using naboo.js both of which have a slightly different play method. Naboo has this play method...

```language-javascript
this.play = function($line) {
   try {
      ....
      switch (options.type) {
      case "playlist":
         oSelf.playAllTracks(true);
         break;
      ...
      }
      dzPlayer.play({
         id: options.id,
         type: type,
         data: songsList,
         index: index,
         context: oSelf.getContext()
      });
      return true;
   } catch (e) {
      error.log(e);
   }
};
```

Where as core has this play method...

```language-javascript
play: function($this, data) {
   try {
      ...
      dzPlayer.play(params);
      EventsDelegation.setSentToMobile($this, data);
      return true;
   } catch (e) {
      error.log(e);
   }
}
```

As you can see naboo calls PlayAllTracks where as Core does not. I've no idea why they are using 2 different play methods or what the separation between Naboo/Core really is. I would have thought the grid could be raising messages that then get piped through the same functions in core as Next/Back and Shuffle but this is not the case. This is a fairly large app and I've only spent an hour or so looking, I'm sure there are lots of good reasons for this separation of naboo and core but as an outsider I have no idea what they are. 

At this point I've found this issue and could probably hack an ugly fix in but really it's time to raise a bug report with this information so it can be fixed at the source. I then went looking for somewhere to log bugs for Deezer but have not been able to find anyway to do this, I've tweeted both the Deezer and DeezerDevs account with no response. That's when I decided to write it up here as I think this has been a fun exercise and it's really neat how much you can do with remote JavaScript.

