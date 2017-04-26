---
layout: post
title: ASP.Net MVC/EF4 TryUpdateModel On Non Nullable Types
date: '2010-09-04 09:34:41'
---

I ran in to a problem to do with non nullable types in EF4 and ASP.Net MVC the other
day and spent a while trying to solve it in a tidy fashion.

The issue was all my entity objects have data annotations on them for validation.
This allows me in my controller methods call TryUpdateModel function when I want to
convert post data into an object, if this then returns false I can return the errors
to the user.

This has been working fine until I submitted a form with an empty text box that mapped
to a non nullable string in the entity framework, what I expected to happen here was
for it to check my data annotations see its a required field and for the TryUpdateModel
to return false so I could return the error to the user.

However what actualy happened was TryUpdateModel threw an exception because it couldnt
set this field to null. This still seems wrong to me but after some digging around
it appears to be by design, it looks like TryUpdateModel first populates the model
if any exceptions occur here they are thrown. It then runs your data annotations and
if any of them fail it returns false else it returns true.

After a lot of head scratching I finally settled on the following solution.
<ol>
	<li> Extending the particular entity with a partial class.</li>
	<li> Adding a new property called NewSiteName</li>
	<li> Set the getter to return the failing Name Field</li>
	<li> Set the setter to set the value of the failing Name field if the value is not null
else set it to "".</li>
</ol>
This now works fine, If you pass in Null the field gets set to "" and my required
field data annotations can handle the error as they should. Here is the code I ended
up using...
<pre class="brush: csharp; toolbar: false;">    public partial class Site
    {
        //This property is here so if you try to create a new site with no name
        //an error is not thrown as SiteName is non nullable TryUpdateModel fails
        //when you pass in a sitename of null so here I convert null to "" then pass
        //it along to the real Name field
        public string NewSiteName
        {
            get { return Name; }
            set
            {
                Name = value ?? "";
            }

        }

    }</pre>