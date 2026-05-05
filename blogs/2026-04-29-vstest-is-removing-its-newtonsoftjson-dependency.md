---
title: "VSTest is Removing its Newtonsoft.Json Dependency"
url: "https://devblogs.microsoft.com/dotnet/vs-test-is-removing-its-newtonsoft-json-dependency/"
date: "Wed, 29 Apr 2026 17:05:00 +0000"
author: "McKenna Barlow"
feed_url: "https://devblogs.microsoft.com/dotnet/feed/"
---
<p>Starting in .NET 11 Preview 4 and Visual Studio 18.8, VSTest, the platform that powers dotnet test and Test Explorer, will no longer depend on Newtonsoft.Json. The platform will now use System.Text.Json on .NET and JSONite on .NET Framework.  </p>
<p>This is a servicing and security change. Most projects require no action. A small number of projects will see an obvious build or test failure and can resolve it with a one line PackageReference.  </p>
<h2>Why this change?</h2>
<p>VSTest has shipped Newtonsoft.Json as part of the .NET SDK and Visual Studio for years. All versions of Newtonsoft.Json below 13.0.0 are now marked vulnerable on NuGet.org, and carrying the dependency exposes the test platform to future advisories in a component it no longer needs. Removing it is part of the <a href="https://github.com/dotnet/sdk/issues/53287">broader effort to remove Newtonsoft.Json from the .NET SDK.</a></p>
<h2>What is not changing?</h2>
<ul>
<li>The VSTest wire format is unchanged. Messages serialize identically whether Newtonsoft.Json, System.Text.Json, or JSONite is used.</li>
<li>Older testhosts remain compatible with the updated platform, and vice versa.</li>
<li>Serialization performance is the same or better.</li>
</ul>
<h2>Who is not affected?</h2>
<p>Most test projects fall into this bucket:</p>
<ul>
<li>Projects that do not use Newtonsoft.Json.</li>
<li>Projects that reference Newtonsoft.Json as a normal PackageReference.</li>
<li>xUnit and NUnit projects running on .NET, or using AppDomains. These already required an explicit reference.</li>
</ul>
<h2>Who is affected, and how to fix it</h2>
<p>Every failure below is non-silent, is reported in the test run, and flows into TRX and the Azure DevOps / GitHub test views.</p>
<h3>Build error: missing Newtonsoft.Json reference</h3>
<p>If a test project uses Newtonsoft.Json types (for example JObject, JsonConvert) without referencing the package, it previously compiled only because Newtonsoft.Json leaked through VSTest. After the update it will not.</p>
<p><strong>Fix:</strong> add the package.</p>
<pre><code class="language-text">&lt;PackageReference Include="Newtonsoft.Json" Version="13.0.3" /&gt;</code></pre>
<h3>Runtime error: FileNotFoundException for Newtonsoft.Json</h3>
<p>Projects that reference Newtonsoft.Json but exclude the runtime asset — for example:</p>
<pre><code class="language-xml">&lt;PackageReference Include="Newtonsoft.Json" Version="13.0.3"&gt;
  &lt;ExcludeAssets&gt;runtime&lt;/ExcludeAssets&gt;
&lt;/PackageReference&gt;</code></pre>
<p>relied on VSTest&#8217;s copy being present at test time. After the update the test run will fail with:</p>
<pre><code class="language-text">System.IO.FileNotFoundException: Could not load file or assembly 'Newtonsoft.Json, Version=13.0.0.0, Culture=neutral, PublicKeyToken=30ad4fe6b2a6aeed'.</code></pre>
<p><strong>Fix:</strong> remove <code>&lt;ExcludeAssets&gt;runtime&lt;/ExcludeAssets&gt;</code>, or install Newtonsoft.Json without excluding runtime assets.</p>
<h3>Extension load error in a test adapter or data collector</h3>
<p>Adapters and data collectors that used Newtonsoft.Json without declaring it as a dependency will fail at load time with a message such as:</p>
<pre><code class="language-text">Data collector 'SampleDataCollector' threw an exception during type loading, construction, or initialization: System.IO.FileNotFoundException: Could not load file or assembly 'Newtonsoft.Json, Version=13.0.0.0, Culture=neutral, PublicKeyToken=30ad4fe6b2a6aeed'.</code></pre>
<p>The message names the exact extension that needs to be updated, however we are not aware of any shipping adapter or collector that hits this today. Extension authors should add Newtonsoft.Json (or migrate off it) in their own package; consumers can add Newtonsoft.Json to the test project, or disable the affected extension until it is updated.</p>
<h3>Public API change</h3>
<p>VSTest exposed Newtonsoft.Json.Linq.JToken in one spot of its communication API. That type is being removed from the public surface. It was only meaningful to code participating in the VSTest protocol and is unlikely to have real world consumers, but extension authors should be aware.  </p>
<h3>When you will see this</h3>
<table>
<thead>
<tr>
<th><strong>Release</strong></th>
<th><strong>Date</strong></th>
</tr>
</thead>
<tbody>
<tr>
<td>.NET 11 preview 4</td>
<td>May 12th, 2026</td>
</tr>
<tr>
<td>Visual Studio 18.8 Insiders 1</td>
<td>June 9, 2026</td>
</tr>
</tbody>
</table>
<h3>Try it early</h3>
<p>Preview packages are available on NuGet as <a href="https://www.nuget.org/packages/Microsoft.TestPlatform/1.0.0-alpha-stj-26213-07">Microsoft.TestPlatform.* version 1.0.0-alpha-stj</a>. Details and discussion: <a href="https://github.com/microsoft/vstest/issues/15677">microsoft/vstest#15677</a>. Source PR: <a href="https://github.com/microsoft/vstest/pull/15540">microsoft/vstest#15540</a>. SDK breaking change tracking: <a href="https://github.com/dotnet/docs/issues/53174">dotnet/docs#53174</a>.</p>
<p>If your build or test run fails after updating with an error mentioning Newtonsoft.Json, the fix is almost always a single PackageReference. For anything not covered above, please open an issue on <a href="https://github.com/microsoft/vstest/issues">microsoft/vstest</a>.</p>
<p>The post <a href="https://devblogs.microsoft.com/dotnet/vs-test-is-removing-its-newtonsoft-json-dependency/">VSTest is Removing its Newtonsoft.Json Dependency</a> appeared first on <a href="https://devblogs.microsoft.com/dotnet">.NET Blog</a>.</p>
