---
title: "What’s new for .NET in Ubuntu 26.04"
url: "https://devblogs.microsoft.com/dotnet/whats-new-for-dotnet-in-ubuntu-2604/"
date: "Thu, 23 Apr 2026 17:25:00 +0000"
author: "Richard Lander"
feed_url: "https://devblogs.microsoft.com/dotnet/feed/"
---
<p>Today is launch day for <a href="https://canonical.com/blog/canonical-releases-ubuntu-26-04-lts-resolute-raccoon">Ubuntu 26.04</a> (<a href="https://ubuntu.com/blog/unmasking-the-resolute-raccoon">Resolute Raccoon</a>). Congratulations on the release to our friends at Canonical. Each new Ubuntu LTS comes with the latest .NET LTS so that you can develop and run .NET apps easily. Ubuntu 26.04 comes with .NET 10. You can also install .NET 8 and 9 via a separate PPA/feed. Installation instructions are demonstrated later in the post. .NET is one of the <a href="https://ubuntu.com/toolchains">officially supported toolchains</a> on Ubuntu. The two companies work together to ensure that .NET works well on Ubuntu.</p>
<p>To install .NET 10:</p>
<pre><code class="language-bash">sudo apt update
sudo apt install dotnet-sdk-10.0</code></pre>
<p><a href="https://github.com/dotnet/dotnet-docker/issues/7094">Ubuntu 26.04 container images</a> are also available for .NET 10+, released earlier this month. They use the <code>resolute</code> tag. You can update <code>-noble</code> tags to <code>-resolute</code> to upgrade to the new OS.</p>
<p><a href="https://documentation.ubuntu.com/release-notes/26.04/summary-for-lts-users/">Ubuntu 26.04 release notes</a> describe many changes. The most relevant changes are <a href="https://lkml.org/lkml/2026/4/12/604">Linux 7.0</a>, post-quantum cryptography, and removal of <a href="https://documentation.ubuntu.com/release-notes/26.04/summary-for-lts-users/#systemd-259">cgroup v1 (container-related)</a>. We will start Linux 7.0 testing once we get Ubuntu 26.04 VMs in our lab, shortly. We added <a href="https://devblogs.microsoft.com/dotnet/post-quantum-cryptography-in-dotnet/">support for post-quantum cryptography</a> in .NET 10. We added <a href="https://github.com/dotnet/runtime/issues/30337">support for cgroup v2</a> many years ago so cgroup v1 removal should be a non-event.</p>
<p>We&#8217;ve been on the &#8220;road to next Ubuntu LTS&#8221; since early 2025 when we started testing Debian 13 and Ubuntu 24.10 in CI. We started testing 26.04 in late 2025. Every PR in <a href="https://github.com/dotnet/runtime/pulls">dotnet/runtime</a> runs validation on Ubuntu 26.04. It&#8217;s possible that our repo is one of the biggest users of Ubuntu 26.04 container images, at least up until today. Ubuntu standard releases (24.10, 25.04, &#8230;) invariably require some level of change on our part, which is all part of being ready for GA day.</p>
<h2>Installing .NET 10</h2>
<p>I&#8217;ll demonstrate installing .NET 10 in an <code>ubuntu:resolute</code> container image.</p>
<pre><code class="language-bash">$ docker run --rm -it ubuntu:resolute
$ apt update
$ apt install -y dotnet-sdk-10.0
$ dotnet --version
10.0.105</code></pre>
<p>Let&#8217;s run some C#.</p>
<pre><code class="language-bash">$ dotnet run - &lt;&lt; 'EOF'
using System.Runtime.InteropServices;

