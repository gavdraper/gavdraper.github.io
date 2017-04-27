---
layout: post
title: Switch To SSL With CloudFlare
date: '2017-04-28 19:05:38'
---

## SSL All The Things
Browsers are starting to highlight sites without SSL and insecure in a bid to push everyone to using SSL. This is a great move and the sooner we can get to near 100% sites running on SSL the better. Google are also apparently going to be ranking sites running over SSL higher than their non encrypted counterparts. In the past switching to SSL has been a bit of a pain from expensive certififates to CA's that don't do the correct checks and hosts that don't allow SSL on their free/lower priced options.

## Why CloudFlare
CloudFlare is a lot of things, they insert themselves into the DNS path to your site, effectively becoming a man in the middle. This allows them to offer a host of services from CDN, DDoS protection, SSL, Page Rules, HTML Rewrites. They have a number of Enterprise level features but everything I discuss here can be done on their free plan. 

CloudFlare is a bit different. They create and insert the certificate into the connection without you having to make any changes to your webserver or even any of your code if you don't want to. They allow rewrite rules to force all resources in your pages to https. 

## As Easy As...
So you've got an existing site on a host and you've got a domain name registered, the only changes you need to make to switch to SSL is to point your Domain DNS server to your CloudFlare account which you then configure to point back to your host.