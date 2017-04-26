---
layout: post
title: 'Using TDD To Create A Working Application In C# : Part 2'
date: '2012-01-14 11:37:53'
---

<h3><font style="font-weight: bold">Last FM Wrapper</font></h3>  <p>You know that web project we created? You can completely forget about that for now as this is TDD and all the ground work happens in your tests. </p>  <p>The first component this project needs is something to wrap around the Last.FM APIs track.getSimilar method. In this part of the series we are going to mock and write the tests for this class. </p>  <p>As this step is about tests and mocking we’re not going to be writing any logic to actually access LastFm, I’ve grabbed a few sample responses from their track.getSimilar method and have put them in XML files in the tests project so I can write methods to test handling the responses. In your test project create a folder called LastFMResponses and add <a href="/content/images/WPImport/2012/01/LastFMResponses.zip">these</a> XML files to it. Set the “Copy to output directory” property of each of these files to “Copy if newer”.</p>  <p>Create a new class in the test project called LastFmWrapperTests.cs, this is where we will be doing all our coding. Any classes or interfaces will be initially put in here whilst we test, then once we are happy everything passes we can move them out into the correct projects.</p>  <p>Lets tackle the main area first, get search results. In this case we’ll use the TenResults.xml file to test that when we get that response we can parse it into a List of Matches. </p>  <p>Add the following code to the top of your LastFMWrapperTests class</p>  <pre class="brush: csharp; gutter: false; toolbar: false;">private const string apiKey = &quot;b25b959554ed76058ac220b7b2e0a026&quot;;

[TestMethod]
public void SearchReturnsListOfTenMatchesFromTenResultsXml()
{
    var lastFmWrapper = new LastFmWrapper();
    var matches = lastFmWrapper.GetSimilarTracks(apiKey,&quot;Cher&quot;, &quot;Belive&quot;);
    Assert.AreEqual(10,matches.Count);
}</pre>

<p>Now we want to write the minimum amount of code to make this compile and the test to fail.</p>

<p>Add this to the bottom of your LastFMWrapperTests class</p>

<pre class="brush: csharp; gutter: false; toolbar: false;">//Temp store for creating objects to be tested before they are moved to their correct classes
public class Song
{
    public string Title { get; set; }
    public string Artist { get; set; }

    public static explicit operator Song(XElement xElem)
    {
        return new Song()
        {
            Title = (string)xElem.Element("name"),
            Artist = xElem.Element("artist") == null ? "" : (string)xElem.Element("artist").Element("name"),
        };
    }
}

public class LastFmWrapper
{
    private string GetResponseFromGetSimilarTracksApi(string apiKey, string artist, string track)
    {
        throw new NotImplementedException();
    }

    protected List&lt;Song&gt;  ParseResponseIntoMatches(string response)
    {
        var xmlDoc = XDocument.Parse(response);
        var songs = xmlDoc.Descendants("track").Select(xElem => (Song)xElem).ToList();
        return songs;
    }

    public List&lt;Song&gt;  GetSimilarTracks(string apiKey, string artist, string track)
    {
        var Response = GetResponseFromGetSimilarTracksApi(apiKey, artist, track);
        var matches = ParseResponseIntoMatches(Response);
        return matches();
    }
}</pre>
<p>You'll notice I've created an explicit cast override on the Song object this is so we can use LinqToXml to parse the XML into a list of Song objects.</p>
<p>The project will now build and if you run the test you will see it fails because of the NotImplementedException in GetResponseFromGetSimilarTracksApi. In this case we don't actually want to implement GetResponseFromGetSimilarTracksApi because that's going to be accessing external data outside of the tests, what we do want to test is that given the responses in our XML files we can return the expected result. To do this we need to bypass GetSimilarTracks and go straight to ParseResponseIntoMatches, which is why I have made it protected so we can extend the class with a testable version. </p>

<p>Add the following class to the bottom of LastFmWrapperTests</p>

<pre class="brush: csharp; gutter: false; toolbar: false;">public class TestableLastFmWrapper : LastFmWrapper
{
    public List&lt;Song&gt; TestParseResponseIntoMatches(string response)
    {
        return ParseResponseIntoMatches(response);
    }
}</pre>

<p>We can now change our SearchReturnsListOfTenMatchesFromTenResultsXml method to use the new TestableLastFmWrapper and TestParseResponseIntoMatches method to run the test. The method should look like this</p>

<pre class="brush: csharp; gutter: false; toolbar: false;">[TestMethod]
public void SearchReturnsListOfTenMatchesFromTenResultsXml()
{
    var lastFmWrapper = new TestableLastFmWrapper();
    string response;
    using (var sr = new StreamReader(Path.GetDirectoryName(Assembly.GetExecutingAssembly().Location) + &quot;\\LastFMResponses\\TenResults.xml&quot;))
        response = sr.ReadToEnd();
    var matches = lastFmWrapper.TestParseResponseIntoMatches(response);
    Assert.AreEqual(10,matches.Count,&quot;Expected 10 matches&quot;);           
}</pre>

<p>If you then run the test you will see it now correctly passes.</p>

<p>Notice how everything starts with writing a failing test then implementing just enough code to get that test to pass. </p>

<p>In the next part I’m going to pick up the pace a bit and implement the rest of the tests for the LastFmWrapper and then get the classes moved out of the test project and into their correct locations out .</p>