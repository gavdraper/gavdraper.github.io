---
layout: post
title: Configuring/Authenticating with Identity Server in C#
date: '2013-02-06 16:09:56'
---

<p>In my previous post I walked through configuring ADFS and writing a small C# console app to authenticate with it and get back a ClaimsPrincipal object. This is fine when your system only needs to authenticate domain users but lets say we have external users we want to be able to authenticate using a username and password. This is where using a Secure Token Service for your authentication really shines as you can pretty much swap them in/out and add more without having to rewrite your security logic.</p> <p>In this post I’m going to alter my previous ADFS project to swap out ADFS for Identity Server. I’ll first walk through configuring Identity Server then modify the console app to use this STS rather than ADFS. To follow the walkthrough you may want to first read my <a href="http://www.gavindraper.co.uk/2013/01/31/ad-fs-token-based-authentication-in-code/">previous post</a> on ADFS as I will be building on that. In a future blog post I’ll cover building a resource STS that can act as a gateway to support multiple STS’s without having to make any code changes but for now we’ll just look at Identity Server.</p> <p>First things first you’re going to need a copy of Identity Server, head over to their <a href="http://thinktecture.github.com/Thinktecture.IdentityServer.v2/">github repository</a> and download the latest version (2.1 at the time of this blog post).</p> <h2></h2> <h3>Installing &amp; Configuring Identity Server</h3> <ol> <li>Unzip IdentityServer to a destination of your choice  <li>In the IdentityServer folder right click App_Data and go to properties  <li>On the Security tab click edit  <li>Click Add  <li>Enter “Network Service” and click OK  <li>Tick the allow box for modify to give the service modify permissions on the app_data directory<br><img title="image" style="border-left-width: 0px; border-right-width: 0px; background-image: none; border-bottom-width: 0px; padding-top: 0px; padding-left: 0px; display: inline; padding-right: 0px; border-top-width: 0px" border="0" alt="image" src="/content/images/WPImport/2013/02/image.png" width="370" height="452">  <li>Click OK on both open modal windows  <li>Open IIS  <li>Under Application Pools, create a new Application Pool called “IdentityServer” using .Net 4.0<br><img title="image" style="border-left-width: 0px; border-right-width: 0px; background-image: none; border-bottom-width: 0px; padding-top: 0px; padding-left: 0px; display: inline; padding-right: 0px; border-top-width: 0px" border="0" alt="image" src="/content/images/WPImport/2013/02/image1.png" width="314" height="290">  <li>Highlight the new Application Pool and click “Advanced Settings”  <li>Set the Identity property to “NetworkService”<br><img title="image" style="border-left-width: 0px; border-right-width: 0px; background-image: none; border-bottom-width: 0px; padding-top: 0px; padding-left: 0px; display: inline; padding-right: 0px; border-top-width: 0px" border="0" alt="image" src="/content/images/WPImport/2013/02/image2.png" width="457" height="201">  <li>Right click the Default Website and click Add Application  <li>Enter "”IdentityServer” into the alias  <li>Set the Application Pool to “IdentityServer”  <li>In physical path point it to where you unzipped Identity Server to<br><img title="image" style="border-left-width: 0px; border-right-width: 0px; background-image: none; border-bottom-width: 0px; padding-top: 0px; padding-left: 0px; display: inline; padding-right: 0px; border-top-width: 0px" border="0" alt="image" src="/content/images/WPImport/2013/02/image3.png" width="532" height="394">  <li>Open a web browser and navigate to <a href="https://localhost/identityserver">https://localhost/identityserver</a>  <li>Enter a site name of your choice  <li>Choose a unique Issuer URI  <li>Select your SSL certificate  <li>Tick Create default roles and admin, then choose a username/password and click save  <li>Click sign in and use your new admin credentials  <li>Click administration  <li>Under users add a new user and make sure you tick the IdentityServerUsers role<br><img title="image" style="border-left-width: 0px; border-right-width: 0px; background-image: none; border-bottom-width: 0px; padding-top: 0px; padding-left: 0px; display: inline; padding-right: 0px; border-top-width: 0px" border="0" alt="image" src="/content/images/WPImport/2013/02/image4.png" width="419" height="157">  <li>Click protocols and make sure WS-Trust is ticked  <li>Click “Relying Parties &amp; Resources” and click new  <li>Tick Enabled, enter a display name, enter an identifier URI in “Realm/Scope Name” then click save. You may want to use the same URI for Realm as you gave for your ADFS relying party identifier but that’s your call<br><img title="image" style="border-left-width: 0px; border-right-width: 0px; background-image: none; border-bottom-width: 0px; padding-top: 0px; padding-left: 0px; display: inline; padding-right: 0px; border-top-width: 0px" border="0" alt="image" src="/content/images/WPImport/2013/02/image5.png" width="524" height="80"></li></ol> <p>Identity Server is now configured to authenticate users over the WS-Trust endpoint. This will be &lt;IdentityServerUri&gt;/issue/wstrust/mixed/username.</p> <p>The following changes need to be made to our ADFS console app to get it to authenticate with Identity Server.</p> <ol> <li>Change the relyingPartyId to your Identity Server relying party id (Only if it’s different to the one you used for ADFS)  <li>Change adfsEndpoint to point at the Identity Server ws-trust endpoint  <li>Change the certSubject to the subject of your Identity Server certificate. Unlike ADFS, by default Identity Server uses the certificate the website is running on to sign the token, rather than using a separate signing certificate.  <li>Change the token handler collection to include an instance of Saml2SecurityTokenHandler as Identity Server uses SAML2  <li>Lastly you need to change the the binding from WindowsWSTrustBinding to UserNameWSTrustBinding and set the username/password you are authenticating.</li></ol> <p>Below is the code with these changes made….</p><pre class="brush: csharp; gutter: false; toolbar: false;">using System;
using System.IO;
using System.IdentityModel.Protocols.WSTrust;
using System.IdentityModel.Tokens;
using System.Linq;
using System.Security.Claims;
using System.ServiceModel;
using System.ServiceModel.Security;
using System.Xml;
using Thinktecture.IdentityModel.WSTrust;

