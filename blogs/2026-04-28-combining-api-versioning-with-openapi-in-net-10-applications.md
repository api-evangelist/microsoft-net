---
title: "Combining API versioning with OpenAPI in .NET 10 applications"
url: "https://devblogs.microsoft.com/dotnet/api-versioning-in-dotnet-10-applications/"
date: "Tue, 28 Apr 2026 17:00:00 +0000"
author: "Sander ten Brinke"
feed_url: "https://devblogs.microsoft.com/dotnet/feed/"
---
<blockquote><p>This is a guest blog from <a href="https://stenbrinke.nl">Sander ten Brinke</a>, a Microsoft MVP and Senior Software Engineer, with a passion for building scalable and maintainable applications.</p></blockquote>
<p>A lot has changed for ASP.NET Core when it comes to building APIs over the last couple of years. The introduction of Minimal APIs, alongside controllers, has made it easier than ever to get started with building APIs, with .NET 10&#8217;s support for <a href="https://learn.microsoft.com/aspnet/core/release-notes/aspnetcore-10.0?view=aspnetcore-10.0#validation-support-in-minimal-apis">built-in request validation</a> making it an even stronger contender.</p>
<p>Even though building APIs has become easier, one aspect that remains crucial is API versioning. Proper versioning ensures that your API can evolve without breaking existing clients. API versioning has always been supported thanks to libraries like <a href="https://github.com/dotnet/aspnet-api-versioning?tab=readme-ov-file">Asp.Versioning</a>. But with the <a href="https://devblogs.microsoft.com/dotnet/dotnet9-openapi/">release</a> of <a href="https://www.nuget.org/packages/Microsoft.AspNetCore.OpenApi">Microsoft.AspNetCore.OpenApi</a>, Microsoft&#8217;s own OpenAPI library for ASP.NET Core, implementing versioning has changed — especially if you want an officially supported approach.</p>
<p>Microsoft&#8217;s package sets up OpenAPI with versioning in mind (due to the URL being <code>/openapi/v1.json</code> by default), but ASP.NET Core doesn&#8217;t come with extensive built-in API versioning support. Since the release of <code>Microsoft.AspNetCore.OpenApi</code> in .NET 9, there have been a lot of questions online about how to integrate versioning without having to write a lot of custom code, duplicate OpenAPI and versioning definitions, and more.</p>
<p>In this post, we&#8217;ll walk through how to implement API versioning in .NET 10 applications — covering both controllers and Minimal APIs — while keeping your OpenAPI documentation accurate and up to date for each version. We&#8217;ll take the officially supported approach, keeping duplicate code and configuration to a minimum.</p>
<p>We&#8217;ll start with implementing API versioning without OpenAPI, and then we&#8217;ll integrate OpenAPI into our versioned API setup, showing how to generate separate OpenAPI documents for each API version. Finally, we&#8217;ll add <a href="https://github.com/domaindrivendev/Swashbuckle.AspNetCore">SwaggerUI</a> support using <code>Swashbuckle.AspNetCore.SwaggerUI</code> and <a href="https://github.com/scalar/scalar">Scalar</a> support with <code>Scalar.AspNetCore</code> to visualize our versioned API documentation and discuss how to maintain it as your API evolves. By presenting API versioning in a step-based approach, it becomes clear what code changes each step requires.</p>
<p>To do this, we&#8217;ll use the <strong>brand new</strong> <code>Asp.Versioning</code> <em>v10</em> package, also known as ASP.NET API Versioning, which is the first version to officially support both .NET 10 and the new built-in OpenAPI support, making the integration cleaner and simpler than ever before.</p>
<h2>The importance of API versioning</h2>
<p>But first, let&#8217;s make sure we understand the importance of API versioning. If you already understand API versioning, you can skip ahead to the next sections.</p>
<p>API versioning is essential for maintaining backward compatibility as your API evolves. It allows you to introduce new features, fix bugs, and make changes without disrupting existing clients. There are several strategies for versioning APIs, including:</p>
<ul>
<li>URL Path Versioning (e.g., <code>/api/v1/resource</code>)</li>
<li>Query String Versioning (e.g., <code>/api/resource?version=1.0</code>)</li>
<li>Header Versioning (e.g., <code>X-API-Version: 1.0</code>)</li>
<li>Media Type Versioning (e.g., <code>Accept: application/json; v=1.0</code>)
<ul>
<li>This is less common in ASP.NET Core applications due to the need for custom media type formatters, but it is still a valid and widely-used approach in the industry. GitHub is a well-known example of an API that uses media type versioning.</li>
</ul>
</li>
</ul>
<p>The examples above use a <code>v</code> prefix with either a major version number or a major-minor version number. However, there are other versioning formats you can use, such as date-based versioning (e.g., <code>2026-03-01</code>), status-based versioning (e.g., <code>v1-beta</code>), and more. The versioning format is entirely up to you, and choosing the right versioning strategy depends on your specific use case and client requirements.</p>
<p>While you could implement API versioning yourself, using a library like <a href="https://github.com/dotnet/aspnet-api-versioning">Asp.Versioning</a> simplifies the process significantly, providing built-in support for various versioning strategies and seamless integration with ASP.NET Core.</p>
<p><div class="alert alert-primary"><p class="alert-divider"><i class="fabric-icon fabric-icon--Info"></i><strong>Note</strong></p>This post focuses on <code>Asp.Versioning v10.0.0</code>, the first release to <strong>officially</strong> support both ASP.NET Core 10 and the new built-in OpenAPI library. The prior stable version, <code>Asp.Versioning v8.x.x</code>, will work with .NET 10 via implicit roll-forward, but v10 is purpose-built for the new OpenAPI integration and brings improvements and bug fixes — so it&#8217;s the recommended choice for .NET 10 applications.</div></p>
<p>We&#8217;ll look at how to set this up in both Minimal APIs and controllers in a bit. First, let&#8217;s explore why OpenAPI is important in this context.</p>
<p><div class="alert alert-primary"><p class="alert-divider"><i class="fabric-icon fabric-icon--Info"></i><strong>About the code samples</strong></p>All complete code samples in this post are formatted as <strong><a href="https://learn.microsoft.com/dotnet/core/sdk/file-based-apps">file-based apps</a></strong>, a feature introduced in C# 14 and .NET 10 that lets you run .NET applications from a single <code>.cs</code> file — no project file required! Copy any complete sample to a <code>.cs</code> file and run it with <code>dotnet &lt;filename&gt;.cs</code>. The <code>#:sdk</code> and <code>#:package</code> directives at the top of each sample automatically configure the required SDK and NuGet packages. Make sure you have the <a href="https://dotnet.microsoft.com/download/dotnet/10.0">.NET 10 SDK</a> installed!</div></p>
<h2>The changes to OpenAPI in .NET 9 and 10</h2>
<p>Since .NET 9, <a href="https://www.nuget.org/packages/Microsoft.AspNetCore.OpenApi">Microsoft.AspNetCore.OpenApi</a> has become the default way to generate OpenAPI documentation for ASP.NET Core applications, replacing <code>Swashbuckle.AspNetCore</code>. Setting it up is straightforward, and it seems geared for versioning out of the box, as the URL for accessing the OpenAPI document includes a version segment by default: <code>/openapi/v1.json</code>.</p>
<p><div class="alert alert-primary"><p class="alert-divider"><i class="fabric-icon fabric-icon--Info"></i><strong>Note about OpenAPI tools</strong></p>While Swashbuckle and NSwag are still viable and widely-used options for OpenAPI documentation in .NET, this post focuses on the newer built-in OpenAPI support.</div></p>
<p>If you haven&#8217;t set up OpenAPI in your .NET 9/10 application yet, here&#8217;s a quick example of how to do it:</p>
<pre><code class="language-csharp">#:property PublishAot=false
#:sdk Microsoft.NET.Sdk.Web
#:package Microsoft.AspNetCore.OpenApi@10.0.4

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddOpenApi();

