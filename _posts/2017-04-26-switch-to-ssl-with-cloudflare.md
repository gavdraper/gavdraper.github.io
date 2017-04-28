---
layout: post
title: Switch Your Site To HTTPS Using CloudFlare
date: '2017-04-26 21:05:39'
---

## SSL All the Things
Browsers are starting to highlight sites without SSL as insecure in a bid to push everyone to using SSL. This is a great move and the sooner we can get to near 100% sites running on SSL the better. Google are also apparently going to be ranking sites running over SSL higher than their non-encrypted counterparts. In the past switching to SSL has been a bit of a pain from expensive certificates to CA's that don't do the correct checks and hosts that don't allow SSL on their free/lower priced options.

## Why CloudFlare
CloudFlare is a lot of things, they insert themselves into the DNS path to your site, effectively becoming a man in the middle. This allows them to offer a host of services from CDN, DDoS protection, SSL, Page Rules, HTML Rewrites. They have a number of Enterprise level features but everything I discuss here can be done on their free plan. 

CloudFlare is a bit different. They create and insert the certificate into the connection without you having to make any changes to your webserver or even any of your code if you don't want to. They allow rewrite rules to force all resources in your pages to https. 

## As Easy As...
So you've got an existing site on a host and you've got a domain name registered, the only changes you need to make to switch to SSL is to point your Domain DNS server to your CloudFlare account which you then configure to point back to your host.

Sign up for a CloudFlare account and follow the below steps...

1. Enter your domain name...
    ![CloudFlareScreenshot]({{site.url}}/content/images/2017-cloudflare-ssl/cloudflare-1.png)

1. All being well all your DNS records will be picked up and imported into CloudFlare. If for some reason they're not then you can manually add them later
    ![CloudFlareScreenshot]({{site.url}}/content/images/2017-cloudflare-ssl/cloudflare-2.png)

1. A business plan is selected by default, change that to the free plan. 
    ![CloudFlareScreenshot]({{site.url}}/content/images/2017-cloudflare-ssl/cloudflare-3.png)

1. The next screen gives you the details or where you need to point your domains namesevers. 
    ![CloudFlareScreenshot]({{site.url}}/content/images/2017-cloudflare-ssl/cloudflare-4.png)

1. In my case my domain is on Fasthosts so the change on the FastHosts control panel looks like this. It should be pretty similar no matter who the domain is with.
    ![CloudFlareScreenshot]({{site.url}}/content/images/2017-cloudflare-ssl/fasthosts-1.png)

1. The overview page on CloudFlare will change to active once the nameserver change has reached cloudflare. 
    ![CloudFlareScreenshot]({{site.url}}/content/images/2017-cloudflare-ssl/cloudflare-5.png)

1. SSL took a few hours to enable for me  but you can check this on the Crypto tab in CloudFlare. 
    ![CloudFlareScreenshot]({{site.url}}/content/images/2017-cloudflare-ssl/cloudflare-6.png)

1. At the bottom of the screen if you want HTTP resources rewritten to HTTPS without having to change the site then switch this setting on... 
    ![CloudFlareScreenshot]({{site.url}}/content/images/2017-cloudflare-ssl/cloudflare-7.png)

1. Lastly you can optionally create a page rule to divert al HTTP traffic to HTTPS 
    ![CloudFlareScreenshot]({{site.url}}/content/images/2017-cloudflare-ssl/cloudflare-8.png)

And... We're done.

All of the above can be done without CloudFlare by require numerous configurations on the WebServer and possibly a lot of changes to your website to switch over any content referenced on HTTP. Using CloudFlare this  can all be done in 5 minutes for free, I was really impressed how easy this made it to move all my content to HTTPS, not to mention all the other features CloudFlare offer that I've yet to explore