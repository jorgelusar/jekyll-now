---
layout: post
title: "WsFed Provider for Sitecore"
date: 2019-03-10 16:00:00 +0000
comments: true
categories: [sitecore,owin]
---
A simple WsFed provider for Sitecore

### What is WS-Federation?

[WS-Federation](https://auth0.com/docs/protocols/ws-fed) (which is short for Web Services Federation) is a protocol that can be used to negotiate the issuance of a token. You can use this protocol for your applications (such as a Windows Identity Foundation-based app) and for identity providers (such as Active Directory Federation Services or Azure AppFabric Access Control Service).

There is an existing blog post on [how to add support for Federated Authentication and claims to Sitecore using OWIN](http://blog.baslijten.com/how-to-add-federated-authentication-with-sitecore-and-owin/) for Sitecore 8. This is just a follow up on Sitecore 9.

### How to implement it with Sitecore

We need to reference the `Microsoft.Owin.Security.WsFederation version 3.1.0` package so we can use the WsFederation Authentication, and then implement our provider like this:

```c#
using Microsoft.IdentityModel.Protocols;
using Microsoft.Owin.Security.Notifications;
using Microsoft.Owin.Security.WsFederation;
using Owin;
using Sitecore.Diagnostics;
using Sitecore.Owin.Authentication.Configuration;
using Sitecore.Owin.Authentication.Extensions;
using Sitecore.Owin.Authentication.Pipelines.IdentityProviders;
using Sitecore.Owin.Authentication.Services;
using System.IdentityModel.Tokens;
using System.Threading.Tasks;
using System.Web;
using Sitecore.Configuration;

namespace Lusar.Foundation.FederatedLogin.Pipelines.IdentityProviders
{
    public class WsFedIdentityProvider : IdentityProvidersProcessor
    {
        private readonly string metadataAddress = Settings.GetSetting("wsfed:MetadataAddress");
        private readonly string wtrealm = Settings.GetSetting("wsfed:Wtrealm");
        private readonly string wreply = Settings.GetSetting("wsfed:Wreply");
        private readonly string signOutWreply = Settings.GetSetting("wsfed:SignOutWreply");
        private readonly string audiences = Settings.GetSetting("wsfed:Audiences");
        protected IdentityProvider IdentityProvider { get; set; }
        protected override string IdentityProviderName => "WsFed";

        public WsFedIdentityProvider(FederatedAuthenticationConfiguration federatedAuthenticationConfiguration)
            : base(federatedAuthenticationConfiguration)
        {
        }

        protected override void ProcessCore(IdentityProvidersArgs args)
        {
            Assert.ArgumentNotNull(args, "args");
            IdentityProvider = GetIdentityProvider();
            args.App.UseWsFederationAuthentication(
                new WsFederationAuthenticationOptions
                {
                    Wtrealm = wtrealm,
                    MetadataAddress = metadataAddress,
                    Wreply = wreply,
                    SignOutWreply = signOutWreply,
                    Notifications = new WsFederationAuthenticationNotifications
                    {
                        SecurityTokenValidated = OnSecurityTokenValidated,
                        AuthenticationFailed = OnAuthenticationFailed,
                        RedirectToIdentityProvider = OnRedirectToIdentityProvider
                    },
                    TokenValidationParameters = new TokenValidationParameters
                    {
                        ValidAudiences = string.IsNullOrWhiteSpace(audiences) ? new string[] { } : audiences.Split(';')
                    },
                    AuthenticationType = IdentityProvider.Name
                });
        }

        private Task OnSecurityTokenValidated(SecurityTokenValidatedNotification<WsFederationMessage, WsFederationAuthenticationOptions> context)
        {
            var identity = context.AuthenticationTicket.Identity;
            var transformationContext = new TransformationContext(FederatedAuthenticationConfiguration, IdentityProvider);
            identity.ApplyClaimsTransformations(transformationContext);
            return Task.CompletedTask;
        }

        private Task OnAuthenticationFailed(AuthenticationFailedNotification<WsFederationMessage, WsFederationAuthenticationOptions> context)
        {
            Log.Error($"FederatedLogin: AuthenticationFailed {context.Exception.Message}", context.Exception, this);
            return Task.CompletedTask;
        }

        private Task OnRedirectToIdentityProvider(RedirectToIdentityProviderNotification<WsFederationMessage, WsFederationAuthenticationOptions> context)
        {
            if (!context.ProtocolMessage.IsSignOutMessage)
            {
                return Task.CompletedTask;
            }

            var query = context.Request.Query;
            var redirectUrl = query.Get("redirectUrl");
            var wreply = string.Empty;
            if (string.IsNullOrWhiteSpace(redirectUrl) || !context.ProtocolMessage.Parameters.TryGetValue("wreply", out wreply))
            {
                return Task.CompletedTask;
            }

            var signOutUrl = $"{signOutWreply}?redirectUrl={HttpUtility.UrlEncode(redirectUrl)}";
            context.ProtocolMessage.Parameters["wreply"] = signOutUrl;
            return Task.CompletedTask;
        }
    }
}
```

And set the new provider in a configuration file:

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration xmlns:patch="http://www.sitecore.net/xmlconfig/" xmlns:role="http://www.sitecore.net/xmlconfig/role/">
  <sitecore role:require="Standalone or ContentDelivery or ContentManagement">
    <settings>
      <setting name="wsfed:MetadataAddress" value="https://yourmetadateaddress/federationmetadata/2007-06/federationmetadata.xml" />
      <setting name="wsfed:Wtrealm" value="yourrealm" />
      <setting name="wsfed:Wreply" value="https://yourdomain.com" />
      <setting name="wsfed:SignOutWreply" value="https://yourdomain.com/yoursignoutreply" />
      <setting name="wsfed:Audiences" value="youraudience" />
    </settings>
    <pipelines>
      <owin.identityProviders>
        <processor type="Lusar.Foundation.FederatedLogin.Pipelines.IdentityProviders.WsFedIdentityProvider, Lusar.Foundation.FederatedLogin" resolve="true" />
      </owin.identityProviders>
    </pipelines>
    <federatedAuthentication type="Sitecore.Owin.Authentication.Configuration.FederatedAuthenticationConfiguration, Sitecore.Owin.Authentication">
      <identityProvidersPerSites hint="list:AddIdentityProvidersPerSites">
        <mapEntry name="public" type="Sitecore.Owin.Authentication.Collections.IdentityProvidersPerSitesMapEntry, Sitecore.Owin.Authentication">
          <sites hint="list">
            <site>website</site>
          </sites>
          <identityProviders hint="list:AddIdentityProvider">
            <identityProvider ref="federatedAuthentication/identityProviders/identityProvider[@id='WsFed']" />
          </identityProviders>
          <externalUserBuilder type="Sitecore.Owin.Authentication.Services.DefaultExternalUserBuilder, Sitecore.Owin.Authentication">
            <param desc="isPersistentUser">false</param>
          </externalUserBuilder>
        </mapEntry>
      </identityProvidersPerSites>
      <identityProviders hint="list:AddIdentityProvider">
        <identityProvider id="WsFed" type="Sitecore.Owin.Authentication.Configuration.DefaultIdentityProvider, Sitecore.Owin.Authentication">
          <param desc="name">$(id)</param>
          <param desc="domainManager" type="Sitecore.Abstractions.BaseDomainManager" resolve="true" />
          <caption>Log in with WsFed</caption>
          <icon>sitecore/shell/themes/standard/Custom/24x24/profile.png</icon>
          <domain>extranet</domain>
          <transformations hint="list:AddTransformation">
            <transformation name="set idp claim" ref="federatedAuthentication/sharedTransformations/setIdpClaim" />
          </transformations>
        </identityProvider>
      </identityProviders>
    </federatedAuthentication>
  </sitecore>
</configuration>
```
