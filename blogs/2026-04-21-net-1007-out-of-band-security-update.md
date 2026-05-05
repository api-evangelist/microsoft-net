---
title: ".NET 10.0.7 Out-of-Band Security Update"
url: "https://devblogs.microsoft.com/dotnet/dotnet-10-0-7-oob-security-update/"
date: "Tue, 21 Apr 2026 18:48:34 +0000"
author: "Rahul Bhandari (MSFT)"
feed_url: "https://devblogs.microsoft.com/dotnet/feed/"
---
<p>We are releasing .NET 10.0.7 as an out-of-band (OOB) update to address a security issue introduced in <a href="https://www.nuget.org/packages/Microsoft.AspNetCore.DataProtection">Microsoft.AspNetCore.DataProtection</a></p>
<h2>Security update details</h2>
<p>This release includes a fix for <a href="https://github.com/dotnet/announcements/issues/395">CVE-2026-40372</a></p>
<p>After the Patch Tuesday <code>.NET 10.0.6</code> release, some customers reported that decryption was failing in their applications. This behavior was reported in <a href="https://github.com/dotnet/aspnetcore/issues/66335">aspnetcore issue #66335</a>.</p>
<p>While investigating those reports, we determined that the regression also exposed a vulnerability. In versions <code>10.0.0</code> through <code>10.0.6</code> of the <code>Microsoft.AspNetCore.DataProtection</code> NuGet package, the managed authenticated encryptor could compute its HMAC validation tag over the wrong bytes of the payload and then discard the computed hash, which could result in elevation of privilege.</p>
<p><div class="alert alert-warning"><p class="alert-divider"><i class="fabric-icon fabric-icon--Warning"></i><strong>Update required</strong></p>If your application uses ASP.NET Core Data Protection, update the <code>Microsoft.AspNetCore.DataProtection</code> package to 10.0.7 as soon as possible to address the decryption regression and security vulnerability.</div></p>
<h3>Download .NET 10.0.7</h3>
<table>
<thead>
<tr>
<th></th>
<th>.NET 10.0</th>
</tr>
</thead>
<tbody>
<tr>
<td>Release Notes</td>
<td><a href="https://github.com/dotnet/core/blob/main/release-notes/10.0/README.md">10.0 release notes</a></td>
</tr>
<tr>
<td>Installers and binaries</td>
<td><a href="https://dotnet.microsoft.com/download/dotnet/10.0">10.0.7</a></td>
</tr>
<tr>
<td>Container Images</td>
<td><a href="https://mcr.microsoft.com/catalog?search=dotnet/">images</a></td>
</tr>
<tr>
<td>Linux packages</td>
<td><a href="https://github.com/dotnet/core/blob/main/release-notes/10.0/install-linux.md">10.0</a></td>
</tr>
<tr>
<td>Known Issues</td>
<td><a href="https://github.com/dotnet/core/blob/main/release-notes/10.0/known-issues.md">10.0</a></td>
</tr>
</tbody>
</table>
<h3>Installation guidance</h3>
<ol>
<li>Download and install the <a href="https://dotnet.microsoft.com/download/dotnet/10.0">.NET 10.0.7 SDK or Runtime</a>.</li>
<li>Verify installation by running <code>dotnet --info</code> and confirming you are on 10.0.7.</li>
<li>Rebuild and redeploy your applications using updated images or packages.</li>
</ol>
<h2>Share your feedback</h2>
<p>If you experience any issues after installing this update, please let us know in the <a href="https://github.com/dotnet/core/issues">.NET release feedback issues</a>.</p>
<p>The post <a href="https://devblogs.microsoft.com/dotnet/dotnet-10-0-7-oob-security-update/">.NET 10.0.7 Out-of-Band Security Update</a> appeared first on <a href="https://devblogs.microsoft.com/dotnet">.NET Blog</a>.</p>