var app = builder.Build();

// This sets up the OpenAPI endpoint at /openapi/v1/openapi.json
// If you'd prefer YAML, you can change the URL to end up with .yaml instead
app.MapOpenApi();

app.MapGet("/users", () =&gt;
{
    var users = new List&lt;UserDto&gt;
    {
        new(1, "Ada Lovelace", "ada@example.com"),
        new(2, "Grace Hopper", "grace@example.com"),
        new(3, "Conner Pilot", "copilot@example.com"),
    };

    return TypedResults.Ok&lt;List&lt;UserDto&gt;&gt;(users);
})
.WithName("GetUsers");

app.Run();

record UserDto(int Id, string Name, string Email);</code></pre>
<pre><code class="language-json">{
  "openapi": "3.1.1",
  "info": {
    "title": "UsersApi | v1",
    "version": "1.0.0"
  },
  "servers": [
    {
      "url": "http://localhost:5055/"
    }
  ],
  "paths": {
    "/users": {
      "get": {
        "tags": ["UsersApi"],
        "operationId": "GetUsers",
        "responses": {
          "200": {
            "description": "OK",
            "content": {
              "application/json": {
                "schema": {
                  "type": "array",
                  "items": {
                    "$ref": "#/components/schemas/UserDto"
                  }
                }
              }
            }
          }
        }
      }
    }
  },
  "components": {
    "schemas": {
      "UserDto": {
        "required": ["id", "name", "email"],
        "type": "object",
        "properties": {
          "id": {
            "pattern": "^-?(?:0|[1-9]\\d*)$",
            "type": ["integer", "string"],
            "format": "int32"
          },
          "name": {
            "type": "string"
          },
          "email": {
            "type": "string"
          }
        }
      }
    }
  },
  "tags": [
    {
      "name": "UsersApi"
    }
  ]
}</code></pre>
<p>You can also customize the OpenAPI document by using transformers and enrich your operations by using <code>TypedResults</code>. To learn about this and other approaches, check out the official documentation for <a href="https://learn.microsoft.com/aspnet/core/fundamentals/openapi/overview">Microsoft.AspNetCore.OpenApi</a>.</p>
<h2>An introduction to API versioning with Asp.Versioning</h2>
<p>Before we dive into the specifics of how to set up API versioning with OpenAPI in .NET 10, let&#8217;s briefly introduce the <a href="https://github.com/dotnet/aspnet-api-versioning">Asp.Versioning</a> library.
This library provides a comprehensive solution for API versioning in ASP.NET Core applications, supporting various versioning strategies and seamless integration with both Minimal APIs and controllers.
It has been widely adopted in the .NET community, having 800 million downloads with all packages combined!</p>
<p><code>Asp.Versioning</code> is a collection of several libraries that you can use to add versioning for controllers, Minimal APIs, OData, and more. In this post, we&#8217;ll focus only on controllers and Minimal APIs, which require the following packages:</p>
<table>
<thead>
<tr>
<th>API Type</th>
<th>Required Packages (.NET 10+)</th>
</tr>
</thead>
<tbody>
<tr>
<td>Controllers</td>
<td><code>Asp.Versioning.Mvc</code>
<code>Asp.Versioning.Mvc.ApiExplorer</code></td>
</tr>
<tr>
<td>Minimal APIs</td>
<td><code>Asp.Versioning.Http</code>
<code>Asp.Versioning.Mvc.ApiExplorer</code></td>
</tr>
</tbody>
</table>
<p>The library has an interesting history, being part of the .NET Foundation and developed by <a href="https://github.com/commonsensesoftware">Chris Martinez</a> while he worked at Microsoft.</p>
<p><div class="alert alert-primary"><p class="alert-divider"><i class="fabric-icon fabric-icon--Info"></i><strong>Note</strong></p>Looking for the source code? All of the code in this post, and more, can be found in my <a href="https://github.com/sander1095/openapi-versioning">sample repository on GitHub</a>. For more samples, check out <a href="https://github.com/dotnet/aspnet-api-versioning/tree/main/examples">Asp.Versioning&#8217;s official samples</a>.</div></p>
<p>To demonstrate how to set up API versioning in .NET 10, let&#8217;s create a simple sample application that includes a minimal amount of code required to get started. This will use query string versioning for the sake of simplicity, but you can easily swap to another strategy if you prefer, which will be <a href="#changing-the-versioning-strategy">covered later</a>.</p>
<h3>API versioning for controllers</h3>
<p>Controllers use the <code>Asp.Versioning.Mvc</code> package, which provides a set of attributes and conventions to define API versions.
You can specify the version for each controller or action using attributes like <code>[ApiVersion("1.0")]</code> and <code>[ApiVersion("2.0")]</code>.</p>
<p>First, you have to set up the required services:</p>
<pre><code class="language-csharp">#:property PublishAot=false
#:sdk Microsoft.NET.Sdk.Web
#:package Asp.Versioning.Mvc@10.0.0

