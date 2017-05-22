---
layout: post
title: SQL Server Parameter Sniffing
date: '2017-05-22 07:20:38'
---
If you've ever search for SQL Server parameter sniffing you've probably read all sorts of bad things about it. In truth parameter sniffing is an optimization technique SQL uses when compiling query plans and for the most part it works well and significantly improves overall performance vs not using parameter sniffing.