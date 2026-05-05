---
title: "Welcome to SkiaSharp 4.0 Preview 1"
url: "https://devblogs.microsoft.com/dotnet/welcome-to-skia-sharp-40-preview1/"
date: "Tue, 28 Apr 2026 17:06:00 +0000"
author: "David Ortinau"
feed_url: "https://devblogs.microsoft.com/dotnet/feed/"
---
<p>Created 10 years ago, <a href="https://github.com/mono/SkiaSharp">SkiaSharp</a> has long been the backbone of cross-platform 2D graphics in .NET. It powers graphics rendering across mobile, desktop, web, and server targets with consistent, high-quality output using the <a href="https://github.com/google/skia">open-source Skia engine</a>. SkiaSharp gives .NET developers the tools to draw text, geometries, and images with pixel-perfect fidelity wherever their apps run. SkiaSharp&#8217;s cross-platform bindings are important to Microsoft for various .NET platforms like .NET MAUI, as well as rendering consistent UI on key technologies like WebAssembly and WinUI 3. Its continued evolution and long-term maintenance are strategically important to the broader .NET ecosystem.</p>
<p><strong>Today, we&#8217;re announcing a major milestone for SkiaSharp. <a href="https://aka.ms/skiasharp-40-package">SkiaSharp 4.0 Preview 1</a> is here.</strong> Our goal is to align SkiaSharp with updates and features that come with a new Skia engine and more modern API. Over two years worth of Skia updates are coming to SkiaSharp with this release. Thank you to all contributors that made this release happen.</p>
<p><div class="d-flex justify-content-center"><a class="cta_button_link btn-primary mb-24" href="https://aka.ms/skiasharp-40-package" target="_blank">Get SkiaSharp 4.0 Preview 1</a></div></p>
<p>Please try out the preview and give us feedback by filing an issue on our repository <a href="https://github.com/mono/SkiaSharp">github.com/mono/SkiaSharp</a>.</p>
<p>We&#8217;re also excited to see <a href="https://platform.uno">Uno Platform</a> step up into a co-maintainer role with us on SkiaSharp. Uno Platform <a href="https://platform.uno/blog/Announcing-UnoPlatform-Microsoft-dotnet-collaboration/">has been collaborating with us</a> on .NET, bringing in the latest Android bindings, contributing to AOT, Wasm, and more. They will co-maintain SkiaSharp alongside the .NET team going forward, helping keep cross-platform graphics rendering accessible to all .NET developers in a timely way.</p>
<p>We&#8217;ll celebrate this collaboration and SkiaSharp 4.0 release on June 30 together at <a href="https://platform.uno/skiasharp">Uno Platform&#8217;s Focus on SkiaSharp online event</a>. Save the date!</p>
<h2>What&#8217;s New in SkiaSharp 4.0 Preview 1</h2>
<p>SkiaSharp 4.0 is a major upgrade. Under the hood, the Skia graphics engine has been updated with years of upstream improvements including rendering quality, codec, and performance gains that benefit every SkiaSharp app automatically, without any code changes required.</p>
<p>Here&#8217;s what&#8217;s available in the first preview:</p>
<p><!-- SECTION: A Better Engine Sources: Skia bumps PRs #3560 (m132), #3660 (m133), #3702 (m147) Engine bullets: m131 mipmap, m123 Exif, m125 tiling, m142 color accuracy, m124 noise perf Security: PRs #3397 (CFG), #3404 (/GS), #3496 #3502 (Spectre), #3718 #3717 #3478 #3469 (dep bumps) --></p>
<h3>A Better Engine</h3>
<p>SkiaSharp 4.0 ships with Skia milestone 147 &#8211; two and a half years and 28 milestones of upstream rendering, quality, and security improvements that benefit your apps automatically:</p>
<p><!-- Source: Skia m131 release notes - GrContextOptions::fSharpenMipmappedTextures restored and enabled by default --></p>
<ul>
<li><strong>Sharper downscaled images</strong> &#8211; mipmap sharpening is now on by default.
<!-- Source: Skia m123 release notes - SkCodec::getImage() will now respect the origin in the metadata --></li>
<li><strong>Automatic photo orientation</strong> &#8211; image codecs now respect Exif rotation metadata.
<!-- Source: Skia m125 release notes - SkTiledImageUtils namespace with DrawImage and DrawImageRect --></li>
<li><strong>Improved large image handling</strong> &#8211; oversized bitmaps automatically tile to fit GPU texture limits.
<!-- Source: Skia m142 release notes - kRec709 corrected to BT.1886. Skia m141 - kHLG/kPQ use new skcms representations. Automatic, no API change needed. --></li>
<li><strong>More accurate colors</strong> &#8211; transfer functions for Rec.709, HLG, and PQ corrected to match industry standards.
<!-- Source: Skia m124 release notes - Perlin noise significantly improved on raster --></li>
<li><strong>Incremental performance gains</strong> &#8211; rendering operations across the board are modestly faster, with specific improvements to noise shaders and canvas operations.
<!-- Source: PRs #3397, #3404, #3496, #3502, #3718, #3717, #3478, #3469 --></li>
<li><strong>Security hardened</strong> &#8211; modern compiler mitigations enabled across all platforms, and all bundled native dependencies updated with security fixes.</li>
</ul>
<p><!-- SECTION: New Capabilities Sources: PRs #3703 (variable fonts), #3742 (color palettes), #3702 (SKPathBuilder), #3730 (typeface resolution), #3217 (Linux Bionic), #3620 (Tizen) --></p>
<h3>New Capabilities</h3>
<p>Here&#8217;s what&#8217;s new in Preview 1, with more on the way in upcoming previews:</p>
<p><!-- Source: PR #3703 - Add variable font support to SkiaSharp and HarfBuzzSharp --></p>
<ul>
<li><strong>Variable fonts</strong> &#8211; full OpenType variable font axis control across SkiaSharp and HarfBuzz. Query axes, set positions, and create typeface variants for weight, width, slant, or custom axes.
<!-- Source: PR #3742 - Add color font palette support and improve variable font robustness --></li>
<li><strong>Color font palettes</strong> &#8211; switch between OpenType CPAL palettes for emoji and icon fonts, or override individual glyph colors.
<!-- Source: PR #3702 - Bump skia to milestone 147; SKPathBuilder added --></li>
<li><strong>SKPathBuilder</strong> &#8211; the modern way to construct paths. <code>SKPath</code> is now immutable under the hood, with <code>SKPathBuilder</code> providing the familiar <code>MoveTo</code>/<code>LineTo</code>/<code>CubicTo</code> API plus shape factories. Existing <code>SKPath</code> methods remain available for backward compatibility.
<!-- Source: PRs #3217 (Linux Bionic), #3620 (Tizen x64/arm64) --></li>
<li><strong>Linux Bionic and Tizen 64-bit</strong> &#8211; new native build targets for Android-based Linux systems and Samsung Tizen x64/arm64 devices.</li>
</ul>
<h3>Read the Release Notes, Try out the Preview</h3>
<p>Read the full release notes for all the details: <a href="https://aka.ms/skiasharp-40-notes">SkiaSharp 4.0 Preview 1 Release Notes</a>.</p>
<p><strong><a href="https://aka.ms/skiasharp-40-package">Try SkiaSharp 4.0 Preview 1</a> and tell us what you think.</strong> Whether you&#8217;re building data visualization tools, custom UI, game surfaces, or anything in between, please try the preview releases, contribute, and give us feedback at <a href="https://github.com/mono/SkiaSharp">github.com/mono/SkiaSharp</a> to help us on our journey to a solid SkiaSharp 4.0 release.</p>
<h2>SkiaSharp 4.0 Roadmap &amp; Interactive Gallery</h2>
<p>We plan on releasing SkiaSharp 4.0 previews often as we move toward release candidate to final release this summer. This will give commercial vendors, partner teams, and the community time to evaluate and give us feedback.</p>
<p>You can see our roadmap and the work we have ahead here: <a href="https://github.com/mono/SkiaSharp/issues/3684">SkiaSharp v4 Release Tracking</a>.</p>
<p>A new website and interactive gallery is also in the works to help you get started and showcase what you can do with SkiaSharp. We&#8217;re building it as a Blazor WebAssembly app with live, interactive demos, from shader playgrounds and path effects to variable fonts and image filters. Check out the early preview at <a href="https://mono.github.io/SkiaSharp/">mono.github.io/SkiaSharp</a>.</p>
<p><a href="https://mono.github.io/SkiaSharp/"><img alt="Visual of the interactive gallery page" src="https://devblogs.microsoft.com/dotnet/wp-content/uploads/sites/10/2026/04/gallery-home.webp" /></a></p>
<p>There&#8217;s also a live SkiaSharp playground, <a href="https://mono.github.io/SkiaSharp/fiddle">SkiaFiddle powered by Uno Platform</a>, that you can use to test out your SkiaSharp code interactively.</p>
<p><a href="https://mono.github.io/SkiaSharp/fiddle"><img alt="Interactive SkiaSharp playground" src="https://devblogs.microsoft.com/dotnet/wp-content/uploads/sites/10/2026/04/gallery-uno.webp" /></a></p>
<h2>A New Partner: Uno Platform Joins as Co-Maintainer</h2>
<p>SkiaSharp is important to Uno Platform. They have built their cross-platform rendering pipeline on SkiaSharp, making them one of the most active and invested stakeholders in the project&#8217;s health. <strong>Uno Platform will co-maintain SkiaSharp</strong> in partnership with the .NET team at Microsoft. This will move SkiaSharp forward at a more rapid pace, with a healthy SkiaSharp-loving community invited to engage and contribute. Read all about <a href="https://aka.platform.uno/skiaannouncement">Uno Platform&#8217;s commitment to SkiaSharp on their blog</a>.</p>
<p>Uno Platform engineers have already made significant contributions to SkiaSharp 4.0:</p>
<ul>
<li><strong>Skia engine bumps</strong> (<a href="https://github.com/mono/SkiaSharp/pull/3560">#3560</a>, <a href="https://github.com/mono/SkiaSharp/pull/3702">#3702</a>) &#8211; Major engine upgrades bringing SkiaSharp up to the latest Skia.</li>
<li><strong>Variable font support</strong> (<a href="https://github.com/mono/SkiaSharp/pull/3703">#3703</a>) &#8211; Full variable font API for SkiaSharp and HarfBuzzSharp.</li>
<li><strong>Android typeface crash fix</strong> (<a href="https://github.com/mono/SkiaSharp/pull/3730">#3730</a>) &#8211; The critical Android API 36 startup fix.</li>
<li><strong>Cross-platform generator tooling</strong> (<a href="https://github.com/mono/SkiaSharp/pull/3714">#3714</a>) &#8211; Made the binding generator work on Linux, enabling broader contributor access.</li>
<li><strong>Uno Platform WebAssembly gallery</strong> (<a href="https://github.com/mono/SkiaSharp/pull/3758">#3758</a>) &#8211; An interactive gallery showcasing SkiaSharp via Uno Platform&#8217;s WebAssembly renderer.</li>
</ul>
<p>Formalizing that relationship as co-maintainers means:</p>
<ul>
<li><strong>Better stability</strong> &#8211; more eyes on the codebase, faster triage, and higher confidence in releases</li>
<li><strong>Improved update cadence</strong> &#8211; contributions from teams actively shipping SkiaSharp-dependent products</li>
<li><strong>Lower barrier to contribute</strong> &#8211; streamlined processes make it easier for the broader .NET community to participate</li>
</ul>
<p>For .NET developers who depend on SkiaSharp in production, this is good news: the library now has multi-organizational momentum behind it that matches its importance to the ecosystem.</p>
<h2>Join the Celebration: SkiaSharp Live Event</h2>
<p>To mark this release, Uno Platform is hosting a live event with .NET team members dedicated entirely to all things SkiaSharp.</p>
<p><strong><img alt="📅" class="wp-smiley" src="https://s.w.org/images/core/emoji/17.0.2/72x72/1f4c5.png" style="height: 1em;" /> Date:</strong> June 30th, 11am – 3pm ET</p>
<p><strong><img alt="📺" class="wp-smiley" src="https://s.w.org/images/core/emoji/17.0.2/72x72/1f4fa.png" style="height: 1em;" /> Where to Watch:</strong></p>
<ul>
<li>Uno Platform channels on YouTube, X (Twitter), and LinkedIn</li>
<li>.NET Foundation YouTube</li>
</ul>
<p><strong><img alt="🔗" class="wp-smiley" src="https://s.w.org/images/core/emoji/17.0.2/72x72/1f517.png" style="height: 1em;" /> Event details:</strong> <a href="https://platform.uno/skiasharp">platform.uno/skiasharp</a></p>
<p>Come for the demos, stay for the deep dives! The event will cover what&#8217;s new in 4.0, what the new co-maintenance model means in practice, and what&#8217;s ahead for SkiaSharp in the .NET ecosystem.</p>
<p>We encourage you to <a href="https://aka.ms/skiasharp-40-package">try out the preview</a> and give us feedback by filing an issue on our repository <a href="https://github.com/mono/SkiaSharp">github.com/mono/SkiaSharp</a>.</p>
<p>Have fun and happy SkiaSharp-ing!</p>
<p>The post <a href="https://devblogs.microsoft.com/dotnet/welcome-to-skia-sharp-40-preview1/">Welcome to SkiaSharp 4.0 Preview 1</a> appeared first on <a href="https://devblogs.microsoft.com/dotnet">.NET Blog</a>.</p>
