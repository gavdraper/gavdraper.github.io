---
layout: post
title: Browser Based Web Notifications
date: '2014-03-01 21:49:10'
tags:
- javascript
---

3 of the 4 major browsers now support web notifications (Safari, Firefox, Chrome) and no where near enough sites are making use of this killer feature. One web app that does make good use of this is Gmail, if you've enabled it then it will show notifications when you get new emails. These notifications appear in some sort of notification window in a corner of the screen and are visible even if the browser is minimized or not the active window. 

Before we start looking at implementing it, it's worth making note that the current browser implementations require you to give each site that uses this feature permission. The site can request this permission itself and the user will be prompted to accept it. Another point to note is that in Chrome you are only able to request this permission after the user has performed an action ie clicked a button, it can not be requested on page load.

For this example I'll be using [Notify.js](https://github.com/alexgibson/notify.js) to do this as it provides a nice wrapper around the Web Notifcations API. I'm going to render a page that has a button to notifications, once that has been enabled another button will appear to display a notification. In reality notifications will normally be triggered in the background by some sort of polling or websocket message.

Lets start of with an empty very basic  HTML page

```language-markup
<!doctype html>
<html lang="en">
    <head>
        <meta charset="utf-8">
        <title>Web Notications Rock!</title>
    </head>
    <body>
    </body>
</html>
```

Lets say we then add the two buttons enable and show. The show button shouldnt display until notifications have been enabled so that would look something like....

```language-markup
<!doctype html>
<html lang="en">
    <head>
        <meta charset="utf-8">
        <title>Web Notications Rock!</title>
        <script src="http://code.jquery.com/jquery-1.11.0.min.js"></script>
    </head>
    <body>
        <button onClick="enableNotifications()">Enable Notifications</button>
        <hr/>
        <div id="NotificationActions" style="display:none;">
            Notification Text : <input type="text" id="txtNotification"/>
            <button onClick="showNotifications()">Display Notification</button>
        </div>

        <script type="text/javascript">
            var enableNotifications = function()
            {
                $("#NotificationActions").show();
            }

            var showNotifications = function()
            {
                var notificationText = $("#txtNotification").val();
                alert(notificationText);
            }
        </script>
    </body>
</html>
```

At this point we have the page stubbed out and now we just have to implement the actual notificaiton calls using Notify.js. To do this download [Notify.js](https://github.com/alexgibson/notify.js) and refernce it from the page. You can then use the Notify object in javascript to create Web Notifications.

```language-markup
<!doctype html>
<html lang="en">
    <head>
        <meta charset="utf-8">
        <title>Web Notications Rock!</title>
        <script src="http://code.jquery.com/jquery-1.11.0.min.js"></script>
        <script src="notify.js"></script>
    </head>
    <body>
        <button onClick="enableNotifications()">Enable Notifications</button>
        <hr/>
        <div id="NotificationActions" style="display:none;">
            Notification Text : <input type="text" id="txtNotification"/>
            <button onClick="showNotifications()">Display Notification</button>
        </div>

        <script type="text/javascript">
            var enableNotifications = function()
            {
                if (Notify.isSupported() && Notify.needsPermission())
                {
                    Notify.requestPermission();
                }
                $("#NotificationActions").show();
            }

            var showNotifications = function()
            {
                if (Notify.isSupported() && !Notify.needsPermission())
                {
                    var notificationText = $("#txtNotification").val()
                    var myNotification = new Notify('Notication Title', {
                        body: notificationText,
                    });
                    myNotification.show();
                }
            }
        </script>
    </body>
</html>
```

Assuming your browser supports them and you allowed the permission request triggered by the Enable Notifications button then the Display Notification button should show a notification on the screen. This post should give a good insight as to how easy it is to use Web Notifications with the help of Notify.js. Hopefully these will become more used and supported (come on IE!) in the near future as this is a great new feature available to web apps.