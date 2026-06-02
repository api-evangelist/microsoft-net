---
title: "Improving C# Memory Safety"
url: "https://devblogs.microsoft.com/dotnet/improving-csharp-memory-safety/"
date: "2026-05-21"
author: "Richard Lander"
feed_url: "https://devblogs.microsoft.com/dotnet/feed/"
---
The `unsafe` keyword is being redesigned to mark caller-facing contracts rather than just syntax. Safety obligations between callers and callees become visible and reviewable. The model is motivated by the rise of AI-assisted code generation and arrives as a preview in .NET 11.