using Asp.Versioning;
using Microsoft.AspNetCore.Mvc;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();

builder.Services.AddApiVersioning()
    .AddMvc();

var app = builder.Build();

app.MapControllers();
app.Run();

// For a file-based app, controller classes go below app.Run()
// (see the next code snippet)</code></pre>
<p>Then you need to add the versioning attributes to your controllers. A solid approach is to have one controller per version:</p>
<pre><code class="language-csharp">[ApiController]
[Route("api/users")]
[ApiVersion("1.0")]
public class UsersV1Controller : ControllerBase
{
    [HttpGet]
    public ActionResult&lt;UserV1[]&gt; Get()
    {
        return Ok(new[]
        {
            new UserV1(1, "John Doe"),
            new UserV1(2, "Alice Dewett"),
        });
    }
}

[ApiController]
[Route("api/users")]
[ApiVersion("2.0")]
public class UsersV2Controller : ControllerBase
{
    [HttpGet]
    public ActionResult&lt;UserV2[]&gt; Get()
    {
        return Ok(new[]
        {
            new UserV2(1, "John Doe", new DateOnly(1990, 1, 1)),
            new UserV2(2, "Alice Dewett", new DateOnly(1992, 2, 2)),
        });
    }
}

public record UserV1(int Id, string Name);
public record UserV2(int Id, string Name, DateOnly BirthDate);</code></pre>
<p>Asp.Versioning supports query string versioning by default. You can now reach these endpoints by going to <code>api/users?api-version=1.0</code> for the first version, and <code>api/users?api-version=2.0</code> for the second version!</p>
<h3>API versioning for Minimal APIs</h3>
<p>Minimal APIs use the <code>Asp.Versioning.Http</code> package instead of <code>Asp.Versioning.Mvc</code>. This package provides extension methods to define API versions directly on the route groups. Before you do that, though, you&#8217;ll need to call <code>NewVersionedApi</code> to create a new API versioning group, which will allow you to define multiple versions in the route group.</p>
<pre><code class="language-csharp">#:property PublishAot=false
#:sdk Microsoft.NET.Sdk.Web
#:package Asp.Versioning.Http@10.0.0

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddApiVersioning();

var app = builder.Build();

var usersApi = app.NewVersionedApi("Users");

var usersv1 = usersApi.MapGroup("api/users").HasApiVersion("1.0");
var usersv2 = usersApi.MapGroup("api/users").HasApiVersion("2.0");

usersv1.MapGet("", () =&gt; TypedResults.Ok(new[]
{
    new UserV1(1, "John Doe"),
    new UserV1(2, "Alice Dewett"),
}));

usersv2.MapGet("", () =&gt; TypedResults.Ok(new[]
{
    new UserV2(1, "John Doe", new DateOnly(1990, 1, 1)),
    new UserV2(2, "Alice Dewett", new DateOnly(1992, 2, 2)),
}));

app.Run();

