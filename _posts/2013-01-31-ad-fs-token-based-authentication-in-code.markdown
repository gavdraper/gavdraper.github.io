---
layout: post
title: AD FS Token Based Authentication In Code
date: '2013-01-31 11:51:51'
---

<p>I’m writing this post more as documentation for myself as I know I will be repeating this process quite a lot in coming months.</p> <p>I’m currently looking at implementing token based security in .Net clients/WCF backend services. This needs a Domain with STS configured in this case I’m using Active Directory Federated Services. There are plenty of examples of doing this via configuration in WCF/ASP.Net MVC, but in this case I wanted a pure code solution in a console app to give me a better idea of how it works and what happens under the hood.</p> <p>For my starting point I’ve configured 2 VM’s both running Windows Server 2012 with one configured as a domain controller and the other set to be on that domain. If you want details on getting to this position follow this <a href="http://www.gavindraper.co.uk/2012/11/19/sql-availability-groups-vm-sandbox-step-by-step/">blog post</a> up to the failover clustering header.</p> <h2>Installing AD FS</h2> <ol> <li>On the domain controller open the Server Manager  <li>When you get to the screen listing the roles tick “Active Directory Federation Services”<br><img title="Roles" style="border-left-width: 0px; border-right-width: 0px; background-image: none; border-bottom-width: 0px; padding-top: 0px; padding-left: 0px; display: inline; padding-right: 0px; border-top-width: 0px" border="0" alt="Picture of Roles in Wizard" src="/content/images/WPImport/2013/01/image.png" width="358" height="207">  <li>Make sure “Include Management Tools” is ticked and click “Add Features”  <li>When you get here tick the following option<br><img title="Role Services" style="border-left-width: 0px; border-right-width: 0px; background-image: none; border-bottom-width: 0px; padding-top: 0px; padding-left: 0px; display: inline; padding-right: 0px; border-top-width: 0px" border="0" alt="Picture of role services" src="/content/images/WPImport/2013/01/image1.png" width="520" height="267">  <li>Click next through the installer until it’s finished. </li></ol> <h2></h2> <h2>Configuring AD FS</h2> <p>Before we start the AD FS configuration wizard we need an SSL certificate. AD FS by default tries to use the one on the default site in IIS.</p> <h3></h3> <p><u>SSL Certificate</u></p> <ol> <li>Open IIS  <li>Select the server node on the left  <li>Open “Server Certificates” in the features view <br><img title="IIS Server Features" style="border-left-width: 0px; border-right-width: 0px; background-image: none; border-bottom-width: 0px; padding-top: 0px; padding-left: 0px; display: inline; padding-right: 0px; border-top-width: 0px" border="0" alt="Pictur eOf IIS Features View " src="/content/images/WPImport/2013/01/image2.png" width="379" height="306">  <li>Click “Create Self Signed Certificate”  <li>Choose a certificate name and click OK  <li>Double click the certificate you just created  <li>On the details pane click “Copy to file”  <li>Click next  <li>Select “No, do not export the private key” and click next  <li>Leave “DEF encoded binary” selected and click next  <li>Choose a path to save it to and click next then finish (remember this path as you will need it later)  <li>Click the “Default Web Site” node on the left  <li>Click “Bindings” on the right  <li>Click Add  <li>Set the type to https and the SSL certificate to the certificate you just created </li></ol> <p><u>AD FS Wizards</u></p> <ol> <li>From the Start Menu run “AD FS Management”  <li>Then click “AD FS Federation Server Configuration Wizard”  <li>Select “Create a new Federation Service” and click next  <li>Select “Stand-alone federation server” and click next  <li>The certificate we just created should be automatically selected, click next<br><img title="AD FS Certificate" style="border-left-width: 0px; border-right-width: 0px; background-image: none; border-bottom-width: 0px; padding-top: 0px; padding-left: 0px; display: inline; padding-right: 0px; border-top-width: 0px" border="0" alt="Picture of AD FS wizard showing certificate" src="/content/images/WPImport/2013/01/image3.png" width="469" height="262">  <li>Click next and wait for the install to finish then click close </li></ol> <p><u>Relying Party Configuration</u></p> <p>Still in AD FS Management do the following…</p> <ol> <li>Under AD FS/Trust Relationships/Relying Party Trusts click “Add Relying Party Trust…”  <li>Select “Enter data manually” and click next  <li>Choose a display name and click next  <li>Leave “AD FS profile” selected and click next  <li>Leave the certificate blank and click next  <li>Click next without entering a URL  <li>Choose an identifier for this relying party (something like <a href="https://adfstest.domain.net/adfs/services/myparty">https://adfstest.domain.net/adfs/services/myparty</a>) then click add and next  <li>Leave all users checked and click next  <li>Click next, then finish, then close  <li>You should now have the claims wizard. Click “Add Rule”  <li>Select “Send LDAP Attributes as claims” and click next  <li>Choose a rule name (MyClaims will do)  <li>Set the attribute store to Active Directory  <li>Add a couple of mapped claims, anything that has information in Active Directory will do<br><img title="Claims" style="border-left-width: 0px; border-right-width: 0px; background-image: none; border-bottom-width: 0px; padding-top: 0px; padding-left: 0px; display: inline; padding-right: 0px; border-top-width: 0px" border="0" alt="Image of AD FS claims set up" src="/content/images/WPImport/2013/01/image4.png" width="521" height="349">  <li>Click finish then OK </li></ol> <h2>Installing The Client Certificates</h2> <p>Assuming you’ve followed along and created a self signed certificate we now need to put that certificate on the client machine to allow our app to communicate with ADFS. We also needs to install the AD FS signing certificate to the client machine so once we have the token we can decrypt it. *If you’ve used trusted certificates you can ignore these steps.</p> <ol> <li>On the AD FS server open “AD FS Management”  <li>Under Service/Certificates double click the Token-signing certificate  <li>On the details tab click “Copy to file” and save the certificate like you did with the last one.  <li>You should now have 2 saved certificates, copy them both to the other VM (The client VM) </li></ol> <p>On the client VM repeat the following steps for both certificates</p> <ol> <li>Double click the certificate  <li>Click “Install Certificate”  <li>Select “Local Machine” and click next  <li>Select place in following store "and click browse  <li>Select the “Trusted Root Certification Authorities” store and click OK  <li>Click next then finish </li></ol> <h2>Creating a C# Console App To Use AD FS</h2> <p>I used .Net 4.5 and Visual Studio 2012 but I *think* it would be almost the same for .Net3.5 if you’ve installed the Windows Identity Foundation SDK.</p> <ol> <li>In Visual Studio create a new Console App  <li>Add the “Thinktecture.IdentityModel” Nuget package to the project  <li>Change program.cs to look like this 

```language-csharp
using System;
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
            const string relyingPartyId = "https://adfsserver.security.net/MyApp"; //ID of the relying party in AD FS
            const string adfsEndpoint = "https://adfsserver.security.net/adfs/services/trust/13/windowsmixed";
            const string certSubject = "CN=adfsserver.security.net";
            
            //Setup the connection to ADFS
            var factory = new WSTrustChannelFactory(new WindowsWSTrustBinding(SecurityMode.TransportWithMessageCredential), new EndpointAddress(adfsEndpoint))
            {
                TrustVersion = TrustVersion.WSTrust13
            };

            //Setup the request object 
            var rst = new RequestSecurityToken
            {
                RequestType = RequestTypes.Issue,
                KeyType = KeyTypes.Bearer,
                AppliesTo = new EndpointReference(relyingPartyId)
            };

            //Open a connection to ADFS and get a token for the logged in user
            var channel = factory.CreateChannel();
            var genericToken = channel.Issue(rst) as GenericXmlSecurityToken;

            if (genericToken != null)
            {
                //Setup the handlers needed to convert the generic token to a SAML Token
                var tokenHandlers = new SecurityTokenHandlerCollection(new SecurityTokenHandler[] { new SamlSecurityTokenHandler() });
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
                    x509Token.Certificate.SubjectName.Name!=null &amp;&amp;
                    x509Token.Certificate.SubjectName.Name.Contains(_certSubject))
                    return x509Token.Certificate.SubjectName.Name;
                throw new SecurityTokenException("Untrusted issuer.");
            }
        }

    }
}
```
<li>The top 3 lines need to be changed to match your AD FS setup. </li></ol>
<blockquote>
<p>For the relying party id enter the id you choose when creating the relying party.</p>
<p>For the adfsEndpoint in AD FS Management expand endpoints, make sure the line that contains “/trust/13/windowsmixed” is set to enabled and use that path with the server path to get the endpoint.</p>
<p>The cert subject needs the subject of your signing certificate, if you're not sure what this is put a breakpoint in the GetIssuerName method and have a look at what the subject on the certificate object is set to.</p></blockquote>
<p>Assuming everything is set up correctly you should now be able to run the console application and get back your user details from AD FS With its associated claims.</p>
<p><img title="Claims" style="border-top: 0px; border-right: 0px; background-image: none; border-bottom: 0px; padding-top: 0px; padding-left: 0px; border-left: 0px; display: inline; padding-right: 0px" border="0" alt="Image showing claims in a console app" src="/content/images/WPImport/2013/01/image6.png" width="669" height="150"></p>

<p>On a closing point I highly recommend <a href="http://leastprivilege.com/">Dominick Baier’s blog</a> for more on the subject of Windows Identity Foundation and Secure Token Services. He answered a number of my questions on stackoverflow whilst I was working on this and really knows his stuff. </p>