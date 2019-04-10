---
layout: post
title: "Authorization Code Grant Flow with Sitecore"
date: 2019-04-10 12:30:00 +0000
comments: true
categories: [sitecore,owin]
---
Implementing an Authorization Code Grant Flow with Sitecore

### The flow

The Authorization Code is an OAuth 2.0 grant that regular web apps use in order to access an API.

When the value of `response_type` is `code` and `openid` is included in the `scope` request parameter, an ID token is issued from the token endpoint in addition to an access token.


| Endpoint      | Authorization Code | Access Token | ID Token |
|:-------------:|:------------------:|:------------:|:--------:|
| Authorization | Issued             | X            | X        |
| Token         | X                  | Issued       | Issued   |

This is the [diagram](https://medium.com/@darutk/diagrams-of-all-the-openid-connect-flows-6968e3990660) of the 'response_type=code (scope includes openid)' OpenID Connect Flow

![response_type=code (scope includes openid)](https://cdn-images-1.medium.com/max/1600/1*quwFs1fFCvTvLT80e_QHVA.png "response_type=code (scope includes openid)")

### The nuget packages

In order to control Sitecore dependencies, I would use `Microsoft.Owin.Security.OpenIdConnect -Version 3.1.0`, which is aligned in terms of dependencies with the Microsoft.Owin version that Sitecore 9.0.1 is using. I would also use the package `IdentityModel -Version 2.4.0` which requires `Newtonsoft.Json -Version 9.0.1`.

This is how my `packages.config` looks like:

```xml
<?xml version="1.0" encoding="utf-8"?>
<packages>
  <package id="IdentityModel" version="2.4.0" targetFramework="net47" />
  <package id="Microsoft.AspNet.Identity.Core" version="2.2.1" targetFramework="net47" />
  <package id="Microsoft.AspNet.Identity.Owin" version="2.2.1" targetFramework="net47" />
  <package id="Microsoft.AspNet.Mvc" version="5.2.3" targetFramework="net47" />
  <package id="Microsoft.AspNet.Razor" version="3.2.3" targetFramework="net47" />
  <package id="Microsoft.AspNet.WebPages" version="3.2.3" targetFramework="net47" />
  <package id="Microsoft.IdentityModel.Protocol.Extensions" version="1.0.4.403061554" targetFramework="net47" />
  <package id="Microsoft.Owin" version="3.1.0" targetFramework="net47" />
  <package id="Microsoft.Owin.Host.SystemWeb" version="3.1.0" targetFramework="net47" />
  <package id="Microsoft.Owin.Security" version="3.1.0" targetFramework="net47" />
  <package id="Microsoft.Owin.Security.Cookies" version="3.1.0" targetFramework="net47" />
  <package id="Microsoft.Owin.Security.OAuth" version="3.1.0" targetFramework="net47" />
  <package id="Microsoft.Owin.Security.OpenIdConnect" version="3.1.0" targetFramework="net47" />
  <package id="Microsoft.Web.Infrastructure" version="1.0.0.0" targetFramework="net47" />
  <package id="Newtonsoft.Json" version="9.0.1" targetFramework="net47" />
  <package id="Owin" version="1.0" targetFramework="net47" />
  <package id="Sitecore.Kernel.NoReferences" version="9.0.171219" targetFramework="net47" developmentDependency="true" />
  <package id="Sitecore.Owin.Authentication.NoReferences" version="9.0.171219" targetFramework="net47" developmentDependency="true" />
  <package id="Sitecore.Owin.NoReferences" version="9.0.171219" targetFramework="net47" developmentDependency="true" />
  <package id="System.IdentityModel.Tokens.Jwt" version="4.0.4.403061554" targetFramework="net47" />
</packages>
```

### The challenge

According to [this question in SO](https://stackoverflow.com/questions/33661935/owin-middleware-for-openid-connect-code-flow-flow-type-authorizationcode), the `Microsoft.Owin.Security.OpenIdConnect -Version 3.1.0` middleware doesn't support the code flow. When looking into the implementation of the [OpenidConnectAuthenticationHandler.cs](https://github.com/aspnet/AspNetKatana/blob/dev/src/Microsoft.Owin.Security.OpenIdConnect/OpenidConnectAuthenticationHandler.cs) in the method AuthenticateCoreAsync(), when can see the following code:

```c#
// code is only accepted with id_token, in this version, hence check for code is inside this if 
// OpenIdConnect protocol allows a Code to be received without the id_token     
if (string.IsNullOrWhiteSpace(openIdConnectMessage.IdToken))    
{       
    _logger.WriteWarning("The id_token is missing.");       
    return null;    
}        
```

### The solution

We can hook up on the OnMessageReceived event and:

1. Get authorization code 
2. Send it to the token endpoint and get the `id_token` and `access_token`
3. Set the `ProtocolMessage.Code = null` to avoid being checked again on the `OnSecurityTokenValidated` event
4. Pass the `id_token` and `access_token` on the `ProtocolMessage`

```c#
using System;
using System.IdentityModel.Tokens;
using System.Security.Claims;
using System.Threading.Tasks;
using IdentityModel.Client;
using Microsoft.IdentityModel.Protocols;
using Microsoft.Owin.Security.Notifications;
using Microsoft.Owin.Security.OpenIdConnect;
using Owin;
using Sitecore.Configuration;
using Sitecore.Diagnostics;
using Sitecore.Owin.Authentication.Configuration;
using Sitecore.Owin.Authentication.Extensions;
using Sitecore.Owin.Authentication.Pipelines.IdentityProviders;
using Sitecore.Owin.Authentication.Services;

namespace Lusar.Foundation.FederatedLogin.Pipelines.IdentityProviders
{
    public class Auth0Provider : IdentityProvidersProcessor
    {
        private readonly string auth0Domain = Settings.GetSetting("auth0:Domain");
        private readonly string auth0ClientId = Settings.GetSetting("auth0:ClientId");
        private readonly string auth0ClientSecret = Settings.GetSetting("auth0:ClientSecret");
        private readonly string auth0RedirectUri = Settings.GetSetting("auth0:RedirectUri");
        private readonly string auth0PostLogoutRedirectUri = Settings.GetSetting("auth0:PostLogoutRedirectUri");
        private readonly string auth0Scope = Settings.GetSetting("auth0:Scope");
        private readonly string auth0Audience = Settings.GetSetting("auth0:Audience");
        protected IdentityProvider IdentityProvider { get; set; }
        protected override string IdentityProviderName => "Auth0";

        public Auth0Provider(FederatedAuthenticationConfiguration federatedAuthenticationConfiguration)
            : base(federatedAuthenticationConfiguration)
        {
        }

        protected override void ProcessCore(IdentityProvidersArgs args)
        {
            Assert.ArgumentNotNull(args, "args");
            IdentityProvider = GetIdentityProvider();

            // Configure Auth0 authentication
            args.App.UseOpenIdConnectAuthentication(new OpenIdConnectAuthenticationOptions
            {
                Authority = $"https://{auth0Domain}",
                ClientId = auth0ClientId,
                ClientSecret = auth0ClientSecret,
                RedirectUri = auth0RedirectUri,
                PostLogoutRedirectUri = auth0PostLogoutRedirectUri,
                ResponseType = "code",
                Scope = auth0Scope,
                TokenValidationParameters = new TokenValidationParameters
                {
                    NameClaimType = "name"
                },
                Notifications = new OpenIdConnectAuthenticationNotifications
                {
                    AuthenticationFailed = OnAuthenticationFailed,
                    MessageReceived = OnMessageReceived,
                    RedirectToIdentityProvider = OnRedirectToIdentityProvider,
                    SecurityTokenValidated = OnSecurityTokenValidated
                },
                AuthenticationType = IdentityProvider.Name
            });
        }

        private Task OnAuthenticationFailed(AuthenticationFailedNotification<OpenIdConnectMessage, OpenIdConnectAuthenticationOptions> context)
        {
            Log.Error($"FederatedLogin: AuthenticationFailed {context.Exception.Message}", context.Exception, this);
            return Task.CompletedTask;
        }

        private async Task OnMessageReceived(MessageReceivedNotification<OpenIdConnectMessage, OpenIdConnectAuthenticationOptions> context)
        {
            var tokenClient = new TokenClient($"https://{auth0Domain}/oauth/token", auth0ClientId, auth0ClientSecret);
            var tokenResponse = await tokenClient.RequestAuthorizationCodeAsync(context.ProtocolMessage.Code, context.Options.RedirectUri);
            if (tokenResponse.IsError)
            {
                throw new Exception(tokenResponse.Error);
            }

            context.ProtocolMessage.Code = null;
            context.ProtocolMessage.IdToken = tokenResponse.IdentityToken;
            context.ProtocolMessage.AccessToken = tokenResponse.AccessToken;
        }

        private Task OnRedirectToIdentityProvider(RedirectToIdentityProviderNotification<OpenIdConnectMessage, OpenIdConnectAuthenticationOptions> context)
        {
            if (context.ProtocolMessage.RequestType != OpenIdConnectRequestType.LogoutRequest)
            {
                context.ProtocolMessage.SetParameter("audience", auth0Audience);
                return Task.CompletedTask;
            }

            var logoutUri = $"https://{auth0Domain}/v2/logout?client_id={auth0ClientId}";
            var postLogoutUri = context.ProtocolMessage.PostLogoutRedirectUri;
            if (!string.IsNullOrEmpty(postLogoutUri))
            {
                if (postLogoutUri.StartsWith("/"))
                {
                    // transform to absolute
                    var request = context.Request;
                    postLogoutUri = request.Scheme + "://" + request.Host + request.PathBase + postLogoutUri;
                }

                logoutUri += $"&returnTo={ Uri.EscapeDataString(postLogoutUri)}";
            }

            context.Response.Redirect(logoutUri);
            context.HandleResponse();
            return Task.CompletedTask;
        }

        private Task OnSecurityTokenValidated(SecurityTokenValidatedNotification<OpenIdConnectMessage, OpenIdConnectAuthenticationOptions> context)
        {
            var identity = context.AuthenticationTicket.Identity;
            identity.AddClaim(new Claim("id_token", context.ProtocolMessage.IdToken));
            identity.AddClaim(new Claim("access_token", context.ProtocolMessage.AccessToken));
            var transformationContext = new TransformationContext(FederatedAuthenticationConfiguration, IdentityProvider);
            identity.ApplyClaimsTransformations(transformationContext);
            return Task.CompletedTask;
        }
    }
}
```

Here is how my configuration file looks like:

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration xmlns:patch="http://www.sitecore.net/xmlconfig/" xmlns:role="http://www.sitecore.net/xmlconfig/role/">
  <sitecore role:require="Standalone or ContentDelivery or ContentManagement">
    <settings>
      <setting name="auth0:Domain" value="yourdomain.eu.auth0.com" />
      <setting name="auth0:ClientId" value="yourclientid" />
      <setting name="auth0:ClientSecret" value="yourclientsecret" />
      <setting name="auth0:RedirectUri" value="https://yourdomain.com/identity/externallogincallback" />
      <setting name="auth0:PostLogoutRedirectUri" value="https://yourdomain.com" />
      <setting name="auth0:Scope" value="openid" />
      <setting name="auth0:Audience" value="youraudience" />
    </settings>
    <pipelines>
      <owin.identityProviders>
        <processor type="Lusar.Foundation.FederatedLogin.Pipelines.IdentityProviders.Auth0Provider, Lusar.Foundation.FederatedLogin" resolve="true" />
      </owin.identityProviders>
    </pipelines>
    <federatedAuthentication type="Sitecore.Owin.Authentication.Configuration.FederatedAuthenticationConfiguration, Sitecore.Owin.Authentication">
      <identityProvidersPerSites hint="list:AddIdentityProvidersPerSites">
        <mapEntry name="public" type="Sitecore.Owin.Authentication.Collections.IdentityProvidersPerSitesMapEntry, Sitecore.Owin.Authentication">
          <sites hint="list">
            <site>website</site>
          </sites>
          <identityProviders hint="list:AddIdentityProvider">
            <identityProvider ref="federatedAuthentication/identityProviders/identityProvider[@id='Auth0']" />
          </identityProviders>
          <externalUserBuilder type="Sitecore.Owin.Authentication.Services.DefaultExternalUserBuilder, Sitecore.Owin.Authentication">
            <param desc="isPersistentUser">false</param>
          </externalUserBuilder>
        </mapEntry>
      </identityProvidersPerSites>
      <identityProviders hint="list:AddIdentityProvider">
        <identityProvider id="Auth0" type="Sitecore.Owin.Authentication.Configuration.DefaultIdentityProvider, Sitecore.Owin.Authentication">
          <param desc="name">$(id)</param>
          <param desc="domainManager" type="Sitecore.Abstractions.BaseDomainManager" resolve="true" />
          <caption>Log in with Auth0</caption>
          <icon>sitecore/shell/themes/standard/Custom/24x24/profile.png</icon>
          <domain>extranet</domain>
          <transformations hint="list:AddTransformation">
            <transformation name="set idp claim" ref="federatedAuthentication/sharedTransformations/setIdpClaim" />
            <transformation name="devRole" type="Sitecore.Owin.Authentication.Services.DefaultTransformation, Sitecore.Owin.Authentication">
              <sources hint="raw:AddSource">
                <claim name="idp" value="Auth0" />
              </sources>
              <targets hint="raw:AddTarget">
                <claim name="http://schemas.microsoft.com/ws/2008/06/identity/claims/role" value="Sitecore\Developer" />
              </targets>
              <keepSource>true</keepSource>
            </transformation>
          </transformations>
        </identityProvider>
      </identityProviders>
    </federatedAuthentication>
  </sitecore>
</configuration>
```