record UserV1(int Id, string Name);
record UserV2(int Id, string Name, DateOnly BirthDate);</code></pre>
<p>Just like with controllers, you can reach these endpoints by going to <code>api/users?api-version=1.0</code> for the first version, and <code>api/users?api-version=2.0</code> for the second version!</p>
<p><div class="alert alert-success"><p class="alert-divider"><i class="fabric-icon fabric-icon--Lightbulb"></i><strong>How to organize your API versions?</strong></p>Adding all the API groups and versions in the <code>Program.cs</code> file can quickly become unmanageable as your API grows. I really like the approach from one of <code>Asp.Versioning</code>&#8216;s <a href="https://github.com/dotnet/aspnet-api-versioning/blob/08b13779604439c1dc80fab2a84fe9c8591842a2/examples/AspNetCore/WebApi/MinimalOpenApiExample/Program.cs#L46">example projects</a>, which keeps <code>Program.cs</code> focused and easy to scan.</div></p>
<pre><code class="language-csharp">app.MapUsers().ToV1().ToV2().ToV3();
app.MapScores().ToV1().ToV2().ToV3();</code></pre>
<p>Here, <code>MapUsers()</code> and <code>MapScores()</code> are extension methods that call <code>app.NewVersionedApi()</code>, and <code>ToV1()</code>, <code>ToV2()</code>, etc. are extension methods that define the versioned route groups and endpoints. This way, you can keep your <code>Program.cs</code> file clean and organized, and you can easily find and manage your API versions.</p>
<p>For controller-based projects, <code>Asp.Versioning</code> supports convention-based versioning such as Version by .NET Namespace.
See <a href="https://github.com/dotnet/aspnet-api-versioning/wiki/API-Version-Conventions#version-by-namespace-convention">the documentation</a> and <a href="https://github.com/dotnet/aspnet-api-versioning/tree/main/examples/AspNetCore/WebApi/ByNamespaceExample">example in the repo</a> for more information.</p>
<h3>Changing the versioning strategy</h3>
<p>In the examples above, we used query string versioning for simplicity, but <code>Asp.Versioning</code> supports various versioning strategies, and you can easily switch between them by configuring the API versioning options. Let&#8217;s take a look at implementing URL and header versioning.</p>
<h4>URL versioning</h4>
<p>To swap to URL versioning, you need to change <code>AddApiVersioning</code>:</p>
<pre><code class="language-csharp">builder.Services.AddApiVersioning(options =&gt;
{
    // API versioning by URL segment (api/v1/users)
    options.ApiVersionReader = new UrlSegmentApiVersionReader();
});</code></pre>
<p>Now, you can use <code>api/v1/users</code> in the URL to go to the first version of the API, and <code>api/v2/users</code> to go to the second version!</p>
<p>URL versioning is a popular choice as it makes the versioning explicit in the URL, making it very easy to use and see what version you&#8217;re calling.
However, it does mean clients need to update their URLs when a new version is released. Another downside is that it isn&#8217;t &#8220;truly&#8221; RESTful, as the URL should represent the resource, which, even though it is a different version, is still the same resource, and thus the URL should ideally not change. That said, this is a common and widely accepted approach to versioning, and it works well in many scenarios, so it&#8217;s a good option to consider.</p>
<h4>Header versioning</h4>
<p>If you want to use header versioning, you can change the setup to this:</p>
<pre><code class="language-csharp">builder.Services.AddApiVersioning(options =&gt;
{
    // API versioning by header (X-API-Version: 1.0)
    options.ApiVersionReader = new HeaderApiVersionReader("X-API-Version");
});</code></pre>
<p>Header, query string, and media type versioning are more RESTful, as the URL represents the resource, and the version is specified in the header, query string, or content type. This allows clients to call the same URL regardless of the version, and simply specify the version they want to use in the header, query string, or content type.</p>
<p>This approach, just like query string versioning, does lead to a scenario of a user potentially forgetting to specify the version. To deal with this, you can set a default API version, which will be used when no version is specified. This can be done by setting the <code>DefaultApiVersion</code> property in the API versioning options:</p>
<pre><code class="language-csharp">builder.Services.AddApiVersioning(options =&gt;
{
    // Set the default API version to 1.0 explicitly
    // This is already set to 1.0 by default, but shown here for demonstration
    options.DefaultApiVersion = new ApiVersion(1, 0);

    // If the user does not specify a version, you can let the API use the default version
    // This is disabled by default.
    // Enabling this feature is a trade-off between convenience and explicitness.
    // Changing the default version could break clients that aren't using versioning.
    // Consider your API's audience and usage patterns when deciding to enable this.
    options.AssumeDefaultVersionWhenUnspecified = true;
});</code></pre>
<p>It&#8217;s also possible to combine multiple versioning strategies, for example allowing clients to use either query or header versioning,
which can be useful for supporting different types of clients. To do this, you can use the <code>ApiVersionReader.Combine</code> method:</p>
<pre><code class="language-csharp">builder.Services.AddApiVersioning(options =&gt;
{
    options.ApiVersionReader = ApiVersionReader.Combine(
        new QueryStringApiVersionReader("api-version"),
        new HeaderApiVersionReader("X-API-Version")
    );
});</code></pre>
<p>And now that you understand the basics of API versioning, it&#8217;s time to put OpenAPI and API versioning together! Keep in mind that there are many more features to explore in <code>Asp.Versioning</code>, so make sure to check out <a href="https://github.com/dotnet/aspnet-api-versioning/wiki">the official documentation</a> and the <a href="https://github.com/dotnet/aspnet-api-versioning/tree/main/examples">samples</a>!</p>
<h2>Combining API versioning with OpenAPI in .NET 10</h2>
<p><div class="alert alert-primary"><p class="alert-divider"><i class="fabric-icon fabric-icon--Info"></i><strong>Note</strong></p><code>Asp.Versioning.OpenApi</code> v10.0.0-rc.1 is currently in Release Candidate. See the <a href="https://github.com/dotnet/aspnet-api-versioning/releases/tag/v10.0.0">release notes</a> for details.</div></p>
<p>This section will also cover both controllers and Minimal APIs. As discussed at the beginning of this post, <code>Asp.Versioning</code> v10.0.0 introduces a <strong>new</strong> package that can be used to integrate API versioning with OpenAPI in a clean and simple way, without having to write a lot of custom code or duplicate configuration: <code>Asp.Versioning.OpenApi</code>. This package is required for both controllers and Minimal APIs, and it provides a set of extension methods to generate OpenAPI documentation for each API version.</p>
<p>We&#8217;ll update the samples we created in the previous sections to include OpenAPI documentation for each version of the API with the query string versioning strategy. We&#8217;ll also focus on one document per version, which is the recommended approach for versioning your OpenAPI documentation, as it allows clients to easily access the documentation for the specific version of the API they are using, without having to filter through a single document that contains all versions.</p>
<h3>Setting up API versioning with OpenAPI for controllers</h3>
<p>Combining OpenAPI with API versioning for controllers requires the following changes to the setup:</p>
<ul>
<li>You must call <code>AddApiExplorer</code> after <code>AddApiVersioning</code> to ensure that the API versioning information is included in the OpenAPI document.
<ul>
<li>The API Explorer is ASP.NET Core&#8217;s built-in service for discovering and describing the API endpoints in your application. By adding it after <code>AddApiVersioning</code>, you ensure that the versioning information is included in the API descriptions, which is crucial for generating accurate OpenAPI documentation.</li>
</ul>
</li>
<li>You must call <code>AddOpenApi</code> from the <code>Asp.Versioning</code> namespace after activating API versioning to ensure that you use the correct variant of <code>AddOpenApi</code> that integrates with API versioning.</li>
<li>We call <code>WithDocumentPerVersion()</code> after <code>MapOpenApi()</code> to generate a separate OpenAPI document for each API version, preventing us from having to manually call <code>AddOpenApi()</code> multiple times for each version, which can lead to maintenance issues when having to update both Controller attributes and OpenAPI configuration when adding new versions.</li>
</ul>
<pre><code class="language-csharp">#:property PublishAot=false
#:sdk Microsoft.NET.Sdk.Web
#:package Asp.Versioning.Mvc@10.0.0
#:package Asp.Versioning.Mvc.ApiExplorer@10.0.0
#:package Asp.Versioning.OpenApi@10.0.0-rc.1

