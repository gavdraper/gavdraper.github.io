---
layout: post
title: Building Visual Studio Deployment Projects With MSBuild
date: '2012-02-17 08:05:53'
---

<p>I recently hit what I would call a fairly serious limitation of MSBuild and that is that it cant build Visual Studio Deployment Projects. This is a bit of a show stopper when your trying to build a one click release script. </p>  <p>After some Googling it turns out that there are two solutions, the correct solution and the hack. </p>  <p><strong>Correct Solution : </strong>Move away from Visual Studio Deployment projects and use something like Wix. The downside is if you're already deeply invested in using Visual Studio Deployment projects this can be quite a lot of work and possible not a route you want to go down.</p>  <p><strong>Hack : </strong>Use MSBuild to call Visual Studio from the command line asking it to build the Deployment Project for you. The downside of this is your build server will need to have Visual Studio installed. </p>  <p>In my case I went with the Hack as it was the quickest way to get things working with the existing architecture.</p>  <p>I already had my MSBuild script doing the following</p>  <ol>   <li>Grab latest source code from source control </li>    <li>Increment Assembly version numbers </li>    <li>Build the solution in release mode (using MSBuild task) </li> </ol>  <p>From here I wanted to increment the version number for my Deployment Project and build it in release mode. </p>  <p>*Warning* hacks and bad code ahead…</p>  <p>To increment the Deployment Projects version number I wrote a little inline C# task into the build script to edit the project file, the task takes a path to the Deployment Projects project file and a version number (in the format of 1.2.3) to set the installer to. It sets 3 variables inside the project file both ProductCode and PackageCode get set to new GUIDs and the ProductVersion gets set to the version passed in to the task. The code for this task looked like this….</p>  <pre class="brush: csharp; gutter: false; toolbar: false;">&lt;UsingTask TaskName =&quot;SetInstallerVersion&quot; TaskFactory=&quot;CodeTaskFactory&quot; AssemblyFile=&quot;$(MSBuildToolsPath)\Microsoft.Build.Tasks.v4.0.dll&quot;&gt;
  &lt;ParameterGroup&gt;
    &lt;InstallerPath ParameterType=&quot;System.String&quot; Required=&quot;true&quot;/&gt;
    &lt;Version ParameterType=&quot;System.String&quot; Required=&quot;true&quot;/&gt;
  &lt;/ParameterGroup&gt;
  &lt;Task&gt;
    &lt;Code Type=&quot;Fragment&quot; Language=&quot;cs&quot;&gt;
       var installerContents = &quot;&quot;;
       using (var sr = new System.IO.StreamReader(InstallerPath))
       {
         installerContents = sr.ReadToEnd();
         sr.Close();
       }
       var reProductCode = new System.Text.RegularExpressions.Regex(@&quot;(?:\&quot;&quot;ProductCode\&quot;&quot; =\&quot;&quot;8.){([\d\w-]+)}&quot;);
       var rePackageCode = new System.Text.RegularExpressions.Regex(@&quot;(?:\&quot;&quot;PackageCode\&quot;&quot; =\&quot;&quot;8.){([\d\w-]+)}&quot;);
       var reProductVersion = new System.Text.RegularExpressions.Regex(@&quot;&quot;&quot;ProductVersion&quot;&quot; =&quot;&quot;8:[0-9\.]*&quot;&quot;&quot;);
       installerContents = reProductCode.Replace(installerContents,&quot;\&quot;ProductCode\&quot; = \&quot;8:{&quot; + Guid.NewGuid().ToString().ToUpper() + &quot;}&quot;);
       installerContents = rePackageCode.Replace(installerContents,&quot;\&quot;PackageCode\&quot; = \&quot;8:{&quot; + Guid.NewGuid().ToString().ToUpper() + &quot;}&quot;);
       installerContents = reProductVersion.Replace(installerContents,&quot;\&quot;ProductVersion\&quot; = \&quot;8:&quot; + Version + &quot;\&quot;&quot;);
       using(var sw = new System.IO.StreamWriter(InstallerPath,false))
       {
         sw.Write(installerContents);
         sw.Close();
       }
    &lt;/Code&gt;
  &lt;/Task&gt;
&lt;/UsingTask&gt;</pre>

<p>OK so at this point the version numbers have been updated and we are ready to build the installer. I’d read how to do this by executing devenv.com from a few different sites and the suggested method was this </p>

<pre class="brush: xml; gutter: false; toolbar: false;">&lt;DevEnvExePath&gt;C:\Program Files (x86)\Microsoft Visual Studio 10.0\Common7\IDE\devenv.com&lt;/DevEnvExePath&gt;
&lt;DeploymentProjectPath&gt;c:\temp\myinstaller.vdproj&lt;/DeploymentProjectPath&gt;

&lt;Target Name=&quot;BuildInstaller&quot;&gt;
  &lt;Exec Command=&quot;&amp;quot;$(DevEnvExePath)&amp;quot; &amp;quot;$(DeploymentProjectPath)&amp;quot; /build Release&quot;/&gt;
&lt;/Target&gt;</pre>

<p>When I tried to run the build code above it just kept throwing exceptions saying missing dependencies. It turns out that if your installer references the output from other projects in the solution (As most do) you need to point DevEnv at the solution then specify the Deployment Project to build. This way it first builds all the projects in the solution the installer needs the output from, then it builds that actual installer. My working code for building the deployment project looked like this….</p>

<pre class="brush: xml; gutter: false; toolbar: false;">&lt;DevEnvExePath&gt;C:\Program Files (x86)\Microsoft Visual Studio 10.0\Common7\IDE\devenv.com&lt;/DevEnvExePath&gt;
&lt;SolutionPath&gt;c:\temp\mysolution.sln&lt;/SolutionPath&gt;
&lt;DeploymentProjectPath&gt;c:\temp\myinstaller.vdproj&lt;/DeploymentProjectPath&gt;

&lt;Target Name=&quot;BuildInstaller&quot;&gt;
  &lt;Exec Command=&quot;&amp;quot;$(DevEnvExePath)&amp;quot; &amp;quot;$(SolutionPath)&amp;quot; /build quot;Release&amp;quot; /project &amp;quot;$(DeploymentProjectPath)&amp;quot;&quot;/&gt;
&lt;/Target&gt;</pre>

<p>At this point I removed the step I already had to build the solution as this is no longer needed now the solution and installer are being built in the same step. Its worth also noting that if you wanted to introduce logging you could add something like “&amp;gt; BuildLog.txt” to the end of the Exec command to pipe the output from Visual Studio in to a text file called BuildLog.txt.</p>

<p>I am in no way encouraging you to use DevEnv to build projects from MSBuild rather than the standard MSBuild tasks but in cases like this it's pretty much the only option. </p>