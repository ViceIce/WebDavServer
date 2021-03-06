﻿# idunno.Authentication

Now updated for ASP.NET Core 2.0. Really though? You're going to update to version 2.0 and still use Basic Auth? You're a horrible person, you know that, right?

This project contains an implementation of [Basic Authentication](https://tools.ietf.org/html/rfc1945#section-11) for ASP.NET Core. 

It is meant as a demonstration of how to write authentication middleware and **not** as something you would seriously consider using.

## How do I use this?

First acquire an HTTPS certificate (see Notes below). Apply it to your website. Remember to renew it in two years.

Remember this is meant as a demonstration on how to write middleware. Yes, this could be easier, 
I could put a package on nuget, but **it's not meant to be for use in production**.

In your web application add a reference to the package, then in the `ConfigureServices` method in `startup.cs` call
`app.AddAuthentication(BasicAuthenticationDefaults.AuthenticationScheme).UseBasicAuthentication(...);` with your options, 
providing a delegate for `OnValidateCredentials` to validate any user name and password sent with requests and turn that information 
into an `ClaimsPrincipal`, set it on the `context.Principal` property and call `context.Success()`.

If you change your scheme name in the options for the basic authentication handler you need to change the scheme name in 
`AddAuthentication()` to ensure it's used on every request which ends in an endpoint that requires authorization.

You should also add `app.UseAuthentication();` in the `Configure` method, otherwise nothing will ever get called.

You can also specify the Realm used to isolate areas of a web site from one another.

For example;

```c#
public void ConfigureServices(IServiceCollection services)
{
    services.AddAuthentication(BasicAuthenticationDefaults.AuthenticationScheme)
            .AddBasic(options =>
            {
                options.Realm = "idunno";
                options.Events = new BasicAuthenticationEvents
                {
                    OnValidateCredentials = context =>
                    {
                        if (context.Username == context.Password)
                        {
                            var claims = new[]
                            {
                                new Claim(
                                    ClaimTypes.NameIdentifier, 
                                    context.Username, 
                                    ClaimValueTypes.String, 
                                    context.Options.ClaimsIssuer),
                                new Claim(
                                    ClaimTypes.Name, 
                                    context.Username, 
                                    ClaimValueTypes.String, 
                                    context.Options.ClaimsIssuer)
                            };

                            context.Principal = new ClaimsPrincipal(
                                new ClaimsIdentity(claims, context.Scheme.Name));
                            context.Success();
                        }

                        return Task.CompletedTask;
                    }
                };
            });
    
    // All the other service configuration.
}

public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
    app.UseAuthentication();

    // All the other app configuration.
}
```

In the sample you can see that the delegate checks if the user name and password are identical. If they
are then it will consider that a valid login, create set of claims about the user, using the `ClaimsIssuer` from the handler options, 
then create an `ClaimsPrincipal` from those claims, using the `SchemeName` from the handler options, then finally call `context.Success();`
to show there's been a successful authentication.

Of course you'd never implement such a simple validator would you? No? Good. Have a cookie.

The handler will throw an exception if wired up in a site not running on HTTPS (thanks to [@alexwiese](https://github.com/alexwiese) for the original idea and PR)
and will refuse to respond to the challenge flow which ends up prompting the browser to ask for a username and password. 
You can override this if you're a horrible person by setting `AllowInsecureProtocol` to `true` in the handler options. 
If you do this you deserve everything you get. If you're using a non-interactive client, and are sending a username and password to a 
server over HTTP the handler will not throw and will process the authentication header because frankly it's too late, you've sent 
everything in plain text, what's the point of trying to secure anything?

## How do I use this in production?

Seriously? I'd never recommend you use basic authentication in production, but if you must here are some ideas on how to harden it. 

1. In your `OnValidateCredentials` implementation keep a count of failed login attempts, and the IP addresses they come from.
2. Lock out accounts after X failed login attempts, where X is a count you feel is reasonable for your situation.
3. Implement the lock out so it unlocks after Y minutes. In case of repeated attacks increase Y.
4. Be careful when locking out your administration accounts. Have at least one admin account that is not exposed via basic auth, so an attacker cannot lock you out of your site just by sending an incorrect password.
5. Throttle attempts from an IP address, especially one which sends lots of incorrect passwords. Considering dropping/banning attempts from an IP address that appears to be under the control of an attacker. Only you can decide what this means, what consitutes legimate traffic varies from application to application.
6. Always use HTTPS. Redirect all HTTP traffic to HTTPS using `[RequireHttps]`. You can apply this to all of your site via a filter;

    ```c#
    services.Configure<MvcOptions>(options =>
    {
        options.Filters.Add(new RequireHttpsAttribute());
    }
    ```
7. Implement HSTS. 
8. Reconsider your life choices, and look at using OAuth2 or OpenIDConnect instead.

## What about older versions of ASP.NET Core?

Older versions are available in the appropriate branch.

| ASP.NET Core MVC Version | Branch                                                           |
|--------------------------|------------------------------------------------------------------|
| 1.1                      | [rel/1.1.1](https://github.com/blowdart/idunno.Authentication/tree/rel/1.1.1) |
| 1.0                      | [rel/1.0.0](https://github.com/blowdart/idunno.Authentication/tree/rel/1.0.0) |

## Notes

Basic Authentication sends credentials unencrypted. You should only use it over [HTTPS](https://en.wikipedia.org/wiki/HTTPS). 

It may also have performance impacts, credentials are sent and validated with every request. As you should not be storing passwords in clear text your validation procedure will have to hash and compare values
with every request, or cache results of previous hashes (which could lead to data leakage). 

Remember that hash comparisons should be time consistent to avoid [timing attacks](https://en.wikipedia.org/wiki/Timing_attack).