using Asp.Versioning;
using Microsoft.AspNetCore.Mvc;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();

// We don't need to customize the API versioning options for this example as we are using query string versioning.
builder.Services.AddApiVersioning()
.AddApiExplorer(options =&gt;
{
    // Calling "AddApiExplorer" is required for OpenAPI versioning to work correctly.
    // Without this, the generated OpenAPI documents will not be versioned.

    // GroupNameFormat specifies the format of the API version.
    // Without this, versioning will use the literal group names. In our case, that would be 1.0.
    // For compatibility with the "default" /openapi/v1.json behavior from Microsoft.AspNetCore.OpenApi, we use v'VVV' so we can retrieve it using v1.json.
    // See https://github.com/dotnet/aspnet-api-versioning/wiki/Version-Format#custom-api-version-format-strings for more information about formatting API versions.
    options.GroupNameFormat = "'v'VVV";
})
.AddMvc()
// You must call "AddOpenApi" after "AddApiVersioning" to ensure you use Asp.Versioning's variant.
// This variant of "AddOpenApi" is required to properly integrate with API versioning and generate versioned OpenAPI documents.
// You can call an overload of "AddOpenApi" to customize the OpenAPI generation, just like you would with Microsoft.AspNetCore.OpenApi's "AddOpenApi".
.AddOpenApi();

var app = builder.Build();

// WithDocumentPerVersion() is an extension method provided by the Asp.Versioning.OpenApi package.
// It configures the OpenAPI endpoint to generate a separate document for each API version.
// This allows clients to retrieve documentation specific to the version of the API they are using.
// This approach is preferable compared to having to call "services.AddOpenApi()" multiple times for each version, which can lead to maintenance issues and potential misconfigurations when adding new versions.
app.MapOpenApi().WithDocumentPerVersion();

app.MapControllers();

app.Run();

// For a file-based app, paste the controller classes from
// the "API versioning for controllers" section below app.Run()</code></pre>
<p>You can now retrieve your versioned OpenAPI documents at <code>/openapi/v1.json</code> and <code>/openapi/v2.json</code> for the first and second version of the API, respectively!</p>
<h3>Setting up API versioning with OpenAPI for Minimal APIs</h3>
<p>Next up, Minimal APIs! Luckily, the code is the exact same as for controllers, except for the fact that we do not need to call <code>AddMvc</code>. In case you do want to see it:</p>
<pre><code class="language-csharp">#:property PublishAot=false
#:sdk Microsoft.NET.Sdk.Web
#:package Asp.Versioning.Http@10.0.0
#:package Asp.Versioning.Mvc.ApiExplorer@10.0.0
#:package Asp.Versioning.OpenApi@10.0.0-rc.1

