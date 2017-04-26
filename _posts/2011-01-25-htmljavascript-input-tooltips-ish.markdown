---
layout: post
title: HTML/Javascript Input Tooltips (ish)
date: '2011-01-25 11:25:38'
---

I implemented a tooltip'ish feature into a website I worked on a year or so back and
have since found myself including it in more and more sites. It allows you to add
an attribute to your input fields containing some instructive text for what you expect
the user to enter into these fields, a bit of javascript then displays this tip on
the page. This can be displayed in a fixed area or it can be dynamic so the tip automatically
appears to the right of the input.

It has 2 modes DynamicTop where the tool tip will align itself with the top of the
input, and Fixed where by the tool tip stays in a fixed area of the page.

Here is what it looks like when its dynamic...

With Title field focused
<img src="/content/images/WPImport/2011/01/Dynamic1.jpg" alt="" />

With Email field focused

<img src="/content/images/WPImport/2011/01/Dynamic21.jpg" alt="" />

Heres a step by step to creating this functionaility

Create the following directory/file structure
<ul>
	<li> index.htm</li>
	<li> Style\Site.css</li>
	<li> Scripts\InputEvent.js</li>
	<li> Scripts\jquery-1.4.4.min</li>
</ul>
Obviously replace the jQuery file with the current version of jQuery downloadable
from http://www.jquery.com

In Site.css add the following code
<pre class="brush: csharp; toolbar: false;">.Clear
{
	clear:both;
	height:0px;
}
.FormRow
{
	width:600px;
}

.FormLabel
{
	width:100px;
	float:left;
}
.FormInput
{
	width:400px;
	float:left;
}

/*Forms***********/
.tooltip {
    display:none;
    background-color:#EEEEEE;
    position:absolute;
    padding-left:5px;
    padding-right:5px;
    padding-top:10px;
    padding-bottom:10px;
    width:300px;
}</pre>
If you want to fix the tooltip in place also specify you can either make the tooltip
class not asbsolute and move the div on the page to appear where you want it or you
can specify and top and left to fix it absolutely.

In this example to see fixed working how it looked in the screen shots just add "left:300px"
to the tooltip class.

In index.htm add the following code
<pre class="brush:xml">  &lt;title&gt;Tooltip Shizz&lt;/title&gt;
    &lt;meta http-equiv="Content-Type" content="text/html;charset=utf-8"&gt;
    &lt;link href="Style/site.css" rel="stylesheet" type="text/css"&gt;   
    &lt;script src="Scripts/jquery-1.4.4.min.js" )"="" type="text/javascript"&gt;&lt;/script&gt;        
    &lt;script src="Scripts/InputEvent.js" type="text/javascript"&gt;&lt;/script&gt;
 
 
&lt;script type="text/javascript"&gt;
    DynamicTop = true
&lt;/script&gt;
  
&lt;span class="tooltip" style="z-index: 1000"&gt; &lt;/span&gt;
 
&lt;div class="FormRow"&gt;
    &lt;div class="FormLabel"&gt;Title&lt;/div&gt;
    &lt;div class="FormInput"&gt;&lt;input id="InputTitle" type="text" helptip="Enter your title e.g MR"&gt;&lt;/div&gt;
    &lt;div class="Clear"&gt; &lt;/div&gt;
&lt;/div&gt;
&lt;div class="FormRow"&gt;
    &lt;div class="FormLabel"&gt;Forename&lt;/div&gt;
    &lt;div class="FormInput"&gt;&lt;input id="InputForename" type="text" helptip="Enter your forename e.g Gavin"&gt;&lt;/div&gt;
    &lt;div class="Clear"&gt; &lt;/div&gt;
&lt;/div&gt;
&lt;div class="FormRow"&gt;
    &lt;div class="FormLabel"&gt;Surname&lt;/div&gt;
    &lt;div class="FormInput"&gt;&lt;input id="InputSurname" type="text" helptip="Enter your surname e.g Draper"&gt;&lt;/div&gt;
    &lt;div class="Clear"&gt; &lt;/div&gt;
&lt;/div&gt;
&lt;div class="FormRow"&gt;
    &lt;div class="FormLabel"&gt;Email&lt;/div&gt;
    &lt;div class="FormInput"&gt;&lt;input id="InputEmail" type="text" helptip="Enter your email e.g gavin@gdraper.com. Please note this is the address we will use should we need to contact you."&gt;&lt;/div&gt;
    &lt;div class="Clear"&gt; &lt;/div&gt;
&lt;/div&gt;</pre>
If you want fixed tooltips set DynamicTop to false.

In InputEvent.js add the following code
<pre class="brush: csharp; toolbar: false;">OldVal = "";
DynamicTop = false;

function showToolTip(SelectedElement) {
    $(".tooltip").html($(SelectedElement).attr("helptip"));
    $(".tooltip").css("display", "block");
    if(DynamicTop == true)
    {
		var pos = $(SelectedElement).parent().parent().offset();
		var top = pos.top;
		var left = pos.left + 290;
		$(".tooltip").css("position", "absolute");
		$(".tooltip").css("top", top);
		$(".tooltip").css("left", left);
    }
}

$(document).ready(function () {
    $("input:text").focus(function () {
        showToolTip(this);
        OldVal = $(this).val();
    });
    $("input:password").focus(function () {
        showToolTip(this);
        OldVal = $(this).val();
    });
    $("textarea").focus(function () {
        showToolTip(this);
        OldVal = $(this).val();
    });
    $("input:password").blur(function () {
        $(".tooltip").css("display", "none");
    });
    $("textarea").blur(function () {
        $(".tooltip").css("display", "none");
    });
    $("input:text").blur(function () {
        $(".tooltip").css("display", "none");
    });
});</pre>
Thats it if you now open index.htm in your browser you should see the tooltips

In this case in the JS I have hard coded to position the tooltip 290 pixels to the
right of the input div. In the real world you will want to move this from the javascript
file and specifiy it on your page so you can vary the left position from page to page
to account for different sized forms.