---
layout: post
title: Self Hosting WCF Services With Transport Security
date: '2013-03-15 08:17:52'
---

<p>When developing and debugging transport security WCF services it’s a real pain if each developer on the team needs to set up self signed certificates and a site in IIS in order to host the service locally. You can obviously define extra endpoints that don’t need transport security for debugging/testing but if your services require a binding like ws2007FederationHttpBinding that requires transport security what do you do?</p> <p>I’ve created a quick and dirty console app that will host a WCF service, create certificates and assign the certificates to the port the service is being hosted on. This means that each developer can set the console app as one of their start up projects and just hit F5 to compile and debug. This means that you don’t need to redeploy to local IIS every time you make a change and want to debug/test it. <font color="#ff0000" size="2"><em>Note : this is not production ready code, it's a fairly hacky solution that solves the problem of debugging transport security WCF services locally.</em></font></p> <p>A couple of things to note…</p> <ul> <li>Visual Studio will have to be run in admin mode.  <li>I make use of both netsh and makecert applications and have hard coded the possible paths they could be in. This works on my machine but made need some modification if these programs are in different paths on your machine.</li></ul> <p>To use the below code you will need to….</p> <ul> <li>Create a new console application.  <li>Add a reference to your WCF service project.  <li>Copy the .config from your WCF service into the console apps app.config. You can do this in code rather than config files if you’d rather.  <li>If the endpoint doesn't have a base address it will need to be changed in app.config to have one as we’re not using IIS so need to specify our own base address. For example…</li></ul><pre class="brush: xml; gutter: false; toolbar: false;">&lt;endpoint 
   name="ws" 
   address="https://localhost:8123/DebugHost/" 
   binding="ws2007FederationHttpBinding" 
   contract="Services.IService1"/&gt;</pre>
<ul>
<li>Add an appSetting to for the base address we’re hosting on, this will be the same as the address in the endpoint node and should look like this…</li></ul><pre class="brush: xml; gutter: false; toolbar: false;">  &lt;appSettings&gt;
    &lt;add key="baseUrl" value="https://localhost:8123/DebugHost"/&gt;
  &lt;/appSettings&gt;</pre>
<ul></ul>
<p>You should then be able to use the following code in your console app to host the WCF service. You'll need to change references from Service1 to your service.</p><pre class="brush: csharp; gutter: false; toolbar: false;">using System;
using System.Diagnostics;
using System.IO;
using System.Security.Cryptography.X509Certificates;
using System.ServiceModel;
using Services;
using System.Collections.Generic;
using System.Configuration;

namespace ServiceLocalHost
{
    class Program
    {        
        private static readonly Uri baseAddress = new Uri(ConfigurationManager.AppSettings["baseUrl"]);
        private static readonly string certPath = Path.Combine(Directory.GetCurrentDirectory(),"MyDebugCert.cer");
        private static readonly List&lt;string&gt; makeCertLocations = new List&lt;string&gt;
        	{
			Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.ProgramFiles).Replace(" (x86)",""), @"Microsoft SDKs\Windows\v6.0A\Bin\makecert.exe"),
			Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.ProgramFiles), @"Microsoft SDKs\Windows\v6.0A\Bin\makecert.exe"),
			Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.ProgramFiles).Replace(" (x86)",""), @"Microsoft SDKs\Windows\v7.1A\Bin\makecert.exe"),
			Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.ProgramFiles), @"Microsoft SDKs\Windows\v7.1A\Bin\makecert.exe"),
		}; 

        static void Main()
        {
            CreateCertIfDoesntExist();
            BindSslCertToPort();

            using (var host = new ServiceHost(typeof(Service1), baseAddress))
            {
                host.Open();
                Console.WriteLine("Service Running press enter to stop");
                Console.ReadLine();
                host.Close();
            }
        }

        static void CreateCertIfDoesntExist()
        {
            var makeCertPath = "";
            //Check possible paths for makecert
            foreach (var path in makeCertLocations)
                if (File.Exists(path))
                    makeCertPath = path;

            if(makeCertPath == "")
                throw new FileNotFoundException("Could not find makecert");

            if (!File.Exists(certPath))
            {
                var certificateMakerer = new Process();
                certificateMakerer.StartInfo.FileName = makeCertPath;
                certificateMakerer.StartInfo.Arguments = string.Format(@"-sk RootCA -sky signature -pe -n CN=localhost -r -sr LocalMachine -ss Root ""{0}""",certPath);
                certificateMakerer.Start();
                certificateMakerer.WaitForExit();
                certificateMakerer.StartInfo.Arguments = string.Format(@"-sk server -sky exchange -pe -n CN=localhost -ir LocalMachine -is Root -ic ""{0}"" -sr LocalMachine -ss My ""{1}""", Path.GetFileName(certPath),certPath);
                certificateMakerer.Start();
                certificateMakerer.WaitForExit();
            }
        }

        static void BindSslCertToPort()
        {
            var certificate = new X509Certificate2(certPath);

            var removeSslCertCommand = string.Format("http delete sslcert ipport=0.00.0:{0}", baseAddress.Port);
            var removeReservationCommand = string.Format("http delete urlacl url=https://+:{0}/", baseAddress.Port);
            var addReservationCommand = string.Format("http add urlacl url=https://+:{0}/ user=EVERYONE", baseAddress.Port);
            var addSslCertCommand = string.Format("http add sslcert ipport=0.0.0.0:{0} certhash={1} appid={2}", baseAddress.Port, certificate.Thumbprint, Guid.NewGuid());
            var commands = new List&lt;string&gt; { removeSslCertCommand, removeReservationCommand, addReservationCommand, addSslCertCommand };
            
            var bindPortToCertificate = new Process();
            bindPortToCertificate.StartInfo.FileName = Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.SystemX86), "netsh.exe");

            foreach (var cmd in commands)
            {
                bindPortToCertificate.StartInfo.Arguments = cmd;
                bindPortToCertificate.Start();
                bindPortToCertificate.WaitForExit();
            }
        }
    }
}
</pre>