using Asp.Versioning;

var builder = WebApplication.CreateBuilder(args);

// We don't need to customize the API versioning options for this example as we are using query string versioning.
builder.Services.AddApiVersioning()
.AddApiExplorer(options =&gt;
{
    // Calling "AddApiExplorer" is required for OpenAPI versioning to work correctly.
    // Without this, the generated OpenAPI documents will not be versioned.

    // GroupNameFormat specifies the format of the API version.
    // Without this, versioning will use the literal group names. In our case, that would be 1.0.
    // For compatibility with the "default" /openapi/v1.json behavior from Microsoft.AspNetCore.OpenApi, we use v'VVV' so we can retrieve it using v1.json.
    // See https://github.com/dotnet/aspnet-api-versioning/wiki/Version-Format#custom-api-version-format-strings for more information about formatting API versions.
    options.GroupNameFormat = "'v'VVV";
})
// You must call "AddOpenApi" after "AddApiVersioning" to ensure you use Asp.Versioning's variant.
// This variant of "AddOpenApi" is required to properly integrate with API versioning and generate versioned OpenAPI documents.
// You can call an overload of "AddOpenApi" to customize the OpenAPI generation, just like you would with Microsoft.AspNetCore.OpenApi's "AddOpenApi".
.AddOpenApi();

var app = builder.Build();

// WithDocumentPerVersion() is an extension method provided by the Asp.Versioning.OpenApi package.
// It configures the OpenAPI endpoint to generate a separate document for each API version.
// This allows clients to retrieve documentation specific to the version of the API they are using.
// This approach is preferable compared to having to call "services.AddOpenApi()" multiple times for each version, which can lead to maintenance issues and potential misconfigurations when adding new versions.
app.MapOpenApi().WithDocumentPerVersion();

// Paste the API endpoints and records from the "API versioning for Minimal APIs" section here,
// then add `app.Run();` at the end.</code></pre>
<p>Now you know how to set up API versioning with OpenAPI for both controllers and Minimal APIs in .NET 10!</p>
<h2>Adding SwaggerUI and Scalar support for versioned APIs</h2>
<p>Now that we have our versioned OpenAPI documents, we can add support for visualizing them using tools like <a href="https://github.com/domaindrivendev/Swashbuckle.AspNetCore">SwaggerUI</a> and <a href="https://github.com/scalar/scalar">Scalar</a>. Both of these tools allow you to visualize your API documentation in a user-friendly way, making it easier for developers to understand and interact with your API.</p>
<p>SwaggerUI used to be included by default in ASP.NET Core applications thanks to <code>Swashbuckle.AspNetCore</code>, a NuGet package included in ASP.NET Core project templates. This is no longer the case since ASP.NET Core 9 with the introduction of <code>Microsoft.AspNetCore.OpenApi</code>.</p>
<p>Scalar is a newer tool that provides a more modern and customizable interface for visualizing OpenAPI documentation, and it can be added to your project using the <code>Scalar.AspNetCore</code> NuGet package. Performance-wise, Scalar is more efficient than SwaggerUI, but both tools are great options for visualizing your API documentation, and the choice between them depends on your specific needs and preferences.</p>
<h3>Adding SwaggerUI support</h3>
<p>To add <a href="https://github.com/domaindrivendev/Swashbuckle.AspNetCore">SwaggerUI</a> support for your versioned APIs, you can use the <code>Swashbuckle.AspNetCore.SwaggerUI</code> package, which provides middleware to serve the SwaggerUI interface. Unlike the full <code>Swashbuckle.AspNetCore</code> package, this only includes the UI component and does not include OpenAPI document generation, as we are using <code>Microsoft.AspNetCore.OpenApi</code> for that. You can configure this package to point to your versioned OpenAPI documents, allowing developers to easily explore and test your API endpoints.</p>
<p>The setup required is the same for both controllers and Minimal APIs. We&#8217;ll cover the setup for Minimal APIs, but the same code can be used for controllers as well.</p>
<pre><code class="language-csharp">#:property PublishAot=false
#:sdk Microsoft.NET.Sdk.Web
#:package Asp.Versioning.Http@10.0.0
#:package Asp.Versioning.Mvc.ApiExplorer@10.0.0
#:package Asp.Versioning.OpenApi@10.0.0-rc.1
#:package Swashbuckle.AspNetCore.SwaggerUI@10.1.4

using Asp.Versioning;
using Asp.Versioning.ApiExplorer;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddApiVersioning()
.AddApiExplorer(options =&gt;
{
    options.GroupNameFormat = "'v'VVV";
})
.AddOpenApi();

var app = builder.Build();

app.MapOpenApi().WithDocumentPerVersion();

// Paste the API endpoints and records from the "API versioning for Minimal APIs" section here

// UseSwaggerUI MUST come after MapOpenApi() and the API endpoint definitions.
app.UseSwaggerUI(options =&gt;
{
    // We reverse the list of API versions so the newest version is rendered first
    foreach (var description in app.DescribeApiVersions().Reverse())
    {
        options.SwaggerEndpoint(
            $"/openapi/{description.GroupName}.json",
            description.GroupName.ToUpperInvariant());
    }
});

