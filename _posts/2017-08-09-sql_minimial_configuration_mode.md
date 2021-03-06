---
layout: post
title: Fixing a SQL Server That Wont Start Due To Max Memory Being Set Too Low
date: '2017-08-09 06:05:38'
tags: maintenance
---

I recently came up against a SQL Server instance that wouldnt start, after going through the event log the reason it was having problems was that it had been set to a max memory of 10mb. 

At this point I was unable to change the max memory setting because I couldnt bring the instance online. SQL Server has a minimal configuration argument you can supply whereby it ignores most configuration settings and comes up in default mode. I was able to get the instance up in single user mode using this setting, at which point I changed the max memory back to a sensible value and restarted the server in normal configuration mode and all was good again.

If for any reason you can't start a SQL Server instance due to a configuration change you can start it in minimal configuration mode so you can fix the offending settings by doing the following...

1. Log on to the computer running the SQL instance
1. Open the SQL Server configuration manager
1. In start up arguments add a new one of -f
1. Start the instance
1. Change any settings that need changing
1. Remove the -f argument from startup parameters
1. Restart the instance

