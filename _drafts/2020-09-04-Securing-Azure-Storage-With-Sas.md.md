---
title: Securing Azure Storage With Shared Access Signatures
date: 2020-09-04 00:00:00
subtitle: ''
description: "Blah Blah"
featured_image: '/images/hero-images/hd-storage.jfif'
---

Azure Storage has many options for security from VNets to AAD to Secure Access Key to Shared Access Signatures. This post will get you up to speed on setting up shared key access using Azure CLI, everything here can also be done via the portal or any of the other scripting options.

## Setup

Lets first create a new resource group and a storage account within it to play with...

{% highlight bash %}
$ az group create -n StorageDemoRG -l ukwest 
$ az storage account create -n demoaccount -g StorageDemoRG --sku Standard_LRS
$ az storage container create -n demo1 --account-name demoaccount
$ az storage container create -n demo2 --account-name demoaccount    
{% endhighlight %}

## Access Via Access Keys
Our new storage account comes with two access keys both of which give complete control inside storage account to add, modify, read and delete data from all containers within the account. 

Lets grabs these keys for a test...

{% highlight bash %}
$ az storage account keys list -n demoaccount
{% endhighlight %}

At which point you should see your two access keys...

{% highlight json %}
[
  {
    "keyName": "key1",
    "permissions": "Full",
    "value": "E50mo2yCbs9HKyEOR..."
  },
  {
    "keyName": "key2",
    "permissions": "Full",
    "value": "W1j8nyJoRwla8jS40..."
  }
]
{% endhighlight %}

To reitterate these keys give FULL access to their storage accounts Read, Write, List Delete. Lets use these keys in one of our new containers to upload a file and then list the container

{% highlight bash %}
$ az storage blob upload 
  -c demo1 
  -f "c:\Testfile.txt" 
  -n TestFile.txt 
  --account-name demoaccount 
  --account-key "REPLACEWITHKEY" 
$ az storage blob list 
  -c demo1 
  --account-name demoaccount 
  --account-key "REPLACEIWTHKEY"
{% endhighlight %}

All being well from our list command we'll get an array back containing details on the file we just uploaded.

{% highlight json %}
[
  {
    "container": "demo1",
    "name": "HelloWorld.txt",
    ...
  }
]
{% endhighlight %}

There is no way using access keys to restrict permissions on a container, file, or access type (Read, Delete, Write). 

## Restricting Access With Shared Access Keys

## Clean Up
Finallaly lets remove everything we've created to prevent any unwanted charges...

{% highlight bash %}
$ az group delete --name StorageDemoRG
{% endhighlight %}