Console.WriteLine($"Hello {RuntimeInformation.OSDescription} from .NET {RuntimeInformation.FrameworkDescription}");
EOF
Hello Ubuntu Resolute Raccoon (development branch) from .NET .NET 10.0.5</code></pre>
<p>&#8220;Run some C#&#8221;. That&#8217;s funny wording. The example above is <a href="https://learn.microsoft.com/dotnet/core/sdk/file-based-apps">file-based app</a> constructed via a <a href="https://en.wikipedia.org/wiki/Here_document">heredoc</a> and passed to <code>dotnet run</code> via <code>stdin</code>. Standard unix patterns at play. If you watch closely, AI agents love to do the same thing in Python. Don&#8217;t hesitate to tell them that it works great with C# as well!</p>
<h2>Containers</h2>
<p>The new <code>resolute</code> images will be much the same as <code>noble</code> ones. There is no change in the image flavors we offer, like <a href="https://devblogs.microsoft.com/dotnet/announcing-dotnet-chiseled-containers/">Chiseled</a>. Recall that containers rely on the kernel of the host. For example, a 26.04 container hosted on a 24.04 host (as is the case in my examples) will be using Linux kernel 6.x.</p>
<p>Let&#8217;s try running our <a href="https://github.com/dotnet/dotnet-docker/tree/main/samples/aspnetapp">aspnetapp sample</a>.</p>
<p>We first need to migrate the sample.</p>
<pre><code class="language-bash">$ grep dotnet/ Dockerfile.chiseled
# https://github.com/dotnet/dotnet-docker/blob/main/samples/README.md
FROM --platform=$BUILDPLATFORM mcr.microsoft.com/dotnet/sdk:10.0-noble AS build
FROM mcr.microsoft.com/dotnet/aspnet:10.0-noble-chiseled
$ sed -i "s/noble/resolute/g" Dockerfile.chiseled</code></pre>
<p>And now to build and run it, with <a href="https://github.com/dotnet/designs/blob/main/accepted/2019/support-for-memory-limits.md">resource limits</a>.</p>
<pre><code class="language-bash">docker build --pull -t aspnetapp -f Dockerfile.chiseled .
docker run --rm -it -p 8000:8080 -m 50mb --cpus .5 aspnetapp</code></pre>
<p><img alt="Welcome to .NET" src="https://devblogs.microsoft.com/dotnet/wp-content/uploads/sites/10/2026/04/welcome-to-dotnet.webp" /></p>
<h2>Native AOT</h2>
<p><a href="https://learn.microsoft.com/dotnet/core/deploying/native-aot">Native AOT (NAOT)</a> is a great choice when you want a fast to start self-contained native binary. The <code>-aot</code> variant package gives you most of what you need. I publish all my tools as <a href="https://learn.microsoft.com/dotnet/core/tools/rid-specific-tools">RID-specific tools</a>. That&#8217;s outside the scope of this post. Let&#8217;s focus on the basics. I&#8217;ll use <code>ubuntu:resolute</code> again.</p>
<p>Here&#8217;s the <a href="https://packages.ubuntu.com/resolute/dotnet-sdk-aot-10.0">dotnet-sdk-aot-10.0</a> package, among the other SDK packages:</p>
<pre><code class="language-bash">$ apt list dotnet-sdk*
dotnet-sdk-10.0-source-built-artifacts/resolute 10.0.105-0ubuntu1 amd64
dotnet-sdk-10.0/resolute,now 10.0.105-0ubuntu1 amd64 [installed,automatic]
dotnet-sdk-aot-10.0/resolute,now 10.0.105-0ubuntu1 amd64 [installed]
dotnet-sdk-dbg-10.0/resolute 10.0.105-0ubuntu1 amd64</code></pre>
<p>We need the AOT package + <code>clang</code> (.NET uses LLD for linking):</p>
<pre><code class="language-bash">apt install -y dotnet-sdk-aot-10.0 clang</code></pre>
<p>I&#8217;ll publish the same hello world app (written to a file this time) as NAOT (the default for file-based apps).</p>
<pre><code class="language-bash">$ dotnet publish app.cs
$ du -h artifacts/app/*
1.4M    artifacts/app/app
3.0M    artifacts/app/app.dbg</code></pre>
<p>The binary is 1.4 MB. The <code>.dbg</code> file is a native symbols file, much like Windows PDB. The minimum NAOT binary is about 1.0 MB. The use of the <code>RuntimeInformation</code> class brings in more code.</p>
<p>Let&#8217;s check out the startup performance.</p>
<pre><code class="language-bash">$ time ./artifacts/app/app
Hello Ubuntu Resolute Raccoon (development branch) from .NET .NET 10.0.5

real    0m0.003s
user    0m0.001s
sys     0m0.001s</code></pre>
<p>Pretty good! That&#8217;s 3 milliseconds.</p>
<p>NAOT works equally well for web services. Let&#8217;s take a look at our <a href="https://github.com/dotnet/dotnet-docker/tree/main/samples/releasesapi">releasesapi</a> sample.</p>
<pre><code class="language-bash">$ grep Aot releasesapi.csproj
    &lt;PublishAot&gt;true&lt;/PublishAot&gt;
$ dotnet publish
$ du -h bin/Release/net10.0/linux-x64/publish/*
4.0K    bin/Release/net10.0/linux-x64/publish/NuGet.config
4.0K    bin/Release/net10.0/linux-x64/publish/appsettings.Development.json
4.0K    bin/Release/net10.0/linux-x64/publish/appsettings.json
13M     bin/Release/net10.0/linux-x64/publish/releasesapi
32M     bin/Release/net10.0/linux-x64/publish/releasesapi.dbg
4.0K    bin/Release/net10.0/linux-x64/publish/releasesapi.staticwebassets.endpoints.json
$ ./bin/Release/net10.0/linux-x64/publish/releasesapi
info: Microsoft.Hosting.Lifetime[14]
      Now listening on: http://localhost:5000
info: Microsoft.Hosting.Lifetime[0]
      Application started. Press Ctrl+C to shut down.
info: Microsoft.Hosting.Lifetime[0]
      Hosting environment: Production
info: Microsoft.Hosting.Lifetime[0]
      Content root path: /dotnet-docker/samples/releasesapi</code></pre>
<p>In another terminal:</p>
<pre><code class="language-bash">$ curl -s http://localhost:5000/releases | jq '[.versions[] | select(.supported == true) | {version, supportEndsInDays}]'
[
  {
    "version": "10.0",
    "supportEndsInDays": 942
  },
  {
    "version": "9.0",
    "supportEndsInDays": 207
  },
  {
    "version": "8.0",
    "supportEndsInDays": 207
  }
]</code></pre>
<p>Note: <a href="https://jqlang.org/"><code>jq</code></a> is an excellent tool that is basically &#8220;LINQ over JSON&#8221;. It takes JSON and enables generating new JSON with the analog of anonymous types.</p>
<p>That&#8217;s a 13 MB self-contained web service including a lot of <a href="https://learn.microsoft.com/dotnet/standard/serialization/system-text-json/source-generation">source-generated System.Text.Json</a> metadata and code.</p>
<h2>Installing .NET 8 and 9</h2>
<p>Our partners at Canonical make a hard separation between support and availability. .NET 8 and 9 are provided via the <a href="https://launchpad.net/~dotnet/+archive/ubuntu/backports">dotnet-backports PPA feed</a>. These sometimes older, sometimes newer, versions are offered with &#8220;best-effort support&#8221;. We expect that .NET 11 will be added to this same PPA at GA.</p>
<p>The <code>software-properties-common</code> package is required to configure the feed. It is typically pre-installed on desktop versions of Ubuntu.</p>
<pre><code class="language-bash">apt install -y software-properties-common</code></pre>
<p>Configure the feed:</p>
<pre><code class="language-bash">$ add-apt-repository ppa:dotnet/backports
PPA publishes dbgsym, you may need to include 'main/debug' component
Repository: 'Types: deb
URIs: https://ppa.launchpadcontent.net/dotnet/backports/ubuntu/
Suites: resolute
Components: main
'
Description:
The backports archive provides source-built .NET packages in cases where a version of .NET is not available in the archive for an Ubuntu release.

Currently available Ubuntu releases and .NET backports:

Ubuntu 26.04 LTS (Resolute Raccoon)
├── .NET 8.0 (End of Life on November 10th, 2026) [amd64 arm64]
└── .NET 9.0 (End of Life on November 10th, 2026) [amd64 arm64 s390x ppc64el]

Ubuntu 24.04 LTS (Noble Numbat)
├── .NET 6.0 (End of Life on November 12th, 2024) [amd64 arm64]
├── .NET 7.0 (End of Life on May 14th, 2024)      [amd64 arm64]
└── .NET 9.0 (End of Life on November 10th, 2026) [amd64 arm64 s390x ppc64el]

Ubuntu 22.04 LTS (Jammy Jellyfish)
├── .NET 9.0 (End of Life on November 10th, 2026) [amd64 arm64 s390x ppc64el]
└── .NET 10.0 (End of Life on November 14th, 2028) [amd64 arm64 s390x ppc64el]

Canonical provides best-effort support for packages contained in this archive, which is limited to the upstream lifespan or the support period of the particular Ubuntu version. See the upstream support policy [1] for more information about the upstream support lifespan of .NET releases or the Ubuntu Releases Wiki entry [2] for more information about the support period of any Ubuntu version.

[1] https://dotnet.microsoft.com/en-us/platform/support/policy/dotnet-core
[2] https://wiki.ubuntu.com/Releases
More info: https://launchpad.net/~dotnet/+archive/ubuntu/backports
Adding repository.
Press [ENTER] to continue or Ctrl-c to cancel.</code></pre>
<p>You can see that the support statement for the feed is included.</p>
<p>Once the feed is registered, new <code>dotnet</code> and <code>aspnetcore</code> packages will show up. You can filter them by version or see all of them. Whichever you want.</p>
<pre><code class="language-bash">$ apt list dotnet-*8.0
dotnet-apphost-pack-8.0/resolute 8.0.26-0ubuntu1~26.04.1~ppa1 amd64
dotnet-host-8.0/resolute 8.0.26-0ubuntu1~26.04.1~ppa1 amd64
dotnet-hostfxr-8.0/resolute 8.0.26-0ubuntu1~26.04.1~ppa1 amd64
dotnet-runtime-8.0/resolute 8.0.26-0ubuntu1~26.04.1~ppa1 amd64
dotnet-runtime-dbg-8.0/resolute 8.0.26-0ubuntu1~26.04.1~ppa1 amd64
dotnet-sdk-8.0/resolute 8.0.126-0ubuntu1~26.04.1~ppa1 amd64
dotnet-sdk-dbg-8.0/resolute 8.0.126-0ubuntu1~26.04.1~ppa1 amd64
dotnet-targeting-pack-8.0/resolute 8.0.26-0ubuntu1~26.04.1~ppa1 amd64
dotnet-templates-8.0/resolute 8.0.126-0ubuntu1~26.04.1~ppa1 amd64
$ apt list aspnetcore-runtime-*
aspnetcore-runtime-10.0/resolute 10.0.5-0ubuntu1 amd64
aspnetcore-runtime-8.0/resolute 8.0.26-0ubuntu1~26.04.1~ppa1 amd64
aspnetcore-runtime-9.0/resolute 9.0.15-0ubuntu1~26.04.1~ppa1 amd64
aspnetcore-runtime-dbg-10.0/resolute 10.0.5-0ubuntu1 amd64
aspnetcore-runtime-dbg-8.0/resolute 8.0.26-0ubuntu1~26.04.1~ppa1 amd64
aspnetcore-runtime-dbg-9.0/resolute 9.0.15-0ubuntu1~26.04.1~ppa1 amd64</code></pre>
<p>And the packages that are actually installed on my machine.</p>
<pre><code class="language-bash">apt list --installed 'aspnetcore*' 'dotnet*'
aspnetcore-runtime-10.0/resolute,now 10.0.5-0ubuntu1 amd64 [installed,automatic]
aspnetcore-targeting-pack-10.0/resolute,now 10.0.5-0ubuntu1 amd64 [installed,automatic]
dotnet-apphost-pack-10.0/resolute,now 10.0.5-0ubuntu1 amd64 [installed,automatic]
dotnet-host-10.0/resolute,now 10.0.5-0ubuntu1 amd64 [installed,automatic]
dotnet-hostfxr-10.0/resolute,now 10.0.5-0ubuntu1 amd64 [installed,automatic]
dotnet-runtime-10.0/resolute,now 10.0.5-0ubuntu1 amd64 [installed,automatic]
dotnet-sdk-10.0/resolute,now 10.0.105-0ubuntu1 amd64 [installed,automatic]
dotnet-sdk-aot-10.0/resolute,now 10.0.105-0ubuntu1 amd64 [installed]
dotnet-targeting-pack-10.0/resolute,now 10.0.5-0ubuntu1 amd64 [installed,automatic]
dotnet-templates-10.0/resolute,now 10.0.105-0ubuntu1 amd64 [installed,automatic]</code></pre>
<h2>Summary</h2>
<p>It&#8217;s that time again, another Ubuntu LTS. I wrote a similar post two years ago for <a href="https://devblogs.microsoft.com/dotnet/whats-new-for-dotnet-in-ubuntu-2404/">Ubuntu 24.04</a>. Much is the same. This time around, we put in more effort to ensure that preparing for the next Ubuntu LTS was central to the distro versions we chose for CI testing in dotnet/runtime in the intervening two years. Day-of Ubuntu 26.04 support just &#8220;falls out&#8221; of that. Enjoy!</p>
<p>The post <a href="https://devblogs.microsoft.com/dotnet/whats-new-for-dotnet-in-ubuntu-2604/">What&#8217;s new for .NET in Ubuntu 26.04</a> appeared first on <a href="https://devblogs.microsoft.com/dotnet">.NET Blog</a>.</p>