namespace ConsoleApplication1
{
    class Program
    {
        static void Main()
        {
            const string relyingPartyId = "https://adfsserver.security.net/MyApp"; //ID of the relying party specified in Identity Server
            const string identityServerEndpoint = "https://adfsserver.security.net/IdentityServer/issue/wstrust/mixed/username";
            const string certSubject = "CN=adfsserver.security.net";

            //Setup the connection to Identity Server
            var factory = new WSTrustChannelFactory(new UserNameWSTrustBinding(SecurityMode.TransportWithMessageCredential), new EndpointAddress(identityServerEndpoint))
            {
                TrustVersion = TrustVersion.WSTrust13
            };
            factory.Credentials.UserName.UserName = "myuser";
            factory.Credentials.UserName.Password = "mypassword";

            //Setup the request object 
            var rst = new RequestSecurityToken
            {
                RequestType = RequestTypes.Issue,
                KeyType = KeyTypes.Bearer,
                AppliesTo = new EndpointReference(relyingPartyId)
            };

            //Open a connection to Identity Server and get a token for the logged in user
            var channel = factory.CreateChannel();
            var genericToken = channel.Issue(rst) as GenericXmlSecurityToken;
            
            if (genericToken != null)
            {
                //Setup the handlers needed to convert the generic token to a SAML Token
                var tokenHandlers = new SecurityTokenHandlerCollection(new SecurityTokenHandler[] { new Saml2SecurityTokenHandler() });
                tokenHandlers.Configuration.AudienceRestriction = new AudienceRestriction();
                tokenHandlers.Configuration.AudienceRestriction.AllowedAudienceUris.Add(new Uri(relyingPartyId));

                var trusted = new TrustedIssuerNameRegistry(certSubject);
                tokenHandlers.Configuration.IssuerNameRegistry = trusted;

                //convert the generic security token to a saml token
                var samlToken = tokenHandlers.ReadToken(new XmlTextReader(new StringReader(genericToken.TokenXml.OuterXml)));

                //convert the saml token to a claims principal
                var claimsPrincipal = new ClaimsPrincipal(tokenHandlers.ValidateToken(samlToken).First());

                //Display token information
                Console.WriteLine("Name : " + claimsPrincipal.Identity.Name);
                Console.WriteLine("Auth Type : " + claimsPrincipal.Identity.AuthenticationType);
                Console.WriteLine("Is Authed : " + claimsPrincipal.Identity.IsAuthenticated);
                foreach (var c in claimsPrincipal.Claims)
                    Console.WriteLine(c.Type + " / " + c.Value);
                Console.ReadLine();
            }
        }

        //The token handler calls this to check the token is from a trusted issuer before converting it to a claims principal
        //In this case I authenticate this by checking the certificate name used to sign the token
        public class TrustedIssuerNameRegistry : IssuerNameRegistry
        {
            private string _certSubject;

            public TrustedIssuerNameRegistry(string certSubject)
            {
                _certSubject = certSubject;
            }

            public override string GetIssuerName(SecurityToken securityToken)
            {
                var x509Token = securityToken as X509SecurityToken;
                if (x509Token != null &amp;&amp;
                    x509Token.Certificate.SubjectName.Name != null &amp;&amp;
                    x509Token.Certificate.SubjectName.Name.Contains(_certSubject))
                    return x509Token.Certificate.SubjectName.Name;
                throw new SecurityTokenException("Untrusted issuer.");
            }
        }
    }
}</pre>
<p>After seeing both these STS’s in action the next step is to combine both of them in a way they can be used in the same system for different authentication types. I’ll cover this in a future blog post…..</p>