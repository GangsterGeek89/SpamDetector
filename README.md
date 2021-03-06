# SpamDetector
| Server | Status |
| ------ | ------ |
| AppVeyor | [![Build status](https://ci.appveyor.com/api/projects/status/x55tk8vtffvt354b?svg=true)](https://ci.appveyor.com/project/jensbrak/spamdetector) |
| NuGet | [![Nuget](https://img.shields.io/nuget/v/Zon3.SpamDetector)](https://www.nuget.org/packages/Zon3.SpamDetector) |

Basic Spam Detection Module for PiranhaCMS using Akismet API

# About
A basic module that use Akismet to validate comments submitted to a Piranha Page or Post. This is done by providing Akismet with details from the comment and the site using the module. If the comment is considered to be spam by Akismet, it will not be approved by Piranha. Comments in Piranha Manager are either approved or pending which means (when the module is active): all comments marked approved has been verified as non-spam by the module and all comments marked as pending is spam (and can be removed).

_Please note: using the module as a Comment Validation Hook will override the Manager Config setting for Approve Comments. This means that regardless of that setting a comment will be approved (ie published) if it is not considered spam. If considered spam, the comment will not be approved (ie pending)._

# Dependencies
* `Microsoft.Extensions.Http`
* `Piranha` version 9

_Note: Piranha version 10 is planned to have hooks redesigned, forcing this module to be redesigned too when version 10 is released. Take this into account if using this module. (See referenced issue under Further reading section)._

# Prerequisites
* A solution or project using PiranhaCMS (see https://piranhacms.org/)
* An Akismet Developer API key (see https://akismet.com/development/api)

# Further reading
* Piranha Modules: 
	* https://piranhacms.org/docs/extensions/modules
	* https://github.com/PiranhaCMS/piranha.modules
* Piranha Hooks: 
	* https://piranhacms.org/docs/application/hooks
* Akismet API documentation:
    * https://akismet.com/development/api/#detailed-docs
    * https://akismet.com/development/api/#comment-check
* Related Piranha issues:
    * Redesign of Hooks: https://github.com/PiranhaCMS/piranha.core/issues/1236

# Demo
Please go ahead and try it out by posting a comment with a greeting on my blog. It runs PiranhaCMS with this module active. If your comment is directly visible it's been classified by akismet as non-spam and approved by the SpamDetector module as a valid comment. If not, stop spamming! (Or report a bug to me ;) )

My PiranhaCMS demo site: https://zon3.se 

# Usage
See Code snippets below for pointers or check the sample files `Startup.cs` and `appsettings.json` provided in the `Example` folder:

https://github.com/jensbrak/SpamDetector/blob/master/Examples/ 

## Code
1. Add a reference to `Zon3.SpamDetector` in your Piranha project
1. In `Startup.cs` register reqired services and hooks: 
    1. Register `SpamDetector` service using `SpamDetectorOptions` for settings
    1. Register `SpamDetector` as a Comment hook after Piranha has been)

## Settings
The class `SpamDetectorOptions` defines the options that can be set for SpamDetector. The options available are:

* `Enabled` (optional, default: `true`): If false, module will not send requests and leave comments unreviewed
* `SpamApiUrl` (Required): the complete URL to the API to use for spam detection
* `SiteUrl` (optional): the base URL of the site the comments are posted on. See Note 1. 
* `SiteLanguage` (optional, default: `"en-US"`): The language of the site the comments are posted on 
* `SiteEncoding` (optional, default: `"UTF8"`): The encoding of the site the comments are posted on
* `UserRole` (optional, default: `"guest"`): The name of the user role comments are posted as
* `IsTest` (optional, default: `true`): If true, all requests are marked 'test'. See Note 2.

_Note 1: While SpamDetector will run without `SiteUrl` defined, it is highly recommended to set it to get reliable results. However, while testing it shouldn't matter (see `IsTest`)._

_Note 2: While optional, the value of `IsTest` will have to be changed to `false` eventually. The reason for having to do this explicitly is to prevent undesired live requests to the API while setting up and testing._


# Code snippets
## Registering services
```csharp
        public void ConfigureServices(IServiceCollection services)
        {
			services.Configure<SpamDetectorOptions>(_config.GetSection(SpamDetectorOptions.Identifier));

            services.AddPiranha(options =>
            {
                // (Standard Piranha setup code removed for brevity)

                options.UseSpamDetector<AkismetSpamDetector>(o => 
                    _config.GetSection(SpamDetectorOptions.Identifier).Bind(o));            

```

## Adding Comment Hook
```csharp
        public void Configure(IApplicationBuilder app, IWebHostEnvironment env, IApi api)
        {
            if (env.IsDevelopment())
            {
                app.UseDeveloperExceptionPage();
            }

            // (Standard Piranha configuration code removed for brevity)

            App.Hooks.Comments.RegisterOnValidate(c =>
            {
                using var serviceScope = app.ApplicationServices.GetRequiredService<IServiceScopeFactory>().CreateScope();
                ISpamDetector commentModerator = serviceScope.ServiceProvider.GetRequiredService<ISpamDetector>();
                c.IsApproved = commentModerator.ReviewAsync(c).Result.Approved;
            });
        }
```

## Sample appsettings.json section
A minimal section for SpamDetecor would be:

```json
    "SpamDetector": {
        "SpamApiUrl": "https://<yourakismetapikeyhere>.rest.akismet.com/1.1/comment-check",
        "SiteUrl": "https://<yourawesomepiranhasitehere>",
        "IsTest": true
    }
```

_Note: Only set `IsTest` to false when confident everything is properly setup and working_
