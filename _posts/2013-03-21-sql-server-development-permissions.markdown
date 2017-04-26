---
layout: post
title: SQL Server Development Permissions
date: '2013-03-21 13:49:32'
---

<p>After setting up a new SQL Server instance for development I found the following permissions were needed extra permissions were needed to enable key development features… (Note these were for a development instance and should not be needed on a production database)</p> <h3>Permission To View Query Plans</h3> <p>This needs to be granted for each database….</p><pre class="brush: sql; gutter: false; toolbar: false;">GRANT SHOWPLAN TO [myDomain\myUser]</pre>
<h3>Permission To View Server Activity Monitor</h3><pre class="brush: sql; gutter: false; toolbar: false;">USE master
GO
GRANT VIEW SERVER STATE TO [myDomain\myUser]</pre>
<h3>Permission To View Jobs/History</h3><pre class="brush: sql; gutter: false; toolbar: false;">USE mdsb 
GO 
EXECUTE 'sp_addrolemember 'SQLAgentReaderRole', [myDomain\myUser] </pre>
<p>If you need to be able to start/stop jobs then swap SQLAgentReaderRole for SQLAgentOperatorRole.</p>
<p>Are there any other permissions you need for your day to day development work? Leave a comment and I'll add them here.</p>