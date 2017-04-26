---
layout: post
title: ASP.Net MVC File Stream, Initiated From Javascript Post
date: '2015-01-28 07:40:31'
---

I was recently working on a page that does an AJAX post to an MVC controller passing quite a lot of parameters in the request. I needed to find a way to stream a file back to the browser as a result of a that Javascript call, for obvious reasons a file download can't be started from a stream sent in the response to an AJAX call. 

To solve this I ended up swapping the AJAX post out for a form post, the response of which can be a file stream. The code looked a bit like this....

MVC Controller

```language-csharp
public FileResult Download()
{
	//Fake file
	byte[] fileBytes = new byte[10];
	string fileName = "hello.pdf";
	return File(fileBytes, MediaTypeNames.Application.Octet, fileName);
}
```

Javascript to replace AJAX call

```language-javascript
var form = document.createElement("form");   
form.setAttribute("method", "post");
form.setAttribute("action", "DownloadFile/Download");
var hiddenField = document.createElement("input");
hiddenField.setAttribute("type", "hidden");
hiddenField.setAttribute("name", "Param1");
hiddenField.setAttribute("value", "HelloWorld");
form.appendChild(hiddenField);
document.body.appendChild(form);
form.submit();
```