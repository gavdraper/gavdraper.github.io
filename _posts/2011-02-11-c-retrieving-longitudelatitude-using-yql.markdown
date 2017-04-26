---
layout: post
title: C# Retrieving Longitude/Latitude Using YQL
date: '2011-02-11 11:39:11'
---

If you've ever wanted a user to be able to enter their location in a format like "London, England" and be able to retrieve an approximate Longitude Latitude for that location the Yahoo location services pretty good easy way of achieving this. Users can enter their location in a variety of formats and Yahoo will do its best to guess what they mean.

I'm going to run through a simple example of this using C# ASP.Net MVC but you could easily adapt the code for most languages.

The first step is to write a controller method to query Yahoo's location services and return a JSON result containing the name of the location, the longitude and the latitude.
<pre class="brush: csharp; toolbar: false;">       public JsonResult GetLocation(string location)
       {
            if (location != "")
            {
                location = Server.UrlEncode(location);
                var RequestUrl = string.format("http://query.yahooapis.com/v1/public/yql?q=select%20*%20from%20geo.places%20where%20text%3D%22{0}%22&amp;format=xml",location)
                HttpWebRequest request = WebRequest.Create(RequestUrl) as HttpWebRequest;
                System.Xml.XmlDocument doc = new System.Xml.XmlDocument();
                using (HttpWebResponse response = request.GetResponse() as HttpWebResponse)
                {
                    doc.Load(response.GetResponseStream());
                }
                try
                {
                    string Country = doc.GetElementsByTagName("country")[0].InnerText;
                    string Admin1 = doc.GetElementsByTagName("admin1")[0].InnerText;
                    string Locality1 = doc.GetElementsByTagName("locality1")[0].InnerText;
                    string Latitude = doc.GetElementsByTagName("latitude")[0].InnerText;
                    string Longitude = doc.GetElementsByTagName("longitude")[0].InnerText;
                    return Json(new { success = true, location = Locality1 + ", " + Admin1 + ", " + Country, latitude = Latitude, longitude = Longitude }, JsonRequestBehavior.AllowGet);
                }
                catch (Exception ex)
                {
                    return Json(new { success = false, location = "", latitude = "", longitude = "" }, JsonRequestBehavior.AllowGet);
                }
            }
            return Json(new { success = false, location = "", latitude = "", longitude = ""}, JsonRequestBehavior.AllowGet);
        }</pre>
As you can see the XML parsing needs some tidying up to be used in the real world but I wanted to keep it simple here. Yahoo actually returns and Array of locations with the most likely being the first one, in this example I'm only looking at the first item but again in the real world you might want to show the list to your users so they can pick the correct one.

The reason I'm returning the location name is because Yahoo often provides the location name different to how the user provided it, In this case when the user exits the Location textbox I'm going to call this method and update the textbox with the location returned by Yahoo so the user gets the chance to see how Yahoo has interpreted their location and change it if needed.

At the top of your view where you want to take the location add the following JavaScript
<pre class="brush:js;toolbar: false;">&lt;script type="text/javascript"&gt;
CurrentLocation = "";
$(document).ready(function() {
    $("#LocationTextBox").focus(function() {
        CurrentLocation = $(this).val();
    });
   $("#LocationTextBox").blur(function() {
        if ($(this).val() != "" &amp; $(this).val() != CurrentLocation)
        {
            $.getJSON(
                          "/Home/GetLocation",
                          { "location": $('#LocationTextBox').val() },
                          function(data, textStatus) {
                              if(data.success == true)
                              {
                                  $('#LocationTextBox').val(data.location);
                                  $('#Longitude').val(data.longitude);
                                  $('#Latitude').val(data.latitude);
                              }
                          }
                    );
        }
    });
});
&lt;/script&gt;</pre>
You'll also need somewhere to store the longitude and latitude, in most cases you wont be interested in the user seeing these values so a HiddenTextBox is fine for this and makes the values then easily postable back in the form. Add the following text inputs to your form.
<pre class="brush: csharp; toolbar: false;">
&lt;%= Html.Hidden("Latitude") %&gt;
&lt;%= Html.Hidden("Longitude") %&gt;
Location : &lt;%=Html.TextBoxFor(m =&gt; m.Location, new { @id = "LocationTextBox" })%&gt;
</pre>
If you now go in to the form and enter a location of London England then exit the textbox your longitude and latitude will get updated to point to London and also the name will get updated according to what Yahoo returned, in this case "London, England, United Kingdom". Just to note you would also want to grab the location when the form is posted in case for some reason the onblur didn't fire or didn't have time to finish before the form was submitted.