app.Run();</code></pre>
<p>We&#8217;ve now added SwaggerUI support by calling <code>app.UseSwaggerUI()</code> and configuring it to point to our versioned OpenAPI documents, based on the API versions described in our application, which we retrieve using <code>app.DescribeApiVersions()</code>. We can visit the SwaggerUI interface at <code>/swagger</code> to explore and test our API endpoints!</p>
<p><img alt="SwaggerUI showing versioned API documentation with two API versions in the dropdown." src="./swaggerui.png" /></p>
<p>Figure: <em>SwaggerUI with versioned API documentation</em></p>
<h3>Adding Scalar support</h3>
<p>Next, we can add <a href="https://github.com/scalar/scalar">Scalar</a> support for our versioned APIs using the <code>Scalar.AspNetCore</code> package. This package provides middleware to serve the Scalar interface, which can be configured to point to your versioned OpenAPI documents, similar to how we set up SwaggerUI.</p>
<p>Again, the setup is the same for both controllers and Minimal APIs. We&#8217;ll cover the setup for Minimal APIs, but the same code can be used for controllers as well.</p>
<pre><code class="language-csharp">#:property PublishAot=false
#:sdk Microsoft.NET.Sdk.Web
#:package Asp.Versioning.Http@10.0.0
#:package Asp.Versioning.Mvc.ApiExplorer@10.0.0
#:package Asp.Versioning.OpenApi@10.0.0-rc.1
#:package Scalar.AspNetCore@2.13.0

using Asp.Versioning;
using Asp.Versioning.ApiExplorer;
using Scalar.AspNetCore;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddApiVersioning()
.AddApiExplorer(options =&gt;
{
    options.GroupNameFormat = "'v'VVV";
})
.AddOpenApi();

var app = builder.Build();

app.MapOpenApi().WithDocumentPerVersion();

// Paste the API endpoints and records from the "API versioning for Minimal APIs" section here

// MapScalarApiReference sets up the Scalar UI at /scalar
// AddDocuments registers all known API versions so Scalar shows a dropdown to switch between them.
// You can enrich your OpenAPI document with Scalar specific integrations if you wish.
// To learn more: https://scalar.com/products/api-references/integrations/aspnetcore/openapi-extensions
app.MapScalarApiReference(options =&gt;
{
    var descriptions = app.DescribeApiVersions();

    for (var i = 0; i &lt; descriptions.Count; i++)
    {
        var description = descriptions[i];
        var isDefault = i == descriptions.Count - 1;

        // isDefault is used to mark the default API version in Scalar.
        // This decides which version is selected by default when users visit the Scalar UI.
        options.AddDocument(description.GroupName, description.GroupName, isDefault: isDefault);
    }
});

app.Run();</code></pre>
<p>With <code>app.MapScalarApiReference()</code>, we register Scalar and feed it the same versioned documents via <code>app.DescribeApiVersions()</code>. Visit <code>/scalar</code> to browse and test your endpoints. Scalar is also highly configurable — select <em>Configure</em> in the top-right corner to tweak the theme, layout, and more.</p>
<p><img alt="Scalar UI showing two versions of the API in the dropdown menu." src="./scalar.png" /></p>
<p>Figure: <em>Scalar with versioned API documentation</em></p>
<p><div class="alert alert-success"><p class="alert-divider"><i class="fabric-icon fabric-icon--Lightbulb"></i><strong>Can't decide between SwaggerUI and Scalar?</strong></p>If you&#8217;re having trouble deciding between SwaggerUI and Scalar, you can actually use both! Both tools can be configured to point to your versioned OpenAPI documents, allowing developers to choose their preferred interface for exploring and testing your API endpoints. You can set up SwaggerUI at <code>/swagger</code> and Scalar at <code>/scalar</code>, giving developers the flexibility to use the tool they are most comfortable with.</div></p>
<p>We now have a complete setup for API versioning with OpenAPI in .NET 10, along with support for visualizing our API documentation using both SwaggerUI and Scalar!</p>
<h2>Migrating from Asp.Versioning v8 to v10</h2>
<p>You might encounter some breaking changes during the migration process, just like I did. <a href="https://github.com/sander1095/openapi-versioning/commit/0623dc7ae35eb5a2e253d1771c1d3c03addb24f3">This commit</a> highlights some of the changes I had to make to get my sample application working with the new version. The most significant change is that the <code>Asp.Versioning.OpenApi</code> package is now required for both controllers and Minimal APIs, and that <code>AddOpenApi()</code> must be called from the <code>Asp.Versioning</code> namespace instead of the <code>Microsoft.AspNetCore</code> namespace, after activating API versioning.</p>
<p>During this migration, I actually found a bug in the new version of <code>Asp.Versioning</code> that caused the OpenAPI document to not generate correctly for Minimal APIs, so I <a href="https://github.com/dotnet/aspnet-api-versioning/pull/1166">created a PR for this</a>. For more information about changes between versions, check out the changes from v8 to v10 in <a href="https://github.com/sander1095/openapi-versioning">my sample repository</a> and <a href="https://github.com/dotnet/aspnet-api-versioning/wiki">the official documentation</a> for <code>Asp.Versioning</code>!</p>
<h2>How Asp.Versioning v10 improves the setup of API versioning</h2>
<p>This post has covered the new way of setting up API versioning with OpenAPI in .NET 10 using <code>Asp.Versioning</code> v10.0.0. It also made claims that this new approach reduces duplicate code and makes it easier to set up.</p>
<p>To understand what I mean by this, let&#8217;s compare the new approach to how you would set up API versioning with OpenAPI in <code>Asp.Versioning</code> v8.x.x:</p>
<p><strong>Asp.Versioning v8.x.x:</strong></p>
<pre><code class="language-csharp">var builder = WebApplication.CreateBuilder(args);

