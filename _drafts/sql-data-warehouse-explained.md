---
layout: post
title: SQL Server Data Warehousing Explained
date: '2017-05-09 06:05:38'
---
Data warehousing is the act of transforming application database into a format more suited for reporting, this typically involves flattening the data.

Let's imagine we have an online store

SELECT
    dbo.Sales.Date,
    dbo.Customer.FullName,
    dbo.Product.Name,
    dbo.Product.Price
    dbo.Discount.Name,
    dbo.Discount.Amount
FROM
    Sales
    INNER JOIN Customer ON Customer.Id = Sales.CustomerId,
    INNER JOIN Product ON Product.Id = Sales.ProductId,
    INNER JOIN Discount ON Discount.Id = Sales.DiscountId

This is a fairly simple example but we can see a lot of joins are required to produce this report and as we want to do more and more aggregations on our data for reporting the joins will slow things down more and more. 

Lets pretend this has got to the point where our database is great for running our applications transactions and serving user requests but it's just not holding up the our reporting needs. At this point we can think about creating a data warehouse, in it's simplest form this just involves creating a new database with a new schema and setting up a process that copies data here from our application database. Given the above our warehouse could server the same report with a query like this

SELECT
    Date,
    CustomerName,
    ProductName,
    ProductPrice,
    Name,
    Amount
FROM
    Sales

Ok, so we've saved a few lines on what's probably a simple query to run anyway. Lets imagine the online shop we're talking about is Amazon and they want to see sales per month for the last year, we're then talking millions and millions of records that needs to be joined across 4 tables, in this case the warehouse will make a huge difference.

When reading about data warehousing there are a few commons terms that are used and they are Fact, Dimension, Star and Snowflake, lets take a look at each of these and what they mean.



