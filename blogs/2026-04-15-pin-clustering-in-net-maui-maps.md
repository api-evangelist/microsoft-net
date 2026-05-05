---
title: "Pin Clustering in .NET MAUI Maps"
url: "https://devblogs.microsoft.com/dotnet/pin-clustering-in-dotnet-maui-maps/"
date: "Wed, 15 Apr 2026 16:45:00 +0000"
author: "David Ortinau"
feed_url: "https://devblogs.microsoft.com/dotnet/feed/"
---
<p>If you&#8217;ve ever loaded a map with dozens — or hundreds — of pins, you know the result: an overlapping mess that&#8217;s impossible to interact with. Starting in <a href="https://devblogs.microsoft.com/dotnet/dotnet-11-preview-3/">.NET MAUI 11 Preview 3</a>, the Map control supports pin clustering out of the box on Android and iOS/Mac Catalyst.</p>
<h2>What is pin clustering?</h2>
<p>Pin clustering automatically groups nearby pins into a single cluster marker when zoomed out. As you zoom in, clusters expand to reveal individual pins. This is a long-requested feature (<a href="https://github.com/dotnet/maui/issues/11811">#11811</a>) and one that every map-heavy app needs.</p>
<h2>Enable clustering</h2>
<p>A single property is all it takes:</p>
<pre><code class="language-xml">&lt;maps:Map IsClusteringEnabled="True" /&gt;</code></pre>
<p>That&#8217;s it. Nearby pins are now grouped into clusters with a count badge showing how many pins are in each group.</p>
<h2>Separate clustering groups</h2>
<p>Not all pins are the same. You might want coffee shops to cluster independently from parks, or hotels separately from attractions. The <code>ClusteringIdentifier</code> property on <code>Pin</code> makes this possible:</p>
<pre><code class="language-csharp">map.Pins.Add(new Pin
{
    Label = "Pike Place Coffee",
    Location = new Location(47.6097, -122.3331),
    ClusteringIdentifier = "coffee"
});

map.Pins.Add(new Pin
{
    Label = "Occidental Square",
    Location = new Location(47.6064, -122.3325),
    ClusteringIdentifier = "parks"
});</code></pre>
<p>Pins with the same identifier cluster together. Pins with different identifiers form independent clusters even when geographically close. Pins with no identifier share a default group.</p>
<h2>Handle cluster taps</h2>
<p>When a user taps a cluster, the <code>ClusterClicked</code> event fires with a <code>ClusterClickedEventArgs</code> that gives you access to the pins in the cluster:</p>
<pre><code class="language-csharp">map.ClusterClicked += async (sender, e) =&gt;
{
    string names = string.Join("\n", e.Pins.Select(p =&gt; p.Label));
    await DisplayAlert(
        $"Cluster ({e.Pins.Count} pins)",
        names,
        "OK");

    // Suppress default zoom-to-cluster behavior:
    // e.Handled = true;
};</code></pre>
<p>The event args include:</p>
<ul>
<li><strong><code>Pins</code></strong> — the pins in the cluster</li>
<li><strong><code>Location</code></strong> — the geographic center of the cluster</li>
<li><strong><code>Handled</code></strong> — set to <code>true</code> if you want to handle the tap yourself instead of the default zoom behavior</li>
</ul>
<h2>Platform notes</h2>
<p>On <strong>Android</strong>, clustering uses a custom grid-based algorithm that recalculates when the zoom level changes. No external dependencies are required.</p>
<p>On <strong>iOS and Mac Catalyst</strong>, clustering leverages the native <code>MKClusterAnnotation</code> support in MapKit, providing smooth, platform-native cluster animations.</p>
<h2>Try it out</h2>
<p>The <a href="https://github.com/dotnet/maui-samples/tree/main/10.0/UserInterface/Views/Map/MapDemo/WorkingWithMaps">Maps sample in maui-samples</a> includes a new <strong>Clustering</strong> page that demonstrates pin clustering with multiple clustering groups and cluster tap handling.</p>
<p>For the full API reference and additional examples, see the <a href="https://learn.microsoft.com/dotnet/maui/user-interface/controls/map?view=net-maui-11.0#pins">Maps documentation</a>.</p>
<h2>Summary</h2>
<p>Pin clustering brings a polished, production-ready experience to .NET MAUI Maps. Enable it with a single property, customize it with clustering identifiers, and handle taps with a straightforward event. We&#8217;re looking forward to seeing what you build with it.</p>
<p>Get started by installing <a href="https://dotnet.microsoft.com/download/dotnet/11.0">.NET 11 Preview 3</a> and updating or installing the .NET MAUI workload.</p>
<p>If you’re on Windows using Visual Studio, we recommend installing the latest <a href="https://visualstudio.microsoft.com/insiders">Visual Studio 2026 Insiders</a>. You can also use Visual Studio Code and the <a href="https://marketplace.visualstudio.com/items?itemName=ms-dotnettools.csdevkit">C# Dev Kit</a> extension with .NET 11.</p>
<p>The post <a href="https://devblogs.microsoft.com/dotnet/pin-clustering-in-dotnet-maui-maps/">Pin Clustering in .NET MAUI Maps</a> appeared first on <a href="https://devblogs.microsoft.com/dotnet">.NET Blog</a>.</p>
