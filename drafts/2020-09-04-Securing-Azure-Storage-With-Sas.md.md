---
title: Securing Azure Storage With Shared Access Signatures
date: 2020-09-04 00:00:00
subtitle: ''
description: "Blah Blah"
featured_image: '/images/hero-images/hd-storage.jfif'
---

All demos in this post can also be accomplished easily in the portal, I'm avoiding using that purely because the UI changes so often and any screenshots I post are out of date instantly, if you want to follow along you'll need to download and sign into the Azure CLI. I'm using powershell here but you should be able to follow along in any shell.

Lets first create a new resource group and a storage account within it to play with...

{% highlight bash %}
$ az group create `
    --name storagedemo `
    --location ukwest 
$ az storage account create `
    --name storagesasdemo `
    --resource-group storagedemo `
    --location ukwest `
    --sku Standard_LRS `
    --kind StorageV2 `
    --allow-blob-public-access false
$ az storage container create `
    --name sasblobdemo `
    --public-access container `
    --account-name storagesasdemo
{% endhighlight %}

Now lets try to list the contents of that container using the REST endpoint to proove we don't have any public access enabled...

{% highlight bash %}
$ curl "https://storagesasdemo.blob.core.windows.net/sasblobdemo?restype=container&comp=list"

'curl : <?xml version="1.0" encoding="utf-8"?><Error><Code>PublicAccessNotPermitted</Code><Message>Public access is not permitted on this
storage account.
RequestId:25c8fb40-601e-0014-4a90-82e7ff000000
Time:2020-09-04T07:50:46.9199184Z</Message></Error>
At line:1 char:1
+ curl "https://storagesasdemo.blob.core.windows.net/sasblobdemo?restyp ...
+ ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : InvalidOperation: (System.Net.HttpWebRequest:HttpWebRequest) [Invoke-WebRequest], WebException
    + FullyQualifiedErrorId : WebCmdletWebResponseException,Microsoft.PowerShell.Commands.InvokeWebRequestCommand'
{% endhighlight %}

Access denied, just as we had hoped now lets grab one of our accounts access keys to try using that..

{% highlight bash %}
$ az storage account keys list `
    --account-name storagesasdemo
{% endhighlight %}

At which point you should see your two access keys...

{% highlight json %}
[
  {
    "keyName": "key1",
    "permissions": "Full",
    "value": "E50mo2yCbs9HKyEORiucw3dxUisVyge/MAH7WvSxnzXeJefAmHkHSfM2GUsrnnwPUi2vj+Bsv6ULpFbRoZ2bWg=="
  },
  {
    "keyName": "key2",
    "permissions": "Full",
    "value": "W1j8nyJoRwla8jS40p9vYUPxO8ZgK6iJg5sAvujHX6v57CVXll5n+df0U9THpdUIZKjyDMo9y22+4sd9oU2UJw=="
  }
]
{% endhighlight %}

These keys give FULL access to storage Read, Write, List Delete. We can then use this key to make a rest request to get the container contents...

{% highlight bash %}
{% endhighlight %}


## Clean Up
Finallaly lets remove everything we've created to prevent any unwanted charges...

{% highlight bash %}
$ az group delete --name storagedemo
{% endhighlight %}