builder.Services.AddOpenApi("v1");
builder.Services.AddOpenApi("v2");

builder.Services.AddApiVersioning()
.AddApiExplorer(options =&gt;
{
    options.GroupNameFormat = "'v'VVV";
});

var app = builder.Build();

// Code for the API endpoints using app.NewVersionedApi() can be placed here.

app.MapOpenApi();</code></pre>
<p><strong>Asp.Versioning v10.x.x:</strong></p>
<pre><code class="language-csharp">var builder = WebApplication.CreateBuilder(args);

builder.Services.AddApiVersioning()
.AddApiExplorer(options =&gt;
{
    options.GroupNameFormat = "'v'VVV";
})
.AddOpenApi();

var app = builder.Build();

// Code for the API endpoints using app.NewVersionedApi() can be placed here.

app.MapOpenApi().WithDocumentPerVersion();</code></pre>
<p>The main differences are that in the new version, you only need to call <code>AddOpenApi()</code> once, instead of having to call <code>AddOpenApi()</code> multiple times for each version. This reduces duplicate code between your API endpoints where you already define your API versions. A combination of Asp.Versioning&#8217;s <code>AddOpenApi()</code> and <code>WithDocumentPerVersion()</code> achieves this behavior.</p>
<p>However, this is just the beginning. When you consider SwaggerUI/Scalar, tools many people use in their API development process, a lot more work was needed for Asp.Versioning v8 to get these tools working with versioned OpenAPI documents. Several OpenAPI transformers were needed to manually add the versioning information to the OpenAPI document. You can see <a href="https://github.com/sander1095/openapi-versioning/blob/ef9f7aac3aea0941442d375c1e7305cf0a242c49/v8/minimal-api/queryheader-versioning-openapi-swaggerui/Program.cs#L16-L20">the code that was needed to get SwaggerUI working with versioned OpenAPI documents using Asp.Versioning v8</a>. These workarounds are no longer necessary as this is now included in Asp.Versioning v10.</p>
<h2>Adding API linting to your versioned OpenAPI documents</h2>
<p>We&#8217;ve covered API versioning, integration with OpenAPI, and how to visualize your versioned API documentation using SwaggerUI and Scalar. What&#8217;s next? Well, you can add API linting to your versioned OpenAPI documents to ensure that they adhere to best practices and organizational standards. This can be done using tools like:</p>
<ul>
<li>Spectral</li>
<li>oasdiff
<ul>
<li>or other OpenAPI diffing tools, like <a href="https://github.com/OpenAPITools/openapi-diff">openapi-diff</a> or <a href="https://github.com/pb33f/openapi-changes">openapi-changes</a>.</li>
</ul>
</li>
</ul>
<p><a href="https://github.com/stoplightio/spectral">Spectral</a> is a powerful tool for linting your OpenAPI documents. By defining custom rules — or using community-built ones — you can enforce consistency across your API development process. This turns your &#8220;guidelines&#8221; into <em>enforceable rules</em>, which can be a game-changer for teams looking to maintain high-quality APIs.</p>
<p>You can, for example, add custom rules for validating that all APIs implement API versioning in a specific way. If a team forgets to add API versioning, Spectral can catch this during the pull request review process, preventing unversioned APIs from being released.</p>
<p>Next, there&#8217;s tooling like <a href="https://github.com/oasdiff/oasdiff">oasdiff</a>, which allows you to compare different versions of your OpenAPI documents to identify changes, additions, or removals in your API. This is especially useful to detect unintended breaking changes between API versions, notifying the developer to introduce a new API version instead of breaking the existing one.</p>
<p>By integrating <code>oasdiff</code> into your CI/CD pipeline, you can let the pull request review process fail once a breaking change is detected, and instruct the contributor to use API versioning instead!</p>
<h2>Finishing up</h2>
<p>I hope you enjoyed this post! Whenever I had to implement API versioning with OpenAPI in .NET in the past, I often got caught up in the intricacies, so I&#8217;m glad I was able to write a setup that works for modern projects, and I hope you found it useful, too! If you have any questions or feedback, feel free to reach out or leave a comment below. Happy coding!</p>
<h2>ASP.NET Core Community Standup</h2>
<p>Catch the recent interview on the ASP.NET Core Community Standup: <a href="https://www.youtube.com/watch?v=7m3r6slW68U">Combining API Versioning with OpenAPI</a>:</p>
<p></p>
<p><strong>Author</strong>:
<a href="https://stenbrinke.nl">Sander ten Brinke</a> is a Microsoft MVP and Senior Software Engineer, with a passion for building scalable and maintainable applications. With over 10 years of experience in the industry, he has worked on a wide range of projects, from small startups to large enterprises. He focuses on .NET and Azure, but his interests extend beyond these technologies too, and he enjoys sharing his knowledge through blogging, speaking at conferences, and contributing to open source software, like some of the OpenAPI features he added to ASP.NET Core 10!</p>
<p>The post <a href="https://devblogs.microsoft.com/dotnet/api-versioning-in-dotnet-10-applications/">Combining API versioning with OpenAPI in .NET 10 applications</a> appeared first on <a href="https://devblogs.microsoft.com/dotnet">.NET Blog</a>.